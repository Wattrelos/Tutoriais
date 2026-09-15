# Ambientes isolados para cada aluno
_Essa é uma das maiores vantagens de usar o Docker em um ambiente multiusuário. Cada aluno pode ter o seu banco de dados completamente isolado, protegido e inacessível para os colegas das outras contas._

O Debian, quando você cria contas de usuário diferentes (ex: aluno_manha, aluno_noite ou contas nominais), cada uma ganha a sua própria pasta pessoal protegida em /home/usuario/.
Para garantir que o banco de dados e os projetos de um aluno fiquem 100% isolados dos outros usando o Docker, só precisa seguir duas regras estruturais:

## Como estruturar o isolamento dos alunos
### 1. Salvar os dados dentro da pasta /home de cada usuário
Por padrão, se você subir um banco de dados no Docker sem especificar onde salvar, ele cria um "Volume" global do sistema. Para evitar isso, os arquivos do banco de dados (como o MySQL/PostgreSQL) e do site devem ser mapeados para pastas dentro do /home do respectivo aluno.
Como o Debian bloqueia por padrão que um usuário comum acesse a pasta /home de outro, o "aluno B" jamais conseguirá ver, alterar ou deletar os arquivos do banco de dados do "aluno A".


### 2. Usar portas diferentes nos containers de cada usuário (Se usarem a máquina ao mesmo tempo)

* Se os alunos usam o computador em horários diferentes: Eles podem usar as mesmas portas padrão (ex: o aluno da manhã desliga o container ao sair, e o da noite liga o dele na mesma porta 80 e 3306).
* Se os alunos usam conexões simultâneas (ex: SSH remoto no mesmo servidor Debian): Cada usuário terá que subir o seu Docker mapeando uma porta externa diferente (ex: Aluno 1 usa a porta 8001, Aluno 2 usa a 8002), embora internamente o container continue rodando na porta 80.

------------------------------
## Exemplo Prático: O arquivo docker-compose.yml do Aluno
Na pasta de cada aluno (ex: /home/aluno_manha/projeto/), você deixa um único arquivo chamado docker-compose.yml. Quando o aluno digita docker compose up -d, o ambiente dele sobe isolado assim:

version: '3.8'
services:
  # Servidor Web (Nginx + PHP)
  web:
    image: php:8.2-fpm # Exemplo simples, ou uma imagem com nginx
    ports:
      - "8080:80" # Acessível via http://localhost:8080
    volumes:
      - ./html:/var/www/html # Os arquivos do site ficam na pasta do aluno

  # Banco de Dados Isolado
  banco:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 'senha_segura_do_aluno'
      MYSQL_DATABASE: 'meu_projeto'
    ports:
      - "3306:3306"
    volumes:
      # CRUCIAL: Salva o banco de dados dentro da pasta oculta do próprio aluno
      - ./dados_banco:/var/lib/mysql 

## O resultado prático no Laboratório:

   1. O Aluno A senta na máquina, entra no usuário dele, abre o terminal na sua pasta e digita docker compose up. Ele mexe no site dele e altera o banco de dados dele. Ao fim da aula, ele digita docker compose down.
   2. O Aluno B senta na mesma máquina, faz login na conta dele. A pasta do Aluno A está trancada para ele. Ele entra na sua própria pasta, digita docker compose up e tem um banco de dados zerado ou mantido exatamente como ELE deixou na aula passada, sem qualquer risco de ter seus dados apagados pelo colega.


# Autenticação centralizada dos alunos

_Essa é a arquitetura ideal e padrão da maioria dos laboratórios universitários do mundo._

Ao integrar o seu servidor Debian 13 a um servidor de autenticação centralizado (como Active Directory, FreeIPA ou LDAP), os dados de login dos alunos não ficam salvos na máquina local, mas sim na rede da faculdade.
O Linux possui um recurso nativo excelente chamado PAM (Pluggable Authentication Modules) que faz exatamente o que você descreveu: quando o aluno digita a sua matrícula e senha pela primeira vez em qualquer computador do laboratório, o sistema valida os dados no servidor de rede e, se estiverem corretos, cria a pasta /home/nome_do_aluno automaticamente na hora, gerando um ambiente limpo e isolado para ele.

------------------------------
## Como essa estrutura funciona na prática?

   1. A Autenticação Centralizada: Você configura o Debian 13 usando ferramentas como o sssd (System Security Services Daemon) conectado ao servidor de contas da faculdade.
   2. A Criação Automática do Home: Ativa-se um módulo do PAM chamado pam_mkhomedir.so. Ele copia um modelo de arquivos (geralmente vindo de /etc/skel/) para a nova pasta do aluno.
   3. O Template do Docker (/etc/skel): É aqui que a mágica acontece. Tudo o que você colocar dentro da pasta /etc/skel do servidor local será copiado para a pasta do aluno no primeiro login. Se você deixar lá uma pasta com o arquivo docker-compose.yml pré-configurado, todo aluno que logar já terá o seu ambiente Docker pronto para rodar instantaneamente.

------------------------------
## O Fluxo Completo do Aluno no Laboratório
Para que você visualize o cenário final, veja como seria a experiência do estudante:

* Passo 1: O aluno senta em qualquer computador do laboratório e digita seu usuário (ex: 202610123) e sua senha da faculdade.
* Passo 2: O Debian valida com o servidor central, faz o login e cria a pasta /home/202610123/.
* Passo 3: O aluno abre o terminal e a pasta do projeto já está lá (copiada do /etc/skel). Ele digita docker compose up -d.
* Passo 4: O Docker inicia o Nginx, PHP e o Banco de Dados MySQL dele. Ele estuda, altera os arquivos e salva o banco de dados.
* Passo 5: Na próxima aula, mesmo que ele sente em outro computador, se o laboratório utilizar NFS (pastas compartilhadas em rede), a /home dele será carregada na máquina nova com todos os arquivos e bancos de dados exatamente como ele deixou. Se não usarem rede, os dados ficam salvos de forma segura no HD daquela máquina específica para quando ele voltar.

## O Comando Chave no Debian 13
Para fazer o Debian criar a pasta Home no primeiro login automaticamente, o comando administrado no terminal é:

sudo pam-auth-update

_(Basta marcar a opção "Create home directory on login" na interface que se abre no terminal.)_
