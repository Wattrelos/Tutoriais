# Preparando uma Rust Web Service
> Em um programa em Rust, o ponto de entrada principal geralmente fica no arquivo src/main.rs, dentro de uma função chamada main(). Diferente do PHP, onde o servidor web costuma procurar por um arquivo chamado index.php para iniciar a execução da página, o Rust é uma linguagem compilada. O compilador do Rust (rustc ou o gerenciador de pacotes cargo) procura por esse arquivo e função específicos para gerar o arquivo binário executável final.

## Estrutura básica de um projeto Rust
Quando você cria um novo projeto executável em Rust usando o comando cargo new nome_do_projeto, a estrutura padrão gerada é a seguinte:

* Cargo.toml: O arquivo de configuração do projeto (onde ficam as dependências e metadados).
* src/: A pasta que contém o código-fonte.
* main.rs: O ponto de entrada do teu programa.

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
Em Rust, o teu programa (utilizando o Axum e o Tokio) será compilado em um arquivo binário executável independente. Quando você roda esse executável, ele se torna o seu próprio servidor web, rodando em uma porta interna (como a 3000).
O papel do Nginx aqui será o de um Proxy Reverso: ele irá receber as requisições da internet (portas 80 ou 443) e repassá-las para a porta interna onde o teu executável Rust está rodando.
------------------------------
## O Fluxo Correto de Configuração

### 1. Compilar o projeto Rust

Primeiro, você precisa gerar o binário de produção (otimizado). Dentro da raiz do teu projeto, rode:

```bash
cargo build --release
```

Isso irá gerar um arquivo executável dentro de `target/release/nome_do_teu_projeto`.

### 2. Rodar o executável em segundo plano
Você deve executar esse binário no teu servidor. Para garantir que ele continue rodando mesmo se você fechar o terminal, o ideal é criar um serviço no sistema (usando o systemd no Linux) ou usar um gerenciador de processos.
Teu app Rust irá subir e ficar escutando uma porta, por exemplo: http://127.0.0.1:3000.
## 3. Configurar o Nginx como Proxy Reverso
Agora sim, você configura o Nginx. No arquivo de configuração do teu site no Nginx (geralmente em /etc/nginx/sites-available/default), você irá apontar para a porta interna do Rust, e não para a pasta do código:
```nginx
server {
    listen 80;
    server_name teu-dominio.com; # Ou o IP do teu servidor

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
```

## Criar o arquivo de serviço do Systemd para manter o teu app Rust rodando 24/7.

> Para garantir que teu Web Service em Rust rode continuamente em segundo plano na tua VPS Linux e reinicie automaticamente caso o servidor seja reiniciado ou o app sofra uma queda rara, o Systemd é a ferramenta nativa ideal.
Aqui está o passo a passo direto para criar e ativar esse arquivo de serviço:

## 1. Criando o arquivo de configuração
No terminal da tua VPS, crie o arquivo de serviço (geralmente nomeado com o nome do teu app) usando um editor como o nano:

```bash
sudo nano /etc/systemd/system/meu-ecommerce.service
```

Cole o seguinte conteúdo dentro dele (ajuste os caminhos e usuários conforme o teu servidor):

```bash
[Unit]
Description=Web Service de E-commerce em Rust (Axum)
After=network.target

[Service]
# Usuário e grupo do Linux que vão rodar o app (evite usar o root por segurança)
User=www-data
Group=www-data

# O diretório onde está o teu executável e o teu arquivo .env
WorkingDirectory=/var/www/meu-ecommerce

# O caminho absoluto para o teu executável Rust
ExecStart=/var/www/meu-ecommerce/meu-app-rust

# Estratégia de reinicialização automática em caso de falha
Restart=always
RestartSec=5

# Captura os logs do Tracing e envia para o sistema de logs do Linux
StandardOutput=journal
StandardError=journal
SyslogIdentifier=meu-ecommerce

# Limita o uso de memória ou arquivos se desejar (opcional)
Environment=RUST_LOG=info

[Install]
WantedBy=multi-user.target
```

Nota: Salve o arquivo pressionando Ctrl + O, Enter e saia com Ctrl + X.

## 2. Ativando e iniciando o serviço
Agora que o arquivo foi criado, você precisa dizer ao Linux para ler a nova configuração e iniciar o teu e-commerce:

# 1. Atualiza o Systemd com o novo arquivo de serviço

```bash
sudo systemctl daemon-reload
```

# 2. Ativa o serviço para que ele inicie automaticamente junto com a VPS

```bash
sudo systemctl enable meu-ecommerce.service
```

# 3. Inicia o teu e-commerce Rust agora mesmo

```bash
sudo systemctl start meu-ecommerce.service
```

## 3. Comandos úteis para o dia a dia

* Ver o status do app: Para checar se ele está rodando perfeitamente e qual porta está consumindo:

```bash
sudo systemctl status meu-ecommerce.service
```

* Ver os logs em tempo real (Graças ao Tracing): Como você usa a crate tracing, todos os logs do teu app vão direto para o gerenciador de logs do Linux. Você pode assisti-los ao vivo com:

```bash
sudo journalctl -u meu-ecommerce.service -f
```

* Reiniciar o app após uma atualização: Quando você enviar um executável novo para a VPS, basta rodar:

```bash
sudo systemctl restart meu-ecommerce.service
```







