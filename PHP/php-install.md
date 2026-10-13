# Como Instalar o PHP no Debian 13 (Trixie)

> Guia passo a passo completo para instalação, configuração de múltiplas versões e integração com Apache e Nginx no Debian 13.

---

## 📌 Visão Geral

No **Debian 13 (Trixie)**, o PHP nativo nos repositórios oficiais é o **PHP 8.4**.

Dependendo da sua necessidade, você pode escolher entre dois métodos:
- **Método 1 (Oficial):** Recomendado para quem busca estabilidade e suporte nativo da distribuição (PHP 8.4).
- **Método 2 (Repositório SURY):** Recomendado caso precise de versões legadas (como PHP 8.1, 8.2, 8.3) ou versões mais recentes/específicas (PHP 8.5) gerenciadas lado a lado.

---

## Método 1: Instalação Padrão (PHP 8.4 via Repositório Oficial)

Este é o método mais direto e recomendado para a maioria dos ambientes.

### 1. Atualizar o sistema

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Instalar o PHP e extensões comuns

O meta-pacote `php` instalará a versão padrão do Debian 13 (PHP 8.4) juntamente com o `php-cli` e o `php-fpm`:

```bash
sudo apt install -y php php-cli php-fpm php-common \
    php-mbstring php-xml php-zip php-curl php-mysql \
    php-gd php-intl php-bcmath php-sqlite3
```

### 3. Verificar a instalação

```bash
php -v
```

Para verificar quais módulos estão ativos:

```bash
php -m
```

---

## Método 2: Instalação de Versões Específicas via Repositório SURY (PHP 8.2, 8.3, 8.5)

O repositório mantido por **Ondřej Surý** é o padrão de fato da comunidade Debian/Ubuntu para suporte a múltiplas versões do PHP.

### 1. Instalar dependências prévias

```bash
sudo apt update
sudo apt install -y lsb-release ca-certificates apt-transport-https software-properties-common curl gnupg
```

### 2. Importar a chave GPG e adicionar o repositório

```bash
# Baixar e salvar o chaveiro GPG oficial
sudo curl -sSLo /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg

# Adicionar a lista de fontes do repositório
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
```

### 3. Atualizar o índice de pacotes

```bash
sudo apt update
```

### 4. Instalar a versão desejada

Substitua `X.Y` pela versão necessária (exemplo: `8.3` ou `8.5`):

#### Exemplo: PHP 8.3
```bash
sudo apt install -y php8.3 php8.3-cli php8.3-fpm php8.3-common \
    php8.3-mbstring php8.3-xml php8.3-zip php8.3-curl php8.3-mysql \
    php8.3-gd php8.3-intl php8.3-bcmath php8.3-sqlite3
```

#### Alternar entre versões instaladas (CLI)
Se você tiver mais de uma versão instalada no sistema, utilize o `update-alternatives`:

```bash
sudo update-alternatives --config php
```

---

## 🌐 Configuração com Servidores Web

### Opção A: Apache (HTTPD)

#### 1. Usando PHP-FPM (Recomendado para melhor performance e compatibilidade HTTP/2)

Ative os módulos FastCGI e a configuração do FPM correspondente:

```bash
# Ativar módulos necessários no Apache
sudo a2enmod proxy_fcgi setenvif

# Ativar a configuração da sua versão (ajuste se estiver usando 8.3 ou 8.4)
sudo a2enconf php8.4-fpm

# Reiniciar o Apache
sudo systemctl restart apache2
```

#### 2. Verificar o status do serviço PHP-FPM

```bash
sudo systemctl status php8.4-fpm --no-pager
```

---

### Opção B: Nginx

O Nginx requer o **PHP-FPM** para processar scripts PHP. 

1. Certifique-se de que o serviço FPM está em execução:
```bash
sudo systemctl enable --now php8.4-fpm
```

2. Edite o arquivo de configuração do seu site (ex: `/etc/nginx/sites-available/default`):

```nginx
server {
    listen 80;
    server_name seu_dominio.com;
    root /var/www/html;

    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        
        # Ajuste o caminho do socket para a sua versão:
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

3. Teste a sintaxe do Nginx e recarregue o serviço:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🧪 Testando o Ambiente PHP

Para confirmar que seu servidor web está processando o PHP corretamente:

1. Crie um arquivo de teste na raiz do seu site:

```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
```

2. Acesse via navegador: `http://ip-do-servidor/info.php` ou `http://localhost/info.php`.

> ⚠️ **Atenção:** Por motivos de segurança, remova o arquivo `info.php` após o teste para evitar expor dados da máquina:
> ```bash
> sudo rm /var/www/html/info.php
> ```

---

## 🛠️ Ajustes Recomendados no `php.ini`

Localizações padrão do arquivo de configuração:
- **PHP CLI:** `/etc/php/8.4/cli/php.ini`
- **PHP-FPM:** `/etc/php/8.4/fpm/php.ini`
- **Apache (mod_php):** `/etc/php/8.4/apache2/php.ini`

Configurações comuns para ajustar conforme a demanda do projeto:

```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
date.timezone = America/Sao_Paulo
```

Após editar `/etc/php/8.4/fpm/php.ini`, reinicie o serviço:

```bash
sudo systemctl restart php8.4-fpm
```

---

## 📚 Próximos Passos & Referências

- Para gerenciar bibliotecas e dependências dos seus projetos, siga nosso guia de [Instalação do Composer](Composer-install.md).
- Repositório mantido por [Ondřej Surý](https://deb.sury.org/).
