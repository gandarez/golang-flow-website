+++
date = '2025-07-09'
title = 'Options Pattern'
author = 'Carlos Henrique Guardão Gandarez'
slug = 'options-pattern'
tags = ['golang', 'design', 'pattern', 'option']
summary = 'Options Pattern é um padrão de design que permite criar objetos configuráveis de forma flexível e extensível.'
+++

## Sumário

1. [Valores pré-definidos](#valores-pré-definidos)
2. [Parâmetros de Configuração](#parâmetros-de-configuração)
3. [Options Pattern - Declaração](#options-pattern---declaração)
4. [Options Pattern - Implementação](#options-pattern---implementação)
5. [Options Pattern - Controle de Erros](#options-pattern---controle-de-erros)
6. [Na Prática](#na-prática)
7. [Testes Unitários](#testes-unitários)

O __Options Pattern__ é um padrão de design utilizado para configurar objetos de forma flexível, muito útil quando uma função ou struct precisa de muitos parâmetros opcionais. Dessa forma evitamos o uso de múltiplas funções construtoras ou o excesso de parâmetros em funções, o que torna o código mais limpo e extensível.

_Ao longo deste post vamos utilizar como exemplo um cliente http_.

## Valores pré-definidos

A forma mais simples de se criar uma instância de uma struct é utilizando valores pré-definidos ou constantes. Dessa forma, você perde a flexibilidade de modificar os valores em tempo de inicialização, como por exemplo diferentes configurações por ambiente.

```go
type Client struct {
    baseURL string
    client *http.Client
}

func NewClient(baseURL) *Client {
    return &Client{
        baseURL: baseURL,
        client: &http.Client{
            Timeout: time.Duration(10) * time.Second,
            Transport: &http.Transport{
                ForceAttemptHTTP2: true,
                MaxConnsPerHost: 5,
                MaxIdleConns: 2,
                MaxIdleConnsPerHost: 2,
                TLSHandshakeTimeout: time.Duration(5) * time.Second,
            },
        },
    }
}
```

## Parâmetros de Configuração

Podemos melhorar a flexibilidade do nosso cliente http utilizando uma struct de configuração que tem valores padrão e permite que o usuário modifique apenas os valores que deseja. Porém ainda assim, o usuário precisa conhecer todos os campos da struct e seus valores padrão, o que pode ser um pouco confuso.

```go
type ClientConfig struct {
    BaseURL string
    ForceAttemptHTTP2 bool
    Timeout time.Duration
    MaxConnsPerHost int
    MaxIdleConns int
    MaxIdleConnsPerHost int
    TLSHandshakeTimeout time.Duration
}

func NewClient(cfg ClientConfig) *Client {
    if cfg.BaseURL == "" {
        cfg.BaseURL = "https://api.example.com"
    }

    if cfg.Timeout == 0 {
        cfg.Timeout = 10 * time.Second
    }

    if cfg.MaxConnsPerHost == 0 {
        cfg.MaxConnsPerHost = 5
    }

    if cfg.MaxIdleConns == 0 {
        cfg.MaxIdleConns = 2
    }

    if cfg.MaxIdleConnsPerHost == 0 {
        cfg.MaxIdleConnsPerHost = 2
    }

    if cfg.TLSHandshakeTimeout == 0 {
        cfg.TLSHandshakeTimeout = 5 * time.Second
    }

    return &Client{
        baseURL: cfg.BaseURL,
        client: &http.Client{
            Timeout: cfg.Timeout,
            Transport: &http.Transport{
                ForceAttemptHTTP2: cfg.ForceAttemptHTTP2,
                MaxConnsPerHost: cfg.MaxConnsPerHost,
                MaxIdleConns: cfg.MaxIdleConns,
                MaxIdleConnsPerHost: cfg.MaxIdleConnsPerHost,
                TLSHandshakeTimeout: cfg.TLSHandshakeTimeout,
            },
        },
    }
}
```

## Options Pattern - Declaração

O padrão `Options Pattern` é uma forma de encapsular as "opções" de configuração em funções que podem ser passadas para o construtor da struct. Isso permite que o usuário configure apenas os valores que deseja, sem precisar conhecer todos os campos da struct.

```go
type Client struct {
    baseURL string
    client *http.Client
}

type Option func(*Client)

func NewClient(baseURL, opts ...Option) *Client {
    c := &Client{
        baseURL: baseURL,
        client: &http.Client{
            Timeout: time.Duration(10) * time.Second,
            Transport: &http.Transport{
                ForceAttemptHTTP2: true,
                MaxConnsPerHost: 5,
                MaxIdleConns: 2,
                MaxIdleConnsPerHost: 2,
                TLSHandshakeTimeout: time.Duration(5) * time.Second,
            },
        },
    }

    for _, option := range opts {
        option(c)
    }

    return c
}
```

## Options Pattern - Implementação

Agora podemos criar funções que modificam os valores da struct `Client` de forma flexível e extensível. Essas funções são chamadas de "opções" e podem ser passadas para o construtor da struct do cliente http.

```go
// WithTimeout é uma opção que define o tempo limite para as requisições HTTP.
func WithTimeout(timeout time.Duration) Option {
    return func(c *Client) {
        c.client.Timeout = timeout
    }
}

// WithHostname é uma opção que define o header X-Machine-Name com o hostname informado.
func WithHostname(hostname string) Option {
    return func(c *Client) {
        next := c.doFunc
        c.doFunc = func(c *Client, req *http.Request) (*http.Response, error) {
            hostname = url.QueryEscape(hostname)
            req.Header.Set("X-Machine-Name", hostname)

            return next(c, req)
        }
    }
}
```

## Options Pattern - Controle de Erros

É muito importante validar os erros que fazem sentido na "opção" que está sendo aplicada. Por exemplo, se o usuário passar um certificado SSL inválido, devemos retornar um erro.

```go
// WithSSLCertFile sobrescreve os certificados padrões para o certificado passado por parametro.
func WithSSLCertFile(ctx context.Context, filepath string) (Option, error) {
    caCert, err := file.ReadHead(ctx, filepath, 0)
    if err != nil {
        return nil, err
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    return WithSSLCertPool(caCertPool), nil
}

withSSLCert, err := WithSSLCertFile(context.Background(), "/path/to/cert.pem")
if err != nil {
    panic(err) // não faça isso em casa
}
```

## Na Prática

Aqui temos um exemplo completo utilizando as "opcões" que criamos ao longo deste post. Vale ressaltar que a única "opção" que retorna erro é de sobrescrever os certificados, e por isso que validamos o erro apenas dela. Agora vamos ver como ficaria a implementação do cliente http utilizando tudo que aprendemos aqui.

```go
func DoSomething() {
    opts := []Option{
        WithTimeout(20 * time.Second),
        WithHostname("golang-flow"),
    }

    withSSLCertFile, err := WithSSLCertFile(context.Background(), "/path/to/cert.pem")
    if err != nil {
        panic(err) // não faça isso em casa
    }

    opts = append(opts, withSSLCertFile)

    cli := NewClient("https://api.example.com", opts...)

    // Do ...
}
```

## Testes Unitários

Criar testes unitários para as "opções" é muito simples, vamos ver como fazer isso.

```go
// setupTestServer configura um servidor de teste para simular requisições HTTP.
func setupTestServer() (string, *http.ServeMux, func()) {
    router := http.NewServeMux()
    srv := httptest.NewServer(router)

    return srv.URL, router, func() { srv.Close() }
}

// TestOption_WithTimeout testa a opção WithTimeout para o cliente http.
func TestOption_WithHostname(t *testing.T) {
    url, router, tearDown := setupTestServer()
    defer tearDown()

    router.HandleFunc("/", func(_ http.ResponseWriter, req *http.Request) {
        // Verifica se o header X-Machine-Name foi adicionado corretamente
        assert.Equal(t, []string{"my-computer"}, req.Header["X-Machine-Name"])
    })

    // Cria o slice de opções com a opção WithHostname
    opts := []Option{WithHostname("my-computer")}

    req, err := http.NewRequest(http.MethodGet, url, nil)
    require.NoError(t, err)

    // Cria o cliente com a URL do servidor de teste e as opções
    c := NewClient("https://api.example.com", opts...)

    resp, err := c.Do(t.Context(), req)
    require.NoError(t, err)

    defer resp.Body.Close()
}

// TestOption_WithSSLCertFile testa a opção WithSSLCertFile para o cliente http e valida se o certificado SSL foi aplicado corretamente.
func TestOption_WithSSLCertFile(t *testing.T) {
    url, router, tearDown := setupTestServer()
    defer tearDown()

    router.HandleFunc("/", func(_ http.ResponseWriter, req *http.Request) {
        // Verifica se o certificado SSL foi aplicado corretamente
        assert.NotNil(t, req.TLS)
        assert.NotEmpty(t, req.TLS.PeerCertificates)
    })

    caCertFile := "testdata/ca_cert.pem"
    withSSLCertFile, err := WithSSLCertFile(ctx, caCertFile)
    require.NoError(t, err)

    // Cria o slice de opções com a opção WithSSLCertFile
    opts := []Option{withSSLCertFile}

    req, err := http.NewRequest(http.MethodGet, url, nil)
    require.NoError(t, err)

    // Cria o cliente com a URL do servidor de teste e as opções
    c := NewClient("https://api.example.com", opts...)

    resp, err := c.Do(ctx, req)
    require.NoError(t, err)

    defer resp.Body.Close()
}
```
