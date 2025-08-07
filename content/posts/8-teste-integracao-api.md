+++
date = '2025-07-29'
title = 'Testes de integração para sua API'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'teste-integracao-api'
tags = ['golang', 'teste', 'test', 'docker', 'docker-compose', 'integration']
summary = 'O teste de integração valida desde a chamada a sua API ou CLI, até a persistência de dados no banco de dados ou integrações com terceiros.'
+++

## Sumário

1. [Introdução](#introdução)
2. [Testando sua API](#testando-sua-api)

Se você chegou até esse post e não leu ainda sobre testes unitários, eu te convido a ler a [parte 1](/posts/teste-unitario-salva-vida-parte-1).

## Introdução

O teste de integração garante que módulos ou componentes combinados em grupos sejam testados. Esse processo visa verificar a eficiência e a segurança da comunicação entre sistemas. Ele se torna essencial para garantir que o software funcione sem erros de integração. Em resumo, o teste de integração testa desde a chamada a sua API ou CLI, até a persistência de dados no banco de dados ou integrações com terceiros.

## Testando sua API

![Arquitetura da API de Video Games](/images/posts/video-game-api-arquitetura.jpg)

Vamos considerar uma API que gerencia jogos de video games onde ela precisa hora conectar no banco de dados, hora conectar em uma API de terceiros para buscar informações adicionais. Para testar essa API, podemos usar o Docker Compose para criar um ambiente de teste que inclua o banco de dados e a API de terceiros. No nosso exemplo, o banco de dados será o PostgreSQL e para a API de terceiros, vamos usar o [Mock Server](https://www.mock-server.com/).

Abaixo está um exemplo completo de cada arquivo que precisamos para configurar o ambiente de teste com Docker Compose e os arquivos de Dockerfile para a nossa API e testes.

{{< partial-code id="docker-compose" lang="yaml" >}}

// docker-compose.yaml

services:
  video-game-api:
    container_name: video-game-api
    ports:
      - 17020:17020
      - 17022:17022
    environment:
      DOCKER: "true"
      DATABASE_HOST: postgres-db
      DATABASE_PORT: 5432
      DATABASE_USER: postgres
      DATABASE_PASSWORD: postgres
      DATABASE_NAME: postgres
      DATABASE_SSL_MODE: disable
      REDIS_HOST: redis
      SERVICE_NAME: video-game-api
      VENDOR_IGDB_HOST: http://mock_igdb_service:1081
      VENDOR_TWITCH_HOST: http://mock_twitch_service:1080
      VENDOR_TWITCH_CLIENT_ID: client-id
      VENDOR_TWITCH_CLIENT_SECRET: client-secret
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      postgres-db:
        condition: service_healthy
      mock_twitch_service:
        condition: service_started
      mock_igdb_service:
        condition: service_started
    networks:
      - video-game-api-network

  integration-test:
    container_name: integration-test
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      VIDEO_GAME_API_URL: http://video-game-api:17020
      VIDEO_GAME_GRPC_HOST: video-game-api:17022
    depends_on:
      video-game-api:
        condition: service_started
    networks:
      - video-game-api-network

  postgres-db:
    container_name: postgres-db
    image: postgres:16.2-bullseye
    ports:
      - 5438:5432
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      timeout: 10s
      retries: 20
    networks:
      - video-game-api-network

  redis:
    container_name: redis
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mock_twitch_service:
    container_name: mock_twitch_service
    image: mockserver/mockserver
    ports:
      - "1080:1080"
    environment:
      SERVER_PORT: 1080
      MOCKSERVER_INITIALIZATION_JSON_PATH: /config/twitch_expectations.json
    volumes:
      - $PWD/config/mock/:/config/
    networks:
      - video-game-api-network

  mock_igdb_service:
    container_name: mock_igdb_service
    image: mockserver/mockserver
    ports:
      - "1081:1081"
    environment:
      SERVER_PORT: 1081
      MOCKSERVER_INITIALIZATION_JSON_PATH: /config/igdb_expectations.json
    volumes:
      - $PWD/config/mock/:/config/
    networks:
      - video-game-api-network

networks:
  video-game-api-network:
    driver: bridge
{{< /partial-code >}}

{{< partial-code id="dockerfile" lang="dockerfile" >}}

# Dockerfile

FROM golang:1.24-bullseye AS build

WORKDIR /src

# Copy everything but defined in docker ignore file
COPY . .

# Build
RUN go mod vendor
RUN make build-linux-amd64

#####################
# Build final image #
#####################
FROM alpine AS bin

# Copy from build
COPY --from=build /src/build/video-game-api-linux-amd64 ./video-game-api
COPY --from=build /src/db ./db

# Specify the container's entrypoint as the action
ENTRYPOINT ["./video-game-api"]

{{< /partial-code >}}

{{< partial-code id="dockerfile-test" lang="dockerfile" >}}

# Dockerfile.test

FROM golang:1.24-bullseye

WORKDIR /src

# Copy everything but defined in docker ignore file
COPY . .

# Download dependencies
RUN go mod vendor

CMD [ "go", "test", "-v", "-race", "-timeout=30s", "-tags=integration", "./test/integration_test/..." ]

{{< /partial-code >}}

Agora que temos o ambiente configurado, podemos escrever os testes de integração. Vamos criar um arquivo de teste chamado `console_http_test.go` dentro do diretório `test/integration_test`.

{{< partial-code id="integration-test" lang="go" >}}

func TestConsoleSearch_Http(t *testing.T) {
  apiURL := os.Getenv("VIDEO_GAME_API_URL")
  url := apiURL + "/consoles/b171ae30-2d02-4da2-98b4-33ad2c331669"

  req, err := http.NewRequest(http.MethodGet, url, nil)
  require.NoError(t, err)

  req.Header.Set("Accept", "application/json")
  req.Header.Set("Content-Type", "application/json")

  client := &http.Client{}

  resp, err := client.Do(req)
  require.NoError(t, err)

  defer resp.Body.Close()

  require.Equal(t, http.StatusOK, resp.StatusCode)

  resbody, err := io.ReadAll(resp.Body)
  require.NoError(t, err)

  var console model.Console

  err = json.Unmarshal(resbody, &console)
  require.NoError(t, err)

  assert.Equal(t, "b171ae30-2d02-4da2-98b4-33ad2c331669", console.ID)
  assert.Equal(t, "Xbox 360", console.Name)
  assert.Equal(t, "Microsoft", console.Manufacturer)
  assert.Equal(t, "2005-11-22", console.ReleaseDate)
}

{{< /partial-code >}}

1. Obtenha a URL da API a partir da variável de ambiente `VIDEO_GAME_API_URL`. Nesse caso ela é definida no `docker-compose.yaml` e aponta para o serviço `video-game-api`.

```Go
apiURL := os.Getenv("VIDEO_GAME_API_URL")
```

2. Crie uma nova requisição HTTP para a rota `/consoles/{id}`.

```Go
url := apiURL + "/consoles/b171ae30-2d02-4da2-98b4-33ad2c331669"
req, err := http.NewRequest(http.MethodGet, url, nil)
```

3. Defina os cabeçalhos `Accept` e `Content-Type` para `application/json`.

```Go
req.Header.Set("Accept", "application/json")
req.Header.Set("Content-Type", "application/json")
```

4. Use o cliente HTTP para enviar a requisição e obter a resposta.

```Go
client := &http.Client{}
resp, err := client.Do(req)
```

5. Verifique se o status da resposta é `200 OK`.

```Go
require.Equal(t, http.StatusOK, resp.StatusCode)
```

6. Leia o corpo da resposta e verifique se não houve erro.

```Go
resbody, err := io.ReadAll(resp.Body)
require.NoError(t, err)

var console model.Console

err = json.Unmarshal(resbody, &console)
require.NoError(t, err)
```

7. Verifique se os dados retornados estão corretos.

```Go
assert.Equal(t, "b171ae30-2d02-4da2-98b4-33ad2c331669", console.ID)
assert.Equal(t, "Xbox 360", console.Name)
assert.Equal(t, "Microsoft", console.Manufacturer)
assert.Equal(t, "2005-11-22", console.ReleaseDate)
```

Para executar os testes de integração, você pode usar o seguinte comando:

```bash
docker compose run --build --rm integration-test
```

Dessa forma conseguimos garantir que a nossa API está funcionando corretamente iniciando pela chamada HTTP e validando a resposta.

Deixo aqui uma sugestão de repositório com testes de integração: [Video Game API](https://github.com/gandarez/video-game-api/tree/master/test/integration_test).

Na segunda parte deste post, vamos ver como podemos testar CLIs.
