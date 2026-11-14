# Tutorial: Instalação e Configuração do Forgejo no Debian 13 (Trixie)

> O **Forgejo** é uma plataforma de hospedagem de código Git autohospedada, leve, rápida e mantida pela comunidade (um fork orientado pelo ecossistema do Gitea/Codeberg). Este guia aborda a implantação completa e nativa do Forgejo no **Debian 13 (Trixie)**, utilizando **MariaDB** como banco de dados relacional, **Systemd** para gerenciamento do serviço, **Apache2** como Proxy Reverso (escutando na porta customizada `8095`), regras de firewall com **UFW** e integração segura via token com a IDE **Antigravity**.

---

## 1. Configuração do Banco de Dados (MariaDB)

O Forgejo armazena metadados de contas, repositórios, issues e pull requests no banco de dados.

### 1.1. Acessar o console do MariaDB
Acesse o cliente interativo com privilégios de administrador:

```bash
sudo mysql -u root -p
```

### 1.2. Criar a base e o usuário exclusivo
Execute as instruções SQL abaixo para criar o banco de dados em `utf8mb4` e o usuário dedicado ao serviço (substitua `'SuaSenhaSegura'` por uma credencial forte):

```sql
CREATE DATABASE forgejo CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
CREATE USER 'forgejo'@'localhost' IDENTIFIED BY 'SuaSenhaSegura';
GRANT ALL PRIVILEGES ON forgejo.* TO 'forgejo'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## 2. Estrutura do Sistema e Usuário Linux

Para garantir o princípio de menor privilégio e padronizar URLs de clone via SSH (ex: `git@servidor:usuario/repo.git`), criaremos um usuário de sistema dedicado chamado `git`.

> [!NOTE]
> Criar um usuário Linux chamado `git` não entra em conflito com o comando `git` do terminal, pois um representa a identidade do sistema operacional e o outro é o binário executável em `/usr/bin/git`.

### 2.1. Criar o usuário de sistema `git`
```bash
sudo adduser --system --shell /bin/bash --group --disabled-password --home /home/git git
```

### 2.2. Criar diretórios operacionais e definir permissões
O Forgejo necessita de diretórios específicos para dados persistentes (repositórios e anexos), configurações e logs:

```bash
# Criar as pastas do Forgejo
sudo mkdir -p /var/lib/forgejo /etc/forgejo /var/log/forgejo

# Atribuir posse ao usuário git
sudo chown -R git:git /var/lib/forgejo /etc/forgejo /var/log/forgejo
sudo chmod 750 /var/lib/forgejo /var/log/forgejo
```

---

## 3. Download e Instalação do Binário Oficial

O Forgejo é distribuído como um executável compilado em Go, sem necessidade de runtimes pesados.

### 3.1. Baixar o binário estável
Verifique os lançamentos oficiais no [Codeberg Releases do Forgejo](https://codeberg.org/forgejo/forgejo/releases) ou baixe a versão estável mais recente para Linux AMD64:

```bash
cd /tmp
# Baixar o executável oficial (exemplo: v10.0.1 estável)
curl -fsSL https://codeberg.org/forgejo/forgejo/releases/download/v10.0.1/forgejo-10.0.1-linux-amd64 -o forgejo

# Mover para o diretório de executáveis do sistema
sudo mv forgejo /usr/local/bin/forgejo

# Conceder permissão de execução
sudo chmod +x /usr/local/bin/forgejo
```

### 3.2. Validar a instalação
Verifique se o binário foi instalado e responde corretamente:

```bash
forgejo --version
```

---

## 4. Gerenciamento do Serviço com Systemd

Para que o Forgejo inicie automaticamente no boot do Debian 13 e se recupere caso pare inesperadamente, configuramos uma unidade do `systemd`.

### 4.1. Criar o arquivo de serviço
Crie o arquivo `/etc/systemd/system/forgejo.service`:

```bash
sudo nano /etc/systemd/system/forgejo.service
```

Cole a configuração abaixo:

```ini
[Unit]
Description=Forgejo (Beyond coding. We forge.)
After=syslog.target network.target mariadb.service

[Service]
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/forgejo/
RuntimeDirectory=forgejo
ExecStart=/usr/local/bin/forgejo web --config /etc/forgejo/app.ini
Restart=always
RestartSec=2s
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/forgejo

# Ajustes de segurança do processo
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
ProtectSystem=full
ProtectHome=read-only
ReadWritePaths=/var/lib/forgejo /var/log/forgejo /etc/forgejo

[Install]
WantedBy=multi-user.target
```

### 4.2. Ativar e iniciar o Forgejo
Recarregue as definições do Systemd e habilite o serviço:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now forgejo
```

Para conferir se o serviço está rodando e escutando na porta interna padrão (`3000`):

```bash
sudo systemctl status forgejo
```

---

## 5. Configuração do Proxy Reverso no Apache2

Como a porta padrão `80` já está ocupada por outras aplicações no servidor, o Apache receberá as requisições na porta **`8095`** e fará o encaminhamento interno (*reverse proxy*) transparente para o Forgejo na porta `3000`.

### 5.1. Habilitar os módulos de Proxy do Apache
```bash
sudo a2enmod proxy proxy_http headers rewrite
```

### 5.2. Adicionar a porta `8095` ao Apache
Abra o arquivo de portas do Apache:

```bash
sudo nano /etc/apache2/ports.conf
```

Abaixo da linha `Listen 80`, adicione a nova porta:

```apache
Listen 8095
```

