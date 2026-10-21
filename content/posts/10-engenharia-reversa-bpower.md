+++
date = '2026-10-21'
title = 'Engenharia reversa de um módulo piggyback: do bytecode ao app novo'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'engenharia-reversa-bpower'
tags = ['flutter', 'dart', 'bluetooth', 'ble', 'engenharia-reversa', 'react-native', 'mobile']
summary = 'Como descobri o protocolo Bluetooth de um módulo de potência automotivo cruzando bytecode decompilado com captura de tráfego, e reescrevi o app do zero: mais estável, mais honesto com o motorista e validado dentro do carro.'
+++

## Sumário

1. [Introdução](#introdução)
2. [O que é um módulo piggyback](#o-que-é-um-módulo-piggyback)
3. [O app original](#o-app-original)
4. [Duas fontes de verdade](#duas-fontes-de-verdade)
5. [O protocolo, byte a byte](#o-protocolo-byte-a-byte)
6. [O mapa e o quebra-cabeça do CRC](#o-mapa-e-o-quebra-cabeça-do-crc)
7. [Os desafios que não estavam no código](#os-desafios-que-não-estavam-no-código)
8. [A arquitetura do app novo](#a-arquitetura-do-app-novo)
9. [Validando dentro do carro](#validando-dentro-do-carro)
10. [A nova interface](#a-nova-interface)
11. [Ganhos em estabilidade e performance](#ganhos-em-estabilidade-e-performance)
12. [Conclusão](#conclusão)

## Introdução

__Todo carro com módulo de potência vem com um app. Nem todo app merece o carro.__

Meu carro tem um módulo __Bpower__: uma caixinha que fica entre o sensor de pressão do coletor e a central do motor, controlada pelo celular via Bluetooth.

O app oficial funciona. Mais ou menos.

Ele mostra `n/c` em quase tudo até conectar. Tem botões que não fazem nada. Não reconecta sozinho se você reinicia o Bluetooth. E, como descobri da pior forma, às vezes mostra um modo de potência que não é o que o módulo está usando.

Então fiz o que qualquer engenheiro faria: abri o app, descobri como ele conversa com o módulo e escrevi outro.

Este texto conta como foi essa engenharia reversa, os problemas que só apareceram no tráfego real e o que mudou entre o app original e o novo.

## O que é um módulo piggyback

Um módulo __piggyback__ não reprograma a central do motor (ECU). Ele "pega carona" no chicote.

O módulo intercepta o sinal do sensor de pressão do coletor (MAP) e entrega para a ECU um valor __um pouco menor__ do que o real. A ECU acha que o turbo está entregando menos pressão do que deveria e compensa pedindo mais.

Quanto menor o sinal, mais a ECU compensa. Os níveis de potência do app (Eco, Stock, Sport 1 a 7, Race) escolhem o quanto o módulo mexe nesse sinal.

Isso parece teoria até você medir. No teste no carro (detalhes mais abaixo), com um leitor OBD gravando o que a ECU enxergava ao mesmo tempo em que o módulo mandava a pressão real, a ECU via __7% a 8% menos pressão__ do que existia de fato nas faixas de maior intervenção do mapa.

## O app original

![App original: tela principal desconectada e tela de configurações](/images/posts/bpower-app-original.jpg)

O app original é um app __React Native__. Visualmente ele lembra um painel de carro: mostrador grande de boost, um botão de "Exhaust", um de "Pedal" e uma barra embaixo.

Olhando com calma, os problemas aparecem:

- Antes de conectar, tudo é `n/c`. Não existe nenhuma informação de __em que etapa__ a conexão está
- Os controles de __Pedal__ e __Exhaust__ são só interface. Nenhum comando Bluetooth é enviado quando você mexe neles
- Se você desliga e liga o Bluetooth do celular, o app __não reconecta__. Ele inicia o scan com o rádio desligado e não tenta de novo
- O nível de potência 0 aparece como __Eco__, o que gera confusão de verdade (já chego nisso)

Nada disso impede o módulo de funcionar. Mas tudo isso atrapalha quem está dirigindo.

## Duas fontes de verdade

Engenharia reversa de protocolo fica muito mais fácil quando você tem __duas fontes que se confirmam__.

__Fonte 1: o código.__
Em apps React Native com Hermes, o JavaScript não vai em texto puro dentro do APK. Ele vai compilado em __bytecode Hermes__ no arquivo `index.android.bundle`. Com o `hermes-dec` dá para decompilar esse bytecode em algo legível.

Toda a lógica de comunicação estava concentrada em uma única tela, a `ModuloScreen`: scan, conexão, handshake, polling, escrita do mapa. O código mostra __o que o app pretende fazer__.

__Fonte 2: o tráfego.__
O Android tem uma opção de desenvolvedor chamada __log de espionagem HCI do Bluetooth__. Com ela ligada, o sistema grava cada pacote que passa pela controladora Bluetooth em um arquivo `btsnoop_hci.log`.

Junto com o logcat do app, isso mostra __o que realmente acontece no ar__: cada escrita, cada notificação, com timestamp.

Um script em Python decodifica esse log e extrai só o que interessa: as escritas na característica do módulo e as notificações que voltam.

Cruzar as duas fontes foi o que destravou tudo. O código explica a intenção. A captura mostra a realidade, incluindo o que o código não trata.

## O protocolo, byte a byte

O módulo é um __BLE-UART transparente__, do tipo HM-10: um serviço `FFE0` com uma característica `FFE1`, por onde passam escritas e notificações.

O app pede MTU de 247 bytes. O módulo aceita __23__. Na prática, cada escrita ou notificação carrega no máximo __20 bytes__ de dados.

O QR code colado no módulo traz tudo o que o app precisa para encontrá-lo e parear: o __MAC__, o __PIN__ de pareamento e um multiplicador de calibração do sensor.

Os comandos são letras ASCII:

| Bytes | ASCII | Função |
|---|---|---|
| `52` | `R` | Dados em tempo real (boost e rpm) |
| `53` | `S` | Versão do firmware e serial |
| `45` | `E` | Lê o bloco de configuração (195 bytes) |
| `54` | `T` | CRC-32 do mapa gravado no módulo |
| `79 …` | `y` | Escreve configuração (troca de potência) |
| `50`, `5A`, `42` | `P`, `Z`, `B` | Envio e gravação do mapa |

Agora vem a parte curiosa: __as respostas não têm cabeçalho__.

Não existe um byte dizendo "isto é a resposta do `R`". O app original guarda um __modo global__ com o último comando enviado e interpreta a próxima notificação de acordo com ele.

| Modo | Comando | Resposta esperada |
|---|---|---|
| 0 | `E` | 195 bytes, acumulados em 10 notificações |
| 1 | `R` | 9 bytes, uma notificação |
| 2 | `S` | 23 bytes, em duas notificações |
| 3 | `T` | 4 bytes |
| 4 | `y` | `OK` ou `ER` |

Isso significa que __só pode existir um comando em voo por vez__. E que uma notificação atrasada pode ser lida como resposta do comando errado.

O pacote de tempo real é um bom exemplo. Esta é uma resposta real do `R`, capturada com o motor em marcha lenta:

```text
00 CE 02 8B 00 C6 FB 43 8B
└─┬─┘ └─┬─┘ ├┘ └────┬────┘
 boost  rpm  │    CRC-32 (little-endian) dos 5 bytes anteriores
 = 206 = 651 status
```

Pressão bruta 206, __651 rpm__, status 0. Os quatro bytes finais são um CRC-32 dos cinco primeiros.

Na captura, __505 de 505__ pacotes tinham CRC válido. Ainda assim, o app novo valida todos e descarta o que não bate:

```dart
static RealtimeSample? tryParse(List<int> bytes, {required int fwNumber}) {
  final len = lengthFor(fwNumber);
  if (bytes.length != len) return null;
  final payloadLen = len - 4;
  if (Crc32.compute(bytes, 0, payloadLen) != crc32FromLe(bytes, payloadLen)) {
    return null;
  }
  return RealtimeSample(
    rawBoost: (bytes[0] << 8) | bytes[1],
    rpm: (bytes[2] << 8) | bytes[3],
    status: payloadLen == 5 ? bytes[4] : 0,
  );
}
```

O valor bruto vira pressão com o multiplicador do QR code, e o boost exibido é essa pressão absoluta menos a pressão atmosférica.

## O mapa e o quebra-cabeça do CRC

O "mapa" é um binário que o app baixa do servidor do fabricante usando o serial do módulo. São __991 bytes__ com um layout bem definido:

| Offset | Conteúdo |
|---|---|
| 0–323 | __Tabela 1__: 18×18, RPM × pressão |
| 324–395 | Eixos da tabela 1 (18 pontos de RPM e 18 de pressão) |
| 398–721 | __Tabela 2__: 18×18 (zerada neste mapa) |
| 722–793 | Eixos da tabela 2 |
| 797–991 | __Bloco de configuração__ |

O detalhe que fez tudo se encaixar: os últimos 195 bytes do mapa são __exatamente__ o que o comando `E` devolve. E o nível de potência mora no nibble alto do byte 797.

Isso cria um problema interessante. Quando você troca a potência, o módulo altera o byte 797, e o CRC do mapa gravado muda. Como saber, então, se o mapa do módulo é o mesmo do servidor?

O app original resolve com força bruta: para cada nível de 0 a 10, ele altera o byte 797 na cópia do servidor, calcula o CRC-32 e compara com o que o comando `T` devolveu. Se algum bater, o mapa é o mesmo e você ainda descobre em qual nível ele está.

Se nenhum bater, aparece o famoso __"Novo mapa online!"__.

O envio do mapa também é todo cronometrado: anuncia o CRC (`P`), sinaliza o início (`Z`), manda 50 escritas de 20 bytes (cerca de 3,3 s com MTU 23), confere o CRC com `T` e só então manda gravar na flash (`B`). O módulo não responde ao `B`, então o app simplesmente espera 4 segundos.

## Os desafios que não estavam no código

Algumas coisas só apareceram com o módulo de verdade, dentro do carro.

__O `E` que não responde de primeira.__
Nas capturas, o primeiro pedido de configuração da conexão ficou sem resposta. O app original já tinha um retry para isso ("GetConfig - no response"). O novo reproduz o mesmo comportamento, porque o firmware claramente depende dele.

__Notificações fora de hora.__
Às vezes chegam notificações cheias de `FF` depois que a resposta já foi montada. No modelo sem cabeçalho, isso pode virar dado errado. O app novo descarta qualquer notificação que chegue fora de um comando em andamento.

__Pareamento com PIN.__
No Android, o PIN do QR code precisa ser entregue ao sistema durante o pareamento, sem o usuário digitar nada. Isso exigiu um `BroadcastReceiver` nativo respondendo ao pedido de bond. No iOS não existe essa API: o app mostra o PIN e o usuário digita no alerta do sistema.

__O boost fantasma.__
Com o motor desligado, o app mostrava entre 0,03 e 0,09 bar de boost.

A fórmula original calcula a pressão atmosférica pela altitude do GPS. O problema é que o sensor do módulo lê um pouco acima disso. No teste, o sensor do módulo marcou 95,7 kPa com o motor parado, contra 93,0 kPa do barômetro da ECU: __2,7 kPa de diferença__, a mesma ordem de grandeza do boost fantasma.

A solução foi um __zero automático__: com o motor parado, o app aprende o offset entre o sensor e o GPS (mediana das amostras com rpm 0) e corrige a referência. Como o que se aprende é o offset e não a pressão absoluta, a correção continua valendo quando o carro sobe ou desce a serra.

__O nível que não era Eco.__
Durante os testes, o módulo estava no nível 0, e a tela dizia __Eco__, como no app original. Resultado: puxadas que pareciam ter sido feitas no Eco tinham sido feitas em outro nível. Hoje o nível 0 aparece como "Nível 0". Parece detalhe, mas foi o tipo de detalhe que invalida uma bateria inteira de testes.

__O famoso erro 133.__
O Android às vezes recusa a conexão Bluetooth com o código genérico 133. O app novo tenta de novo sozinho a cada segundo e, em vez de piscar a mensagem por um segundo, mantém a última falha visível no banner até conectar.

## A arquitetura do app novo

O app novo é __Flutter__, para Android e iOS, com uma regra clara: __o protocolo não sabe que existe Bluetooth__.

```text
lib/protocol/   Dart puro: CRC-32, QR code, comandos, respostas,
                layout do mapa, pressão atmosférica e boost
lib/services/   BleLink        → scan, conexão, pareamento, MTU, notify
                ModuleSession  → um comando em voo, acumulação, retry
                ModuleController → handshake, polling, watchdog, reconexão
lib/pages/      Interface
```

A camada `protocol` tem cerca de 600 linhas e nenhuma dependência de Flutter. Ela é testada com __vetores reais__ tirados das capturas: o mapa de verdade, os pacotes `R` de verdade, os pacotes de troca de potência de verdade.

A `ModuleSession` reproduz o modelo de modos do app original, mas de forma explícita: um comando por vez, notificações acumuladas até o tamanho esperado e qualquer coisa fora de hora descartada. Ela é testada com um transporte falso, sem precisar de módulo.

O `ModuleController` cuida do ciclo de vida: handshake, polling a cada 150 ms, um watchdog que considera a comunicação perdida após 5 s sem notificação, reconexão automática e pausa do polling quando o app vai para o background.

No total, são __76 testes automatizados__.

## Validando dentro do carro

Teste unitário prova que o código segue o protocolo. Não prova que o carro anda diferente.

Para isso, montei um ferramental em Python que, a cada "bateria" (uma marcha e um nível de potência):

- Puxa um bugreport do celular via `adb`, com o log HCI e o log do app
- Recorta só a janela daquela bateria
- Junta com os dados de um leitor OBD (ELM327) gravando rpm, MAP e pressão barométrica da ECU
- Detecta as puxadas, agrupa pelo nível __lido do módulo__ (não pelo rótulo que eu dei) e compara

Um trecho do relatório da 3ª marcha, só com puxadas completas (de 3000 para 5000+ rpm):

| Nível | Puxadas | Pico de boost (p95) | Pressão vista pela ECU ÷ pressão real |
|---|---|---|---|
| Stock | 2 | 1,20 bar | 0,928 |
| Sport 1 | 1 | 1,43 bar | 0,920 |
| Race | 1 | 1,38 bar | 0,916 |

A amostra é pequena e ainda não permite comparar os níveis entre si com confiança. Mas ela confirma o mecanismo: __a ECU vê de 7% a 8% menos pressão do que a real__, e essa redução parece crescer um pouco com o nível.

Foi também esse ferramental que pegou o caso do "Nível 0 mostrado como Eco".

## A nova interface

![Antes e depois: app original à esquerda, app novo à direita](/images/posts/bpower-antes-depois.jpg)

A nova interface nasceu de um protótipo interativo e foi implementada em Flutter com uma ideia simples: __mostrar o que está acontecendo e esconder o que não existe__.

- __Boost e rotação__ lado a lado, com pico de boost e a faixa de corte marcada em vermelho no conta-giros
- __Shift light__: uma fita de LEDs que acende de verde para âmbar e vermelho antes do corte e pisca no corte
- __Gráfico dos últimos 6 segundos__ de boost e rpm
- __Potência__ com Eco, Stock, Sport e Race. Cada troca é uma escrita no módulo, então ela só é enviada quando você solta o controle
- __Status da conexão__ sempre visível: procurando, conectando, conectado, e a última falha, se houver
- __Detalhes do mapa__ com as tabelas decodificadas e a célula em uso destacada em tempo real
- __Diagnóstico__ com o log ao vivo, filtrado por camada (Bluetooth, sessão, módulo, mapa)

Pedal e Exhaust saíram, porque nunca fizeram nada. O shift light e a faixa de corte, que no original ficavam escondidos num modal de configurações, ganharam uma tela própria.

Virando o celular, a tela vira um painel só com os mostradores e atalhos para todos os níveis:

![App novo em modo paisagem](/images/posts/bpower-app-novo-paisagem.jpg)

## Ganhos em estabilidade e performance

Comparando só o que está documentado nas capturas e no código do app original:

| Situação | App original | App novo |
|---|---|---|
| Bluetooth desligado e religado | Não reconecta | Detecta e reconecta sozinho |
| Falha de conexão | `n/c` / "Scanning for device..." | Etapa atual e última falha visíveis |
| Boost com o motor desligado | 0,03 a 0,09 bar fantasma | 0, com zero automático |
| Nível de potência 0 | Exibido como Eco | Exibido como "Nível 0" |
| Pedal e Exhaust | Controles sem efeito | Removidos |

E a performance?

Aqui vale ser honesto: __o gargalo é o módulo, não o app__. Com MTU de 23 bytes e intervalo de conexão de 30 ms, a ida e volta de um `R` leva em torno de 50 ms e o polling roda a cerca de 6 leituras por segundo. Gravar um mapa leva uns 3,3 s só de transferência, mais os 4 s que o firmware exige para gravar na flash.

O app novo segue esses tempos __de propósito__. O firmware foi feito para o ritmo do app original, e acelerar além disso é pedir para perder pacote.

O ganho está em outro lugar:

- O código Dart roda compilado __AOT para nativo__, em vez de bytecode Hermes interpretado
- O boost exibido passa por uma média exponencial de ~0,3 s. O número para de "dançar" com a pulsação do coletor, sem atrasar a leitura
- Os mostradores animam entre amostras, então 6 leituras por segundo parecem um ponteiro contínuo
- Nenhuma leitura chega na tela sem passar pelo CRC, e notificações fora de hora são descartadas em vez de interpretadas
- 76 testes automatizados, alimentados com dados reais das capturas, rodam a cada mudança

Performance aqui não é fazer mais rápido. É __fazer previsível__.

## Conclusão

Este projeto começou com uma irritação simples: um app que mostrava `n/c` e botões que não faziam nada.

Terminou com um protocolo documentado byte a byte, um app que valida cada pacote, um ferramental para medir o efeito real de cada nível de potência no carro e a descoberta de como o módulo realmente funciona.

Algumas lições que valem para qualquer engenharia reversa:

- __Cruze fontes.__ O código diz o que o software pretende fazer. O tráfego diz o que acontece. Os bugs moram na diferença
- __Teste com dados reais.__ Os vetores das capturas viraram testes, e cada regressão aparece antes de chegar ao carro
- __Respeite os tempos do hardware.__ Firmware embarcado raramente perdoa pressa
- __Meça no mundo real.__ O "Nível 0 mostrado como Eco" passaria despercebido em qualquer teste de mesa

Ainda há trabalho: o erro 133 do Android aparece mais do que eu gostaria, e a amostra de puxadas precisa crescer antes de qualquer conclusão sobre qual nível rende mais.

Mas o carro agora tem um app à altura dele.

> __Uma nota sobre engenharia reversa:__ todo o trabalho foi feito no meu próprio módulo, para interoperabilidade com um app próprio. Não houve contorno de nenhuma proteção, e identificadores do módulo e endereços do fabricante foram omitidos deste texto de propósito.
