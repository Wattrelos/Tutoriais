------------------------------
## Guia Definitivo: Instalação e Configuração do Forgejo no Debian 13
Este documento serve como guia de implantação para o servidor Git Forgejo rodando de forma nativa no Debian 13 (Trixie), aproveitando a infraestrutura existente de Apache2 e MariaDB, integrado ao ambiente de desenvolvimento Antigravity.
------------------------------
## 1. Configuração do Banco de Dados (MariaDB)
Acesse o console do MariaDB como root:

sudo mysql -u root -p

Execute os comandos SQL para criar a estrutura isolada para o Forgejo (substitua SuaSenhaSegura por uma senha forte):

CREATE DATABASE forgejo CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';CREATE USER 'forgejo'@'localhost' IDENTIFIED BY 'SuaSenhaSegura';GRANT ALL PRIVILEGES ON forgejo.* TO 'forgejo'@'localhost';
FLUSH PRIVILEGES;
EXIT;

------------------------------
## 2. Estrutura do Sistema e Usuário Linux
Para garantir a segurança do sistema e padronizar as URLs de clonagem SSH, criamos um usuário isolado chamado git. Isso não causa conflitos com o comando git do terminal, pois um é um usuário do sistema e o outro é o executável.

# Criar o usuário de sistema 'git'
sudo adduser --system --shell /bin/bash --gegroup --group --disabled-password --home /home/git git
# Criar os diretórios de dados, logs e configurações do Forgejo
sudo mkdir -p /var/lib/forgejo /etc/forgejo /var/log/forgejo
# Atribuir as permissões das pastas ao usuário git
sudo chown -R git:git /var/lib/forgejo /etc/forgejo /var/log/forgejo

------------------------------
## 3. Download do Binário Oficial do Forgejo
Como o Forgejo é um binário único escrito em Go, a instalação nativa é feita baixando o executável oficial:

# Baixar o binário estável para Linux AMD64
sudo wget https://forgejo.org -O /usr/local/bin/forgejo
# Conceder permissão de execução ao arquivo
sudo chmod +x /usr/local/bin/forgejo

------------------------------
## 4. Gerenciamento do Serviço com Systemd
Crie um arquivo de serviço para que o Debian gerencie o Forgejo em segundo plano e garanta que ele só inicie após o MariaDB estar ativo.

sudo nano /etc/systemd/system/forgejo.service

Cole o conteúdo abaixo:

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
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/forgejo

[Install]
WantedBy=multi-user.target

Recarregue os daemons do sistema, ative a inicialização automática e ligue o serviço:

sudo systemctl daemon-reload
sudo systemctl enable --now forgejo

------------------------------
## 5. Configuração do Proxy Reverso no Apache2
Como a porta padrão 80 do seu servidor já está ocupada, o Apache foi configurado para responder pelo Forgejo na porta 8095, fazendo a ponte com o serviço interno do Forgejo (porta 3000).
## 5.1 Adicionar a porta de escuta global
Abra o arquivo de portas do Apache:

sudo nano /etc/apache2/ports.conf

Abaixo da linha Listen 80, adicione a instrução para o Apache escutar a nova porta:

Listen 8095

## 5.2 Criar o VirtualHost do Forgejo
Crie o arquivo de configuração do site:

sudo nano /etc/apache2/sites-available/forgejo.conf

Cole o conteúdo abaixo (atente-se para o ServerName que deve conter apenas o host, sem prefixos ://):

<VirtualHost *:8095>
    ServerName localhost

    ProxyPreserveHost On
    ProxyRequests Off
    AllowEncodedSlices NoDecode

    <Location />
        Order allow,deny
        Allow from all
        ProxyPass http://localhost:3000/ nocanon
        ProxyPassReverse http://localhost:3000/
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/forgejo_error.log
    CustomLog ${APACHE_LOG_DIR}/forgejo_access.log combined
</VirtualHost>

## 5.3 Ativar módulos e reiniciar o Apache

# Ativar os módulos de Proxy do Apache
sudo a2enmod proxy proxy_http
# Ativar o VirtualHost do Forgejo
sudo a2ensite forgejo.conf
# Testar a integridade da sintaxe do Apache
sudo apache2ctl configtest
# Se retornar "Syntax OK", reinicie o servidor web
sudo systemctl restart apache2

------------------------------
## 6. Configuração de Segurança no Firewall (UFW)
Para manter o servidor seguro, liberamos apenas a porta de entrada pública do Apache (8095) e a porta SSH padrão (22) para conexões Git. A porta interna 3000 permanece bloqueada para o mundo externo.

# Liberar acesso web do Forgejo via Apache
sudo ufw allow 8095/tcp comment 'Forgejo via Apache'
# Liberar porta SSH padrão para clonagem/push seguro
sudo ufw allow 22/tcp comment 'Acesso SSH / Git Clone'
# Aplicar as novas regras
sudo ufw reload

------------------------------
## 7. Assistente de Instalação Web
Abra o navegador e acesse: http://IP_DO_SEU_SERVIDOR:8095
Preencha os campos estritamente com as definições do ambiente local construído:

* Tipo de Banco de Dados: MySQL / MariaDB
* Host do Banco de Dados: 127.0.0.1:3306
* Usuário do Banco de Dados: forgejo
* Senha do Banco de Dados: (A senha definida no Passo 1)
* Nome do Banco de Dados: forgejo
* Executar como usuário: git (Mantido para padronizar URLs)
* URL Base: http://IP_DO_SEU_SERVIDOR:8095/

Importante: Vá até o rodapé da página, expanda a opção Configurações de Conta de Administrador e crie o seu usuário Master (evite usar "admin" por segurança). Finalize clicando em Instalar Forgejo.

------------------------------
## 8. Integração e Conexão com o Antigravity
Para enviar códigos da IDE Antigravity para o seu servidor Forgejo, a autenticação baseada em tokens deve ser utilizada.
## 8.1 Criando o Token com Privilégios Mínimos

   1. No painel web do Forgejo, clique na sua foto de perfil (canto superior direito) > Configurações > Aplicativos.
   2. Em Gerenciar Tokens de Acesso, dê um nome ao token (ex: antigravity-ide).
   3. Modifique as seguintes permissões (restando as outras como Sem acesso):
   * repository (Repositório): Alternar para Escrita (Permite dar git push, criar ramificações e enviar código).
      * user (Usuário): Alternar para Leitura (Permite que o Antigravity valide sua identidade).
   4. Clique em Gerar Token e copie o código gerado.

## 8.2 Fazendo o primeiro Upload via Antigravity
Com o token copiado, crie um repositório vazio na interface web do Forgejo. No terminal integrado da IDE Antigravity, navegue até a pasta do seu código local e rode os comandos de envio:

# Inicializar o repositório local
git init -b main
git add .
git commit -m "Commit Inicial pelo Antigravity"
# Vincular ao servidor Forgejo na porta correta
git remote add origin http://IP_DO_SEU_SERVIDOR:8095/seu-usuario/nome-do-repo.git
# Enviar os dados
git push -u origin main

Quando o terminal solicitar as credenciais:

* Username: O seu nome de usuário do Forgejo.
* Password: Cole o Token de Acesso gerado no passo 8.1.

------------------------------
