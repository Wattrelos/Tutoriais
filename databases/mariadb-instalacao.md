# Instalar e configurar o MariaDB
 
1. Instalação
Atualize os repositórios e instale o servidor e o cliente do MariaDB:
```bash
apt update && apt upgrade -y
apt install mariadb-server mariadb-client -y
```

### 2. Configuração de Segurança
- Após a instalação, execute o script de segurança para definir a senha do administrador e remover configurações padrão inseguras:
```bash
sudo mariadb-secure-installation
```
**Durante o assistente, recomenda-se:**
*   Definir uma senha forte para o usuário **root**.
*   Remover usuários anônimos (**Y**).
*   Desabilitar o login remoto do root (**Y**).
*   Remover o banco de dados de teste (**Y**).
*   Recarregar as tabelas de privilégios (**Y**).

### 3. Gerenciamento do Serviço
Certifique-se de que o banco de dados está ativo e configurado para iniciar com o sistema:
*   **Verificar status:** `systemctl status mariadb`
*   **Iniciar serviço:** `systemctl start mariadb`
*   **Habilitar no boot:** `systemctl enable mariadb`

### 4. Acesso ao Banco de Dados
No Debian moderno, o acesso inicial do root do sistema ao MariaDB geralmente utiliza o plugin `unix_socket`, permitindo acesso direto sem senha via sudo.
*   **Para acessar:** `mariadb` ou `mysql -u root -p`

### 5. Criar um Novo Usuário (Recomendado)
Para maior segurança, evite usar o root para tarefas diárias. Acesse o terminal do MariaDB e execute:
```sql
CREATE USER 'nome_usuario'@'localhost' IDENTIFIED BY 'sua_senha_forte';
GRANT ALL PRIVILEGES ON *.* TO 'nome_usuario'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```
# PHPMyAdmin no servidor Ngnix

- Se for usar phpMyAdmin no servidor Nginx, recomendo criar um arquivo de configuração separado para o phpMyAdmin no Nginx, pois é a melhor prática. Assim, ele fica totalmente independente do seu projeto agsonhos ou de qualquer outro site que você decida colocar ou tirar do ar no futuro.
- A forma mais organizada de fazer isso em um ambiente de desenvolvimento local é criar um domínio próprio para ele, como http://phpmyadmin.local.
Aqui está o passo a passo para isolar o phpMyAdmin:
## 1. Criar o arquivo de configuração independente
Crie um novo arquivo na pasta sites-available:
```bash
sudo nano /etc/nginx/sites-available/phpmyadmin
```
Cole o conteúdo abaixo dentro dele (lembre-se de ajustar a versão do seu PHP se for diferente de 8.2):

```nginx
server {
    listen 80;
    server_name phpmyadmin.local;

    # Pasta padrão onde o pacote do phpMyAdmin fica instalado no Ubuntu/Debian
    root /usr/share/phpmyadmin;

    index index.php index.html index.htm;

    access_log /var/log/nginx/phpmyadmin-access.log;
    error_log /var/log/nginx/phpmyadmin-error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    # Processamento PHP via PHP-FPM 8.4
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    }
}
```
## 2. Ativar a nova configuração
Crie o link simbólico para a pasta sites-enabled:
```bash
sudo ln -s /etc/nginx/sites-available/phpmyadmin /etc/nginx/sites-enabled/
```
## 3. Atualizar o arquivo hosts da sua máquina
Para que o seu computador entenda o endereço phpmyadmin.local, adicione ele no seu arquivo de hosts (junto com o outro que você já fez):
```bash
sudo nano /etc/hosts
```
Adicione a seguinte linha ao final do arquivo:
```bash
127.0.0.1 phpmyadmin.local
```
## 4. Testar e reiniciar o Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```
Pronto! Agora você tem um ambiente modular: quando quiser mexer no banco de dados, basta acessar http://phpmyadmin.local. Se no futuro você deletar ou desativar o site agsonhos, o seu gerenciador de banco de dados continuará funcionando intacto.
Você saberia dizer qual a versão do PHP que está utilizando no momento (php -v)? Se precisar, posso te ajudar a confirmar se o caminho do php-fpm.sock está correto para evitar o erro 502 Bad Gateway.










