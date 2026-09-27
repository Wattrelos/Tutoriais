# Preparando uma Rust Web Service
> Em um programa em Rust, o ponto de entrada principal geralmente fica no arquivo src/main.rs, dentro de uma função chamada main(). Diferente do PHP, onde o servidor web costuma procurar por um arquivo chamado index.php para iniciar a execução da página, o Rust é uma linguagem compilada. O compilador do Rust (rustc ou o gerenciador de pacotes cargo) procura por esse arquivo e função específicos para gerar o arquivo binário executável final.

## Estrutura básica de um projeto Rust
Quando você cria um novo projeto executável em Rust usando o comando cargo new nome_do_projeto, a estrutura padrão gerada é a seguinte:

* Cargo.toml: O arquivo de configuração do projeto (onde ficam as dependências e metadados).
* src/: A pasta que contém o código-fonte.
* main.rs: O ponto de entrada do seu programa.

## O código de entrada
Dentro do arquivo src/main.rs, o menor código válido para iniciar o programa se parece com isto:

fn main() {
    println!("Olá, mundo!");
}

## E se for uma biblioteca (Library)?
_Se você estiver criando uma biblioteca em Rust (um código feito para ser reutilizado por outros programas e não para ser executado sozinho), o ponto de entrada padrão muda para src/lib.rs. Nesse caso, geralmente não há uma função main().
Se você quiser, posso te mostrar como alterar esse ponto de entrada padrão no arquivo Cargo.toml ou explicar como funciona a estrutura de módulos para organizar códigos maiores além do arquivo main.rs._

## Proxy Reverso Nginx

Diferente do PHP, onde o Nginx lê os arquivos de texto .php diretamente da pasta usando o PHP-FPM, o Rust é uma linguagem compilada. O arquivo main.rs é apenas o código-fonte; o Nginx não sabe o que fazer com ele.
Em Rust, o seu programa (utilizando o Axum e o Tokio) será compilado em um arquivo binário executável independente. Quando você roda esse executável, ele se torna o seu próprio servidor web, rodando em uma porta interna (como a 3000).
O papel do Nginx aqui será o de um Proxy Reverso: ele vai receber as requisições da internet (portas 80 ou 443) e repassá-las para a porta interna onde o seu executável Rust está rodando.
------------------------------
## O Fluxo Correto de Configuração

### 1. Compilar o projeto Rust

Primeiro, você precisa gerar o binário de produção (otimizado). Dentro da raiz do seu projeto, rode:

```bash
cargo build --release
```

Isso vai gerar um arquivo executável dentro de `target/release/nome_do_seu_projeto`.

### 2. Rodar o executável em segundo plano
Você deve executar esse binário no seu servidor. Para garantir que ele continue rodando mesmo se você fechar o terminal, o ideal é criar um serviço no sistema (usando o systemd no Linux) ou usar um gerenciador de processos.
Seu app Rust vai subir e ficar escutando uma porta, por exemplo: http://127.0.0.1:3000.
## 3. Configurar o Nginx como Proxy Reverso
Agora sim, você configura o Nginx. No arquivo de configuração do seu site no Nginx (geralmente em /etc/nginx/sites-available/default), você vai apontar para a porta interna do Rust, e não para a pasta do código:
```nginx
server {
    listen 80;
    server_name seu-dominio.com; # Ou o IP do seu servidor

    location / {
        # Transfere a requisição para o Axum rodando na porta 3000
        proxy_pass http://127.0.0.1:3000; 
        
        # Cabeçalhos padrão para repassar o IP real do usuário ao Rust
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Depois de salvar, basta testar e reiniciar o Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx

Se você quiser avançar, posso te ajudar com:

## Criar o arquivo de serviço do Systemd para manter o seu app Rust rodando 24/7.

```bash





