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








