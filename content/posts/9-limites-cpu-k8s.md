+++
date = '2026-01-28'
title = 'Limites de CPU no Kubernetes: como isso afeta aplicações em Go'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'limites-cpu-k8s'
tags = ['golang', 'kubernetes', 'docker', 'performance', 'containers', 'GOMAXPROCS']
summary = 'Entenda o que realmente acontece quando você define limites de CPU no Kubernetes para aplicações Go, como isso impacta o runtime e como o Go 1.25 melhora esse cenário com GOMAXPROCS container-aware.'
+++

## Sumário

1. [Introdução](#introdução)
2. [CPU no Kubernetes: tempo, não quantidade](#cpu-no-kubernetes-tempo-não-quantidade)
3. [O scheduler não se importa com seu código](#o-scheduler-não-se-importa-com-seu-código)
4. [O custo real das operações](#o-custo-real-das-operações)
5. [Onde o Go entra nessa história](#onde-o-go-entra-nessa-história)
6. [Paralelismo demais custa caro](#paralelismo-demais-custa-caro)
7. [Concorrência não é paralelismo](#concorrência-não-é-paralelismo)
8. [CPU-bound vs I/O-bound](#cpu-bound-vs-io-bound)
9. [O problema real: previsibilidade](#o-problema-real-previsibilidade)
10. [Conclusão](#conclusão)

## Introdução

__Limites de CPU no Kubernetes: uma análise para aplicações em Go__

Configurar limites de CPU no Kubernetes costuma ser tratado como algo simples. Você escolhe um valor em millicores, ajusta `requests` e `limits` e segue em frente.

O problema é que, quando rodamos __aplicações em Go dentro de containers__, essa decisão afeta diretamente o comportamento do runtime, o escalonamento de goroutines e, em muitos casos, a previsibilidade da aplicação.

Este texto não é sobre YAML.
É sobre __o que realmente acontece quando você limita CPU no Kubernetes__ e como isso impacta aplicações escritas em Go.

## CPU no Kubernetes: tempo, não quantidade

CPU no Kubernetes não é “quantidade”

Considere a configuração abaixo:

```yaml
resources:
  requests:
    cpu: "250m"
  limits:
    cpu: "250m"
```

É comum interpretar `250m` como “25% de um core”.
Essa interpretação está errada ou, no mínimo, incompleta.

No Kubernetes, CPU é tratada como __tempo__, não como uma fração fixa de hardware.

Na prática:

- `250m` significa 25% do tempo de CPU
- Em uma janela de 100ms, o container pode executar por cerca de 25ms
- No restante do tempo, ele simplesmente não roda

Do ponto de vista da aplicação, isso não aparece como erro.
A execução apenas fica mais lenta e menos previsível.

## O scheduler não se importa com seu código

O controle de CPU no Kubernetes é feito via o __CFS (Completely Fair Scheduler)__ do Linux. Ele interrompe o processo quando o orçamento de CPU é excedido.

Para a aplicação Go, isso é invisível.
Não existe callback, sinal ou erro.

Um loop CPU-bound simples já evidencia isso:

```go
func work() {
    for {
        _ = 1 + 1
    }
}
```

Esse código nunca bloqueia.
Ele tenta usar CPU o tempo todo.

Em um container sem limite, ele roda continuamente.
Com limite de CPU, ele passa boa parte do tempo __impedido de executar__, mesmo sem fazer I/O ou bloqueios explícitos.

## O custo real das operações

Uma CPU moderna executa bilhões de ciclos por segundo. Ainda assim, o custo das operações varia drasticamente.

Por exemplo, um mutex simples:

```go
var mu sync.Mutex

func critical() {
    mu.Lock()
    defer mu.Unlock()

    // seção crítica
}
```

Esse `Lock/Unlock` já custa centenas de ciclos.
Em um ambiente com CPU limitada, esse custo passa a competir diretamente com todo o resto da aplicação.

Nada muda no código.
O impacto aparece no tempo.

## Onde o Go entra nessa história

O Go tem seu próprio scheduler. Ele trabalha com três entidades principais:

- `G`: goroutine
- `M`: thread do sistema operacional
- `P`: logical processor

O número de `Ps` define quantas goroutines podem executar __em paralelo__. Esse valor é controlado por `GOMAXPROCS`.

Um exemplo simples:

```go
fmt.Println(runtime.GOMAXPROCS(0))
```

Historicamente, esse valor era calculado com base no número de cores visíveis para o processo, ignorando limites de CPU impostos pelo container.

O efeito prático disso é que o runtime assume que há mais capacidade de execução do que realmente existe.

## Paralelismo demais custa caro

Considere um workload CPU-bound simples:

```go
func cpuBound() {
    sum := 0
    for i := 0; i < 100_000_000; i++ {
        sum += i
    }
    _ = sum
}
```

Agora execute isso em paralelo:

```go
var wg sync.WaitGroup

for i := 0; i < 8; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        cpuBound()
    }()
}

wg.Wait()
```

Se o container tiver CPU suficiente, isso pode escalar bem.
Mas em um container limitado, o resultado costuma ser o oposto:

- Mais goroutines competindo pelo mesmo tempo de CPU
- Mais preempção
- Mais overhead de escalonamento
- Menor previsibilidade de latência

__Criar mais paralelismo não cria mais CPU.__

## Concorrência não é paralelismo

Go facilita concorrência.
Isso não significa que você deva executar tudo em paralelo.

Para workloads CPU-bound, limitar paralelismo explicitamente costuma ser a escolha correta:

```go
sem := make(chan struct{}, runtime.GOMAXPROCS(0))

for _, job := range jobs {
    sem <- struct{}{}
    go func(j Job) {
        defer func() { <-sem }()
        process(j)
    }(job)
}
```

Esse padrão mantém concorrência, mas impede que a aplicação tente executar mais trabalho em paralelo do que o ambiente suporta.

## CPU-bound vs I/O-bound

Nem toda aplicação sofre da mesma forma com limites de CPU.

Um handler I/O-bound típico:

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    time.Sleep(50 * time.Millisecond)
    w.WriteHeader(http.StatusOK)
})
```

Aqui, a goroutine passa boa parte do tempo bloqueada.
O impacto do limite de CPU é menor.

Agora compare com um handler CPU-bound:

```go
http.HandleFunc("/compute", func(w http.ResponseWriter, r *http.Request) {
    cpuBound()
    w.WriteHeader(http.StatusOK)
})
```

Nesse caso, o limite de CPU afeta diretamente:

- Latência
- Throughput
- Consistência das respostas

__Saber em qual categoria sua aplicação se encaixa é fundamental.__

## O problema real: previsibilidade

O maior problema dos limites de CPU não é apenas performance. É previsibilidade.

Quando a aplicação:

- Acredita ter mais CPU do que realmente tem
- Cria paralelismo excessivo
- É constantemente interrompida pelo scheduler

Os sintomas aparecem como:

- Picos de latência difíceis de explicar
- Quedas de throughput sem erro aparente
- Diferenças grandes entre ambientes

Tudo isso sem que o código esteja tecnicamente errado.

## Conclusão

Limitar CPU no Kubernetes não é apenas uma decisão operacional. É uma decisão que afeta diretamente:

- O modelo de execução da aplicação
- O comportamento do runtime do Go
- A eficiência do paralelismo
- A previsibilidade do sistema

Durante muito tempo, aplicações Go rodando em containers sofriam porque o runtime __não tinha consciência real dos limites de CPU impostos pelo Kubernetes__. O valor de GOMAXPROCS era calculado com base nos cores visíveis para o processo, não no tempo de CPU efetivamente disponível.

A partir do Go __1.25__, em __sistemas Linux__, isso muda.
O runtime passa a detectar automaticamente os limites de CPU quando executando dentro de containers, ajustando `GOMAXPROCS` de forma coerente com o ambiente do Kubernetes.

Isso reduz paralelismo excessivo, melhora previsibilidade e evita parte dos problemas discutidos ao longo deste texto, especialmente em workloads CPU-bound.

Ainda assim, isso não elimina a necessidade de entendimento. Limites de CPU continuam sendo limites de __tempo__ e o comportamento da aplicação continua dependendo do tipo de workload, do padrão de concorrência e das decisões de design.

Não é sobre confiar cegamente no runtime.
É sobre entender como ele funciona e como o ambiente influencia suas escolhas.

Aqui você pode assistir na íntegra a palestra onde apresentei esse conteúdo:

[![Limites de CPU no k8s: Uma Ánalise para Aplicações em Go](https://img.youtube.com/vi/xIld3MATx74/maxresdefault.jpg)](https://youtu.be/xIld3MATx74?t=1143)