### 5.3. Criar o VirtualHost do Forgejo
Crie o arquivo de definição do site em `/etc/apache2/sites-available/forgejo.conf`:

```bash
sudo nano /etc/apache2/sites-available/forgejo.conf
```

Cole a seguinte configuração:

```apache
<VirtualHost *:8095>
    ServerName localhost

    ProxyPreserveHost On
    ProxyRequests Off
    AllowEncodedSlices NoDecode

    <Location />
        Require all granted
        ProxyPass http://127.0.0.1:3000/ nocanon
        ProxyPassReverse http://127.0.0.1:3000/
    </Location>

    # Limite de upload de arquivos e commits grandes (ex: 512MB)
    LimitRequestBody 536870912

    ErrorLog ${APACHE_LOG_DIR}/forgejo_error.log
    CustomLog ${APACHE_LOG_DIR}/forgejo_access.log combined
</VirtualHost>
```

> [!NOTE]
> No Apache 2.4 (Debian 13), a diretiva moderna para controle de acesso é `Require all granted` (substituindo o antigo `Order allow,deny / Allow from all`).

### 5.4. Ativar o site e reiniciar o Apache
```bash
# Ativar o VirtualHost
sudo a2ensite forgejo.conf

# Validar se a sintaxe está íntegra
sudo apache2ctl configtest

# Reiniciar o serviço Apache se retornar 'Syntax OK'
sudo systemctl restart apache2
```

---

## 6. Configuração de Segurança no Firewall (UFW)

Proteja seu servidor expondo estritamente as portas de tráfego autorizadas. A porta interna `3000` do Forgejo permanecerá fechada para o mundo externo, recebendo conexões apenas vindas do Apache localmente (`127.0.0.1`).

```bash
# Liberar porta externa do Apache para acesso web ao Forgejo
sudo ufw allow 8095/tcp comment 'Forgejo Web via Apache'

# Liberar porta SSH padrão para clone e push via Git
sudo ufw allow 22/tcp comment 'SSH Git Access'

# Recarregar as regras do firewall
sudo ufw reload

# Conferir o status das regras ativas
sudo ufw status verbose
```

---

## 7. Assistente de Instalação Web

Abra o navegador e acesse a interface inicial:
```
http://IP_DO_SEU_SERVIDOR:8095
```

Preencha os campos do assistente conforme o ambiente configurado:

- **Tipo de Banco de Dados:** `MySQL` / `MariaDB`
- **Host do Banco de Dados:** `127.0.0.1:3306`
- **Usuário do Banco de Dados:** `forgejo`
- **Senha do Banco de Dados:** *(A senha definida no Passo 1.2)*
- **Nome do Banco de Dados:** `forgejo`
- **Título da Instância:** *(Ex: Meu Forgejo Git)*
- **Caminho Raiz dos Repositórios:** `/var/lib/forgejo/data/forgejo-repositories`
- **Executar como usuário:** `git`
- **Domínio do Servidor:** `IP_DO_SEU_SERVIDOR` ou domínio próprio
- **Porta HTTP de Escuta do Servidor:** `3000`
- **URL Base:** `http://IP_DO_SEU_SERVIDOR:8095/`

> [!IMPORTANT]
> Antes de concluir, role até o final da página e abra a seção **Configurações de Conta de Administrador**. Crie seu usuário Master (defina usuário, e-mail e senha forte; evite usar o nome genérico `admin`).
> Por fim, clique em **Instalar Forgejo**. O sistema salvará o arquivo de configuração em `/etc/forgejo/app.ini`.

---

## 8. Integração e Conexão com o Antigravity

A IDE Antigravity realiza operações de controle de versão (como clone, commit e push) com segurança através de tokens pessoais de acesso (*Personal Access Tokens*).

### 8.1. Gerar o Token de Acesso (Privilégios Mínimos)
1. No painel web do Forgejo, clique na sua foto de perfil (canto superior direito) e acesse: **Configurações** > **Aplicativos**.
2. Em **Gerenciar Tokens de Acesso**, defina um nome identificador (ex: `antigravity-ide`).
3. Configure os escopos com os privilégios recomendados:
   - **`repository` (Repositório):** Marcar como **Escrita** (`write`) — permite push, criação de branches e envio de código.
   - **`user` (Usuário):** Marcar como **Leitura** (`read`) — permite que o Antigravity valide a identidade do autor.
   - *(Deixar os demais recursos desmarcados / Sem acesso)*.
4. Clique em **Gerar Token** e guarde a sequência gerada em local seguro (ela é exibida apenas uma vez).

### 8.2. Primeiro Envio (Push) pelo Terminal do Antigravity
Com o token em mãos e um novo repositório criado na interface web do Forgejo, abra o terminal integrado do Antigravity na pasta do seu projeto local:

```bash
# Inicializar o repositório local
git init -b main

# Adicionar os arquivos e criar o primeiro commit
git add .
git commit -m "feat: commit inicial integrado via Antigravity"

# Vincular ao servidor Forgejo na porta 8095
git remote add origin http://IP_DO_SEU_SERVIDOR:8095/seu-usuario/nome-do-repo.git

# Enviar os arquivos para a branch principal
git push -u origin main
```

Quando o Git solicitar a autenticação:
- **Username:** Seu nome de usuário cadastrado no Forgejo.
- **Password:** Cole o **Token de Acesso** gerado no passo 8.1.

> [!TIP]
> Para evitar ter que colar o token a cada push no ambiente local do Antigravity, você pode ativar o cache seguro de credenciais do Git executando:
> ```bash
> git config --global credential.helper 'cache --timeout=28800'
> ```
