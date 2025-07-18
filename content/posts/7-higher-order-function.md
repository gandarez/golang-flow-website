+++
date = '2025-07-18'
title = 'Higher Order Function'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'higher-order-function'
tags = ['golang', 'design', 'pattern', 'function']
summary = 'Higher Order Function é uma função que pode receber outras funções como argumentos ou retorná-las como resultado.'
+++

A função de ordem superior (Higher Order Function) é um conceito fundamental na programação funcional, onde uma função pode receber outras funções como argumentos ou retornar funções como resultado. Esse padrão é amplamente utilizado em linguagens como JavaScript, Haskell e Lisp, permitindo a criação de código mais modular e reutilizável. Embora Go não seja uma linguagem puramente funcional, ela suporta funções de ordem superior.

Neste exemplo, a função `setupTestServer()` cria um servidor de teste HTTP e retorna uma URL, um roteador e uma função. Ela é considerada uma _Higher Order Function_ porque retorna uma função que pode ser usada para fechar o servidor após os testes.

```go
func setupTestServer() (string, *http.ServeMux, func()) {
    router := http.NewServeMux()
    srv := httptest.NewServer(router)

    return srv.URL, router, func() { srv.Close() }
}

func TestSomething(t *testing.T) {
    url, router, close := setupTestServer()
    defer close()

    // ...
}
```

É muito comum em JS utilziar funções de ordem superior como callbacks, como no exemplo abaixo:

```javascript
function fetchData(url, callback) {
    fetch(url)
        .then(response => response.json())
        .then(data => callback(data))
        .catch(error => console.error('Error:', error));
}

function processData(data) {
    console.log('Processed Data:', data);
}

fetchData('https://api.example.com/data', processData);
```

Em Go também podemos criar funções que recebem outras funções como argumentos. Na função `DoSomething()` abaixo, passamos uma função como argumento para processar um mapa de parâmetros. Essa abordagem permite que a função seja flexível e reutilizável, aceitando diferentes implementações dessa função de carregamento de parâmetros.

```go
func LoadParams(params map[string]string) {
    for key, value := range params {
        fmt.Println("Key:", key, "Value:", value)
    }
}

func DoSomething(ctx context.Context, loader func(map[string]string)) {
    params := map[string]string{
        "param1": "value1",
        "param2": "value2",
    }

    loader(params)
}

func main() {
    DoSomething(context.Background(), LoadParams)
}
```

Imagina criar algo parecido com isso?

```go
func Y(g func(any) any) func(any) any {
    return func(f any) func(any) any {
        return f.(func(any) any)(f).(func(any) any)
    }(func(f any) any {
        return g(func(x any) any{
        return f.(func(any) any)(f).(func(any) any)(x)
        })
    })
}
```
