# 📋 Matriz de Requisitos de Infraestrutura e Ambiente (Alpha Engine)

Este documento especifica os requisitos mínimos e recomendados de sistema, linguagens, extensões, serviços de banco de dados e permissões para instalação e execução da **Alpha Engine**.

---

## 1. Tabela Rápida de Requisitos

| Componente | Versão Mínima | Versão Recomendada | Status | Finalidade Principal |
| :--- | :---: | :---: | :---: | :--- |
| **PHP** | 8.1.0 | 8.2+ ou 8.4 | **Obrigatório** | Runtime da aplicação e POPOs tipados |
| **Composer** | 2.2+ | 2.8+ | **Obrigatório** | Gestão de pacotes e autoload PSR-4 |
| **MariaDB / MySQL** | 10.4+ / 8.0+ | 10.11 LTS / 8.4 LTS | **Obrigatório** | Banco relacional com suporte a `utf8mb4` |
| **Servidor Web** | Apache 2.4 / Nginx 1.18 | Apache 2.4+ / Nginx 1.24+ | **Obrigatório** | Servir arquivos estáticos e rotear para `index.php` |
| **Redis** | 6.0+ | 7.0+ | *Recomendado* | Cache distribuído de sessões e rate limiting |

---

## 2. Extensões do PHP

### 🔴 Extensões Obrigatórias (O Setup Wizard bloqueia se faltar)

| Extensão | Função no Sistema |
| :--- | :--- |
| `pdo` | Camada de abstração de conexão e transações atômicas (Unit of Work). |
| `pdo_mysql` | Driver nativo para comunicação com MariaDB / MySQL. |
| `mbstring` | Suporte a strings multi-byte, UTF-8 e internacionalização. |
| `gd` ou `imagick` | Redimensionamento, compressão e geração de thumbnails de imagens. |
| `xml` / `dom` | Manipulação de XML, feeds RSS e sitemaps automatizados. |
| `curl` | Integrações com gateways de pagamento, APIs de frete e Webhooks. |
| `zip` | Extração de pacotes de release e exportações de relatórios. |
| `openssl` | Criptografia de tokens JWT, hashes `ARGON2ID` e assinaturas HMAC. |
| `fileinfo` | Validação segura de tipos MIME em uploads de imagens e documentos. |
| `json` | Serialização de payloads de API e respostas JSON RESTful. |

### 🟡 Extensões Recomendadas

| Extensão | Benefício |
| :--- | :--- |
| `intl` | Formatação regional de moedas, números e datas conforme o idioma ativo. |
| `bcmath` | Cálculos monetários e de taxas com precisão arbitrária de ponto flutuante. |
| `redis` | Alta performance para cache L2 e filas assíncronas. |
| `opcache` | Aceleração de execução com bytecode pré-compilado em produção. |

---

## 3. Servidor Web & Roteamento

### Apache 2.4+
- Módulo `mod_rewrite` ativado (`sudo a2enmod rewrite`).
- Diretiva `AllowOverride All` habilitada no VirtualHost para interpretação do `.htaccess`.
- Módulo PHP habilitado (`libapache2-mod-php` ou `php-fpm` via `proxy_fcgi`).

### Nginx
Configuração necessária no bloco `location /`:
```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

---

## 4. Permissões de Diretórios e Arquivos

O usuário do servidor web (`www-data`, `nginx` ou `apache`) precisa de permissões de leitura e gravação nas seguintes pastas:

```bash
# Permissão para o diretório de arquivos voláteis e cache
chmod -R 775 /var/www/html/agsonhos/backend/storage/

# Permissão para criação/atualização atômica do arquivo de ambiente (.env)
chmod 775 /var/www/html/agsonhos/backend/
```

### Subdiretórios do `backend/storage/`:
* `cache/` (incluindo `cache/twig_slim/`, `cache/twig_setup/`, `cache/alpha_proxies/`)
* `download/`
* `logs/`
* `session/`
* `upload/`

---

## 5. Script de Pré-Checagem Automatizado (CLI)

Antes de abrir o navegador ou iniciar o Setup Wizard, você pode executar o script de diagnóstico automatizado fornecido na raiz do projeto:

```bash
./scripts/check_requirements.sh
```

O script analisará instantaneamente:
1. Versão e executável do PHP CLI.
2. Presença de todas as 10 extensões obrigatórias e 3 recomendadas.
3. Existência do Composer e integridade da pasta `backend/vendor/autoload.php`.
4. Disponibilidade do cliente e serviço do MariaDB/MySQL.
5. Permissões de escrita em `backend/storage/` e no `.env`.

---

## 6. Comandos Rápidos de Instalação dos Pacotes

### Ubuntu / Debian (PHP 8.2 / 8.4):
```bash
sudo apt update
sudo apt install -y php php-cli php-fpm php-mysql php-mbstring php-gd php-xml php-curl php-zip php-intl php-bcmath mariadb-server composer
```

### Inicialização do Banco de Dados:
```bash
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```
