# Web service with Rust

> Com essa pilha tecnológica (Axum, Tower, Tokio e Tracing), tua aplicação de e-commerce é definitivamente um Web Service moderno e assíncrono.
Essa combinação é o padrão ouro atual no ecossistema Rust para construir microsserviços e APIs robustas de altíssima performance.
## O papel de cada componente no teu Web Service

* Axum (O Framework Web): É ele quem define as rotas da tua API (ex: rotas para criar pedidos, listar produtos) e lida com as requisições e respostas HTTP. Ele foi projetado para se integrar perfeitamente com o ecossistema Tokio.
* Tower (A Infraestrutura de Serviços): Fornece a abstração de middlewares. É o Tower que permite adicionar facilmente camadas essenciais para um serviço web de produção, como controle de taxa (rate limiting), autenticação, timeouts e gerenciamento de CORS.
* Tokio (O Motor Assíncrono): É o runtime que executa tua aplicação. Ele garante que teu e-commerce consiga lidar com milhares de conexões simultâneas de clientes comprando ou navegando ao mesmo tempo, sem travar o servidor (I/O não-bloqueante).
* Tracing (O Sistema de Observabilidade): Essencial para um Web Service em produção. Ele gera logs estruturados e rastreia o fluxo de uma requisição de e-commerce (ex: rastrear desde o clique de "comprar" até a chamada do banco de dados), facilitando o diagnóstico de lentidões ou erros.

## Como teu e-commerce opera como Web Service
A tua aplicação provavelmente funciona de forma desacoplada: o teu front-end (seja um aplicativo mobile, um site em React/Next.js ou uma aplicação WebAssembly) envia dados estruturados (geralmente JSON) para o teu servidor Rust. O Axum processa as regras de negócio do e-commerce (validação de estoque, cálculo de frete, checkout) e devolve uma resposta rápida e limpa.
Para ajudar você a estruturar melhor esse serviço, me diga:


## Compilação para Web Service
O Rust compila teu código diretamente em um único arquivo executável binário nativo (como um .exe no Windows ou um arquivo binário sem extensão no Linux e macOS).
Essa é uma das grandes vantagens de usar Rust para Web Services em comparação com outras linguagens:

* Sem máquina virtual: Diferente do Java (que precisa da JVM) ou do C# (que precisa do .NET Runtime), o executável do Rust roda direto no sistema operacional, sem intermediários.
* Sem interpretador: Diferente do Node.js (JavaScript) ou Python, você não precisa instalar a linguagem no servidor para rodar o e-commerce.
* Tudo incluso: O arquivo gerado já contém o Axum, o Tokio, as regras do teu e-commerce e tudo o que ele precisa para funcionar.

## Como isso funciona na prática para o teu Web Service?

   1. No desenvolvimento, você usa o comando cargo build --release.
   2. O Rust gera o arquivo executável na pasta target/release/.
   3. Para colocar o teu e-commerce no ar (em produção), basta copiar apenas esse arquivo executável para o servidor ou para dentro de um container Docker super leve e executá-lo. Ele abrirá a porta de rede configurada e começará a escutar as requisições HTTP imediatamente.

# Servidor Web Nxinx com Rust

A estratégia padrão de mercado da indústria para colocar aplicações Rust em produção com segurança e alta disponibilidade é usar um arquivo .env para as credenciais mantém teu código limpo, e colocar o Nginx como proxy reverso traz enormes vantagens de segurança e performance para o teu e-commerce.

## Arquitetura do teu E-commerce na VPS
No teu servidor VPS, o fluxo de uma requisição de um cliente funcionará exatamente assim:

Cliente ──( HTTPS / Porta 443 )──> Nginx ──( HTTP / Porta Local 3000 )──> teu Executável Rust

## Por que usar o Nginx como Proxy Reverso com Rust?

* Gerenciamento de SSL/TLS (HTTPS): Em vez de configurar os certificados do Let's Encrypt direto no teu código Rust (via Axum), você deixa o Nginx gerenciar o HTTPS. Ele é extremamente eficiente fazendo isso.
* Segurança Adicional: O Nginx esconde o teu executável Rust da internet pública. Ele pode bloquear ataques comuns, filtrar requisições maliciosas e proteger teu Web Service.
* Arquivos Estáticos Super Rápidos: Se o teu e-commerce tiver imagens de produtos, banners, CSS ou JavaScript, o Nginx pode entregar esses arquivos direto do disco rígido muito mais rápido do que repassar a requisição para o Rust, liberando tua aplicação para focar apenas nas vendas e banco de dados.

## Como o Axum lê o teu arquivo .env
No ecossistema Rust, a forma mais comum e eficiente de carregar esse arquivo .env que você criará na instalação é usando a biblioteca dotenvy.
No início da tua função main, basta adicionar uma linha para carregar as variáveis para o ambiente do sistema, e depois ler a porta ou a URL do banco de dados normalmente:
```rust
#[tokio::main]async fn main() {
    // Carrega o arquivo .env automaticamente
    dotenvy::dotenv().ok();

    // Lê a porta configurada ou usa a 3000 como padrão
    let porta = std::env::var("PORT").unwrap_or_else(|_| "3000".to_string());
    
    // ... restante da inicialização do Axum
}
```

