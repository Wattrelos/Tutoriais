# 📘 PostgreSQL no Debian 13 (Trixie)

Para instalar o PostgreSQL no Debian 13 (Trixie), você pode utilizar os repositórios oficiais da própria distribuição (que geralmente fornecem uma versão estável e testada) ou o repositório oficial do projeto PostgreSQL (PGDG), caso precise da versão mais recente (como o PostgreSQL 18). 
Abaixo está o passo a passo utilizando os repositórios oficiais do Debian 13, que é o método mais simples e recomendado para a maioria dos cenários. 
------------------------------
## Passo 1: Atualizar o Sistema
Antes de começar, certifique-se de que os índices de pacotes do seu sistema estejam atualizados. Abra o terminal e execute: 
```bash 
sudo apt update && sudo apt upgrade -y
```

## Passo 2: Instalar o PostgreSQL
Instale o servidor do PostgreSQL junto com o pacote de extensões adicionais (postgresql-contrib): 
```bash 

sudo apt install -y postgresql postgresql-contrib
```
## Passo 3: Verificar o Status do Serviço
O instalador do Debian inicia e ativa automaticamente o banco de dados. Você pode confirmar se ele está rodando com o comando:
```bash 
sudo systemctl status postgresql
```
Procure pela mensagem em verde active (running) na saída do terminal.
------------------------------
## Passo 4: Primeiros Passos e Configuração Básica
Por padrão, o PostgreSQL cria um usuário no sistema operacional chamado postgres. Para interagir com o banco de dados pela primeira vez, mude para esse usuário:

### 1. Acessar o console do banco (psql)
```bash 
sudo -i -u postgres psql
```
Você verá o prompt mudar para postgres=#.
### 2. Definir uma senha para o usuário root do banco (postgres)
Dentro do terminal do PostgreSQL, digite o comando abaixo trocando SuaSenhaSegura por uma senha forte:
```bash 
ALTER USER postgres PASSWORD 'SuaSenhaSegura';
```
### 3. Criar um novo banco de dados e usuário (Boa Prática)
Evite usar o usuário administrador postgres para as suas aplicações. Crie um usuário dedicado e um banco de dados próprio: 
```bash
-- Criar usuário para a aplicação
CREATE ROLE meu_usuario WITH LOGIN PASSWORD 'SenhaDoUsuario123';
-- Criar o banco de dados definindo o novo usuário como dono
CREATE DATABASE meu_banco OWNER meu_usuario;
```
## 4. Sair do PostgreSQL
Para sair do console interativo, digite: 

\q

------------------------------
## Passo 5: Testar o Novo Usuário
Para garantir que tudo ficou configurado corretamente, tente se conectar diretamente ao novo banco usando suas credenciais: [3, 4] 

psql -U meu_usuario -d meu_banco -h localhost

O sistema solicitará a senha definida para o usuário. 

