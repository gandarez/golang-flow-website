+++
date = '2025-06-13'
title = 'Testes unitários para salvar sua vida - Parte 1'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'teste-unitario-salva-vida-parte-1'
tags = ['golang', 'teste', 'test']
summary = 'Pular os testes unitários é igual pular o dia de pernas na academia - Parte 1'
+++

## Sumário

1. [Introdução](#testes-unitários-para-salvar-sua-vida)
2. [O que são testes unitários?](#o-que-são-testes-unitários)
3. [Boas práticas](#boas-práticas)

## Testes unitários para salvar sua vida

Pular os testes unitários é igual pular o dia de pernas na academia. Você pode até achar que está indo bem, mas no fundo sabe que está fazendo algo errado.
A verdade é que os testes unitários são uma parte essencial do desenvolvimento de software. Eles garantem que o código funcione como esperado, ajudam a identificar bugs e facilitam a manutenção do código a longo prazo.
Neste post, vamos explorar a importância dos testes unitários (e de integração) e como eles podem salvar sua vida (ou pelo menos seu código).

## O que são testes unitários?

O teste unitário garante a confiabilidade específica de um componente do seu software. Ele também assegura que a manutenção do seu método seja mais fácil e reduz o risco de introduzir bugs.

## Boas práticas

1. **Defina um padrão de nome para os testes**: Use nomes descritivos para os testes, como `TestClient_Games`, para que seja fácil entender o que está sendo testado. Evite criar funções com nomes muito extensos ou complexos, comumente escritos em linguagens como [Node.js](#nodejs) e [Ruby](#ruby). Vide exemplo mais abaixo.
2. **Defina a procentagem mínima de cobertura de código**: Estabeleça uma porcentagem mínima de cobertura de código para garantir que os testes cubram uma parte significativa do seu código. Isso ajuda a identificar áreas que precisam de mais testes. Inclusive você pode fazer a sua pipeline de CI falhar caso a cobertura de código esteja abaixo do mínimo estabelecido. Por exemplo, 60% de cobertura de código é um bom ponto de partida.
3. **Testes de funções exportadas**: Os testes devem ser escritos no arquivo `filename_test.go` e devem estar dentro do pacote `{package}_test`. Isso garante que as funções exportadas sejam testadas corretamente e chamamos de black box testing.
4. **Testes de funções não exportadas**: Os testes devem ser escritos no arquivo `filename_internal_test.go` e devem estar dentro do mesmo pacote `{package}`. Isso garante que as funções não exportadas sejam testadas de forma separada.
5. **Crie cenários de sucesso e falha**: Escreva testes para cenários de sucesso e falha, mas explore mais os cenário de falha.
6. **Return struct, accept interface**: Ao escrever funções, retorne uma estrutura de dados e aceite interfaces como parâmetros. Isso facilita a criação de mocks e testes mais flexíveis. Evite criar interfaces muito genéricas e complexas, pois isso pode dificultar a compreensão e manutenção do código, além de tornar a criação de mocks mais difícil.

<a name="nodejs"></a>**Node.js:**

```javascript
describe('Client', () => {
  describe('games', () => {
    it('should return a list of games', () => {
      // Test code here
    });
  });
});
```

<a name="ruby"></a>**Ruby:**

```ruby
describe 'Client' do
  describe '#games' do
    it 'returns a list of games' do
      # Test code here
    end
  end
end
```

**Go:**

```go
package api

import "net/http"

type (
    Client struct {
        baseURL string
        client  *http.Client
    }

    Game struct {
        ID   int    `json:"id"`
        Name string `json:"name"`
    }
)

func (c *Client) Games(criteria string) ([]Game, error) {
    //...
}
```

```go
package api_test

import "testing"

func TestClient_Games(t *testing.T) {
    // caso de sucesso
}

func TestClient_Games_NotFound(t *testing.T) {
    // caso de erro quando não encontra o recurso
}

func TestClient_Games_BadRequest(t *testing.T) {
    // caso de erro quando a requisição é inválida
}
```

Deixo aqui uma sugestão de repositório com muitos testes unitários bem escritos: [WakaTime CLI](https://github.com/wakatime/wakatime-cli).
[![codecov](https://codecov.io/gh/wakatime/wakatime-cli/branch/release/graph/badge.svg?token=9X1Q2Y3Z4W)](https://codecov.io/gh/wakatime/wakatime-cli)

Na segunda parte deste post, vamos abordar subtests e table driven tests, que são técnicas avançadas para escrever testes mais eficientes e organizados. Parte 2 pode ser lida [aqui](/posts/teste-unitario-salva-vida-parte-2).
