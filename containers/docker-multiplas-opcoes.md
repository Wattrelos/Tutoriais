# Docker Compose com múltiplos bancos de dados

A melhor prática para alternar entre múltiplos bancos de dados (como MariaDB e PostgreSQL) no Docker Compose depende do seu objetivo, mas criar dois arquivos totalmente separados ou usar arquivos de sobreposição (override) são as abordagens mais limpas.
Abaixo estão as três melhores estratégias para resolver isso, ordenadas da mais recomendada para a mais simples:
------------------------------
## Opção 1: Arquivos de Extensão/Sobreposição (Recomendado para Desenvolvimento)
Em vez de duplicar todo o arquivo, você mantém a estrutura base da sua aplicação em um arquivo principal e isola os bancos de dados em arquivos separados.

   1. docker-compose.yml (Apenas a sua aplicação/serviço base):
   ```yaml
   services:
     web:
       image: minha-app:latest
       ports:
         - "8080:8080"
   ```
   
   2. docker-compose.postgres.yml (Apenas o Postgres):
   ```yaml
   services:
     db:
       image: postgres:18
       environment:
         POSTGRES_PASSWORD: senha_secreta
   ```
   
   3. docker-compose.mariadb.yml (Apenas o MariaDB):
   ```yaml
   services:
     db:
       image: mariadb:latest
       environment:
         MARIADB_ROOT_PASSWORD: senha_secreta
   ```
   
Como rodar:
Você combina os arquivos na linha de comando usando a flag -f:

* Para subir com PostgreSQL: docker compose -f docker-compose.yml -f docker-compose.postgres.yml up -d
* Para subir com MariaDB: docker compose -f docker-compose.yml -f docker-compose.mariadb.yml up -d

------------------------------
## Opção 2: Perfis do Docker Compose (Profiles) – Tudo no mesmo arquivo
Se você prefere manter tudo em um único arquivo, o Docker Compose possui o recurso de profiles. Os serviços só sobem se o perfil deles for explicitamente chamado.

```yaml
services:
  web:
    image: minha-app:latest
    ports:
      - "8080:8080"

  postgres-db:
    image: postgres:18
    profiles: ["postgres"]
    environment:
      POSTGRES_PASSWORD: senha_secreta

  mariadb-db:
    image: mariadb:latest
    profiles: ["mariadb"]
    environment:
      MARIADB_ROOT_PASSWORD: senha_secreta

```
Como rodar:
Por padrão, rodar apenas docker compose up só subirá o serviço web. Para escolher o banco, use a flag --profile:

* Para usar PostgreSQL: docker compose --profile postgres up -d
* Para usar MariaDB: docker compose --profile mariadb up -d

------------------------------
## Opção 3: Dois arquivos completos (Mais simples para iniciantes)
Se a sua aplicação exige muitas variáveis de ambiente que mudam drasticamente dependendo do banco de dados (ex: DB_TYPE=postgres vs DB_TYPE=mysql), criar dois arquivos independentes evita confusão.

* docker-compose.postgres.yml
* docker-compose.mariadb.yml

Como rodar:

* docker compose -f docker-compose.postgres.yml up -d

------------------------------
## Resumo Comparativo

| Estratégia | Vantagem | Desvantagem |
|---|---|---|
| Arquivos de Extensão (-f) | Reutiliza o código da aplicação; evita duplicação. | Comando para rodar fica mais longo. |
| Perfis (profiles) | Tudo centralizado em um único arquivo de configuração. | Pode poluir o arquivo se crescer muito. |
| Dois arquivos isolados | Muito fácil de entender e rodar de forma independente. | Se mudar algo na aplicação, precisa atualizar ambos os arquivos. |
