------------------------------
## Guia de Migração: Importando Repositório e Issues do GitHub para o Forgejo
Este guia orienta como transferir um repositório do GitHub (incluindo todo o histórico de commits, ramificações, Issues, Pull Requests e Labels) diretamente para o servidor Forgejo local, utilizando a ferramenta de migração nativa via interface web.
------------------------------
## Passo 1: Gerar o Token de Acesso (PAT) no GitHub
Para que o Forgejo possa acessar a API do GitHub e ler seus dados privados (e contornar erros de escopo como o missing required scope 'read:org'), você precisa de um token clássico com as permissões corretas.

   1. Acesse o GitHub e clique na sua foto de perfil (canto superior direito).
   2. Vá em Settings (Configurações) ➡️ Developer Settings (fim da barra lateral esquerda).
   3. Clique em Personal Access Tokens ➡️ Tokens (classic).
   4. Clique no botão Generate new token ➡️ Selecione Generate new token (classic).
   5. Defina um nome para identificação (Ex: Migracao-Forgejo).
   6. Marque obrigatoriamente as seguintes caixas de escopo (scopes):
   * repo (Seleciona automaticamente todas as subcaixas: essencial para clonar códigos privados e públicos).
      * read:org (Essencial para ler dados caso seu repositório pertença a alguma Organização no GitHub).
      * workflow (Opcional: útil se você quiser trazer seus arquivos do GitHub Actions).
   7. Role até o fim da página e clique em Generate token.
   8. Copie o token gerado imediatamente (ele começa com ghp_). Nota: Ele sumirá ao recarregar a página.

------------------------------
## Passo 2: Executar a Migração Automatizada no Forgejo
Todo o processo de importação é feito pela interface do navegador, eliminando a necessidade de usar ferramentas de terminal como o GitHub CLI (gh).

   1. Acesse a interface web do seu Forgejo local no endereço configurado: http://IP_DO_SEU_SERVIDOR:8095.
   2. Faça login com sua conta de administrador/usuário.
   3. No topo direito da tela, clique no ícone de + (sinal de mais) e selecione Nova Migração (New Migration).
   4. Na tela de seleção de provedores, clique no botão GitHub.
   5. Na página de parâmetros de migração, preencha as informações:
   * URL de Clone: Insira a URL HTTPS padrão do seu repositório do GitHub (Ex: https://github.com).
      * Autenticação (Token de Acesso): Cole o token ghp_... que você copiou no Passo 1.
      * Proprietário: Escolha o usuário ou organização do Forgejo que receberá o projeto.
      * Nome do Repositório: Defina o nome do repositório como ele existirá localmente (Ex: agsonhos).
   6. Na seção inferior Itens para Migrar (Migration Items), marque as caixas das informações que deseja importar:
   * Issues (Problemas) 🟩
      * Pull Requests (Solicitações de alteração) 🟩
      * Labels (Rótulos) 🟩
      * Milestones (Marcos) 🟩
      * Releases (Lançamentos) 🟩
   7. Clique no botão Migrar Repositório.

O Forgejo iniciará o download em segundo plano. Dependendo do volume de Issues e do tamanho do histórico, o processo pode levar de alguns segundos a poucos minutos. Ao finalizar, a página do projeto será exibida com todo o conteúdo reconstruído.
------------------------------
## Passo 3: Atualizar o Repositório Local na IDE (Antigravity)
Agora que o seu servidor local Forgejo possui a cópia exata e atualizada do projeto, você deve apontar a pasta do seu código no seu computador local para trabalhar diretamente com o Forgejo, em vez do GitHub.
Abra o terminal integrado do seu projeto (/var/www/html/agsonhos) no Antigravity e reconfigure a URL remota de envio:

# 1. Remover o vínculo antigo chamado 'origin' (evita conflitos e URLs duplicadas)
git remote remove origin
# 2. Adicionar o seu Forgejo local como o novo 'origin' principal
git remote add origin http://IP_DO_SEU_SERVIDOR:8095/seu-usuario-forgejo/agsonhos.git
# 3. Fazer o primeiro envio de segurança para sincronizar a branch principal (main ou master)
git push -u origin main

Dica de Credenciais: Quando o terminal do Antigravity solicitar o Username, digite seu usuário do Forgejo. Quando solicitar a Password, utilize o Token de Escrita do Forgejo (gerado na aba Configurações > Aplicativos do Forgejo), garantindo que você nunca mais precise digitar sua senha mestra no terminal.
------------------------------

Para incluir o número da Issue (que no Git chamamos de ID ou Index), basta adicionar o campo \(.number) no final do script de formatação do jq.
Aqui está o comando completo e atualizado para você rodar no terminal:

curl -s -X 'GET' 'http://localhost:8095/api/v1/repos/JosiasWattrelos/Alpha_Engine_backup/issues' -H 'accept: application/json' | jq -r '.[] | "Nº: #\(.number) | Título: \(.title) | Criado em: \(.created_at)"'

## O que muda no resultado?
Agora, em vez de ver apenas o texto, o jq vai organizar a saída trazendo o identificador numérico na frente, ficando exatamente assim no seu terminal:

Nº: #1 | Título: Corrigir bug na tela de login | Criado em: 2026-08-15T14:30:22Z
Nº: #2 | Título: Atualizar dependências do Apache | Criado em: 2026-08-16T09:15:00Z

Dica extra de fuso horário: O formato Z no final da data significa Zulu Time (UTC). Como você está no Brasil (Horário de Brasília), a hora real em que a Issue foi criada localmente é 3 horas a menos do que o número mostrado nesse relatório da API.
