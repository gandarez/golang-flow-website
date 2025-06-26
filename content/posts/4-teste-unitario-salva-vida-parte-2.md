+++
date = '2025-06-18'
title = 'Testes unitários para salvar sua vida - Parte 2'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'teste-unitario-salva-vida-parte-2'
tags = ['golang', 'teste', 'test']
summary = 'Pular os testes unitários é igual pular o dia de pernas na academia - Parte 2'
+++

## Sumário

1. [Subtests com t.Run()](#subtests-com-trun)
2. [Table Driven Test](#table-driven-test)

Se você está lendo isso, provavelmente já leu a primeira parte deste post. Se não, recomendo que leia a [parte 1](/posts/teste-unitario-salva-vida-parte-1) antes de continuar.

## Subtests com t.Run()

No pacote de testes do Go, você pode usar a função `t.Run()` para criar subtests dentro de um teste. Cada chamada a esta função irá disparar uma goroutine blocante, isso significa que a execução do próximo subtest ocorrerá somente após o término desta execução. Para evitar este comportamento basta incluir a chamada a `t.Parallel()` na primeira linha dentro do subtest. 

Este tipo de abordagem pode ser comumente utilizado para agrupar testes relacionados ou para testar diferentes cenários dentro do mesmo caso de uso.

```go
package api_test

import "testing"

func TestClient_Games(t *testing.T) {
    t.Run("success", func(t *testing.T) {
        // caso de sucesso
    })

    t.Run("not found", func(t *testing.T) {
        // caso de erro quando não encontra o recurso
    })

    t.Run("bad request", func(t *testing.T) {
        // caso de erro quando a requisição é inválida
    })
}
```

No output do comando `go test`, você verá os subtests listados, o que facilita a identificação de quais testes passaram ou falharam.

```bash
$ go test -v
=== RUN   TestClient_Games
=== RUN   TestClient_Games/success
=== RUN   TestClient_Games/not_found
=== RUN   TestClient_Games/bad_request
--- PASS: TestClient_Games (0.00s)
    --- PASS: TestClient_Games/success (0.00s)
    --- PASS: TestClient_Games/not_found (0.00s)
    --- PASS: TestClient_Games/bad_request (0.00s)
PASS
ok      github.com/yourusername/yourproject/api  0.001s
```

## Table Driven Test

O Table Driven Test é uma abordagem que vem se tornando muito comum em Go para escrever testes de forma mais organizada e legível, porém muitas vezes o teste fica tão poluído que fica difícil entender o que está sendo efetivamente testado. Nesses casos, é melhor recorrer aos subtests com `t.Run()` ou criar testes separados para cada caso conforme vimos na primeira parte dessa série.

Vamos considerar esta função que queremos testar, que tem como objetivo fazer o parse de uma string e retornar uma lista de expressões regulares ou um valor booleano:

```go
var (
  matchAllRegex = regexp.MustCompile(".*")
  matchNoneRegex = regexp.MustCompile("a^")
)

func parseBoolOrRegexList(s string) ([]regex.Regex, error) {
    var patterns []regex.Regex

    s = strings.ReplaceAll(s, "\r", "\n")
    s = strings.Trim(s, "\n\t ")

    switch {
      case s == "":
      case strings.ToLower(s) == "false":
        patterns = []regex.Regex{regex.NewRegexpWrap(matchNoneRegex)}
      case strings.ToLower(s) == "true":
        patterns = []regex.Regex{regex.NewRegexpWrap(matchAllRegex)}
      default:
        // lógica para lidar com lista de regexes...
    }

  return patterns, nil
}
```

Para testar esta função, podemos usar o Table Driven Test da seguinte forma:

```go
func TestParseBoolOrRegexList(t *testing.T) {
  tests := map[string]struct {
    Input    string
    Expected []regex.Regex
  }{
    "string empty": {
      Input:    " ",
      Expected: nil,
    },
    "false string": {
      Input:    "false",
      Expected: []regex.Regex{regex.NewRegexpWrap(regexp.MustCompile("a^"))},
    },
    "true string": {
      Input:    "true",
      Expected: []regex.Regex{regex.NewRegexpWrap(regexp.MustCompile(".*"))},
    },
    "valid regex": {
      Input: "\t.?\n\t\n \n\t\tgolang.? \t\n",
      Expected: []regex.Regex{
        regex.NewRegexpWrap(regexp.MustCompile("(?i).?")),
        regex.NewRegexpWrap(regexp.MustCompile("(?i)golang.?")),
      },
    },
    "valid regex with windows style": {
      Input: "\t.?\r\n\t\t\tgolang.? \t\r\n",
      Expected: []regex.Regex{
        regex.NewRegexpWrap(regexp.MustCompile("(?i).?")),
        regex.NewRegexpWrap(regexp.MustCompile("(?i)golang.?")),
      },
    },
    "valid regex with old mac style": {
      Input: "\t.?\r\t\t\tgolang.? \t\r",
      Expected: []regex.Regex{
        regex.NewRegexpWrap(regexp.MustCompile("(?i).?")),
        regex.NewRegexpWrap(regexp.MustCompile("(?i)golang.?")),
      },
    },
  }

  for name, test := range tests {
    t.Run(name, func(t *testing.T) {
      // t.Parallel() // Descomente esta linha se quiser que os testes rodem em paralelo

      regex, err := parseBoolOrRegexList(test.Input)
      require.NoError(t, err)

      assert.Equal(t, test.Expected, regex)
    })
  }
}
```

Na terceira e última parte deste post, vamos abordar mocks de uma maneira simples e objetiva. Parte 3 pode ser lida [aqui](/posts/teste-unitario-salva-vida-parte-3).
