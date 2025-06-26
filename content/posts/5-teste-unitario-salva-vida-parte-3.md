+++
date = '2025-06-24'
title = 'Testes unitários para salvar sua vida - Parte 3'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'teste-unitario-salva-vida-parte-3'
tags = ['golang', 'teste', 'test']
summary = 'Pular os testes unitários é igual pular o dia de pernas na academia - Parte 3'
+++

## Sumário

1. [Como mockar?](#como-mockar)
2. [Geradores de mock](#geradores-de-mock)
3. [Manual](#manual)
4. [Dicas finais](#dicas-finais)

Se você está lendo isso, provavelmente já leu a primeira e a segunda parte deste post. Se não, recomendo que leia a [parte 2](/posts/teste-unitario-salva-vida-parte-2) antes de continuar.

## Como mockar?

"__Accept interface, return struct__" é quase que um mantra em Go e pensar dessa forma vai te ajudar na criação de mocks e testes mais flexíveis. Mas afinal o que isso significa?

Vamos imaginar que a função que você precisa testar aceita parâmetros de entrada, como por exemplo uma use case que aceite uma struct de um repositório. Ao tentar criar um teste para essa função você não consegue alterar o comportamento daquele repositório, já que você vai precisar simular operações em banco de dados. Isso se dá ao fato que tipos concretos não podem ser alterados, e para deixar essa função testável, basta você criar uma interface que implemente as funções dessa struct do lado do chamador. Isso significa que você só precisa criar essa interface de acordo com os métodos que você utiliza nessa função, e não precisa se preocupar com todos os métodos da struct original e passar ela como parâmetro para a função que você quer testar. Logo em Go percebemos que não é possível criar mocks diretamente de structs, mas você pode criar mocks de interfaces.

## Geradores de mock

Existem algumas bibliotecas que te ajudam a gerar mocks de uma forma mais automatizada, como por exemplo:

- [Mockery](https://github.com/vektra/mockery)
- [Mock (Uber)](https://github.com/uber-go/mock)
- [Moq](https://github.com/matryer/moq)
- [pgxmock](https://github.com/pashagolub/pgxmock) para mocks de banco de dados PostgreSQL

```go
// Exemplo utilizando Mockery

//go:generate mockery --name TwitchClient --structname MockTwitchClient --inpackage --case snake
type TwitchClient interface {
  Authenticate(ctx context.Context) (string, error)
  ClientID() string
}
```

```bash
# Para gerar o mock, execute o comando:
$ go generate ./...
```

O comando acima irá gerar um arquivo chamado `mock_twitch_client.go` no mesmo pacote, contendo a implementação do mock da interface `TwitchClient`. Você pode então usar esse mock nos seus testes.

## Manual

Para criar um mock manualmente, você deve criar uma struct que implemente a interface que você quer mockar e implementar os métodos dessa interface. Eu gosto de criar uma struct que aceite uma função e tenha um contador por função para posteriormente fazer asserts em cima do número de vezes que ela foi chamada. Vale salientar que essa struct deve ficar dentro do seu arquivo de testes e não de forma exportada em algum pacote que não seja de testes, veja este exemplo:

```go
type (
  // TwitchClient é a interface que define os métodos que o cliente Twitch deve implementar.
  TwitchClient interface {
    Authenticate(ctx context.Context) (string, error)
    ClientID() string
  }

 // mockTwitchClient é uma implementação de TwitchClient para testes.
  mockTwitchClient struct {
    AuthenticateFn      func(ctx context.Context) (string, error)
    AuthenticateFnCount int
    ClientIDFn          func() string
    ClientIDFnCount     int
  }
)

// Authenticate implementa o método Authenticate da interface TwitchClient.
func (m *mockTwitchClient) Authenticate(ctx context.Context) (string, error) {
  m.AuthenticateFnCount++
  return m.AuthenticateFn(ctx)
}

// ClientID implementa o método ClientID da interface TwitchClient.
func (m *mockTwitchClient) ClientID() string {
  m.ClientIDFnCount++
  return m.ClientIDFn()
}

// NewMockTwitchClient cria um novo mock de TwitchClient
// com funções padrão que podem ser sobrescritas para testes específicos.
func NewMockTwitchClient() *mockTwitchClient {
  return &mockTwitchClient{
    AuthenticateFn: func(_ context.Context) (string, error) {
      return "access-token", nil // Retorne o que for necessário para o teste
    },
    ClientIDFn: func() string {
      return "client-id"
    },
  }
}

// TestClient_Games é um exemplo de teste que utiliza o mockTwitchClient.
func TestClient_Games(t *testing.T) {
  mockTwitchClient := &mockTwitchClient{
    AuthenticateFn: func(_ context.Context) (string, error) {
      return "access-token", nil
    },
    ClientIDFn: func() string {
      return "client-id"
    },
  },

  c := igdb.NewClient(igdb.Config{
    TwitchClient: mockTwitchClient,
  })

  games, err := c.Games(context.Background(), "Mario")
  require.NoError(t, err)

  assert.Len(t, games, 10)
  assert.Equal(t, 1, mockTwitchClient.AuthenticateFnCount)
  assert.Equal(t, 1, mockTwitchClient.ClientIDFnCount)
}
```

## Dicas finais

Como última dica, vale ressaltar que as interfaces devem ser pequenas e específicas, evitando criar interfaces muito genéricas que podem dificultar a compreensão do código e a criação de mocks. Além disso, evite criar mocks de interfaces que não são utilizadas diretamente no seu código, pois isso pode levar a testes desnecessários e complexos.

Por exemplo, se você tem uma interface que define métodos para manipulação de arquivos, mas você só precisa de um método específico para o seu teste, crie uma interface menor que contenha apenas esse método.

Rob Pike já dizia: ["The bigger the interface, the weaker the abstraction."](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=317s). 

__"Don’t design your interfaces in advance, discover them as you go."__ Não crie interfaces antes dos tipos concretos, descubra-as conforme você vai escrevendo o código. Isso ajuda a manter as interfaces pequenas e específicas, facilitando a criação de mocks e testes mais flexíveis.
