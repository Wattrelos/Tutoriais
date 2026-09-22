# 📝 Antigravity IDE - Como criar perfis separados

Para garantir que o Antigravity IDE trate cada projeto como um cérebro isolado (com sua própria chave de API, histórico e contexto), siga este passo a passo adaptado para o ambiente dele:

## 1. Criar Perfis Globais de IA no Antigravity
O Antigravity IDE gerencia as extensões e logins através de perfis.

   1. Abra o Antigravity IDE.
   2. Pressione Ctrl + Shift + P para abrir a barra de comandos.
   3. Digite Profiles: Create Profile... e selecione a opção.
   4. Nomeie o perfil de forma intuitiva (ex: Brain - Projeto A).
   5. Escolha a opção Empty Profile (Perfil Vazio). Isso limpa os tokens antigos e impede que a IA puxe dados da sua conta padrão.

## 2. Configurar a API Key / Conta Google Isolada por Perfil
Com o perfil limpo aberto:

   1. Vá até as configurações da IA do Antigravity (geralmente localizadas no painel lateral de chat ou clicando no ícone de engrenagem de configurações).
   2. Se você utiliza o Gemini via API Key, insira a chave específica criada no Google AI Studio para este projeto.
   3. Se você utiliza via Google Auth/Login, faça o login com a conta Google dedicada a este ecossistema.
   4. Dica de ouro: Vá em Settings (Ctrl + ,), digite Gemini e certifique-se de salvar as credenciais no escopo de User (que agora está amarrado apenas a este Perfil ativo).

## 3. Limpar e Fixar o Contexto do Workspace (.vscode / .antigravity)
IAs locais ou embutidas criam índices (vetores dos seus arquivos) para entender o código. Para garantir que o cérebro leia apenas o necessário:

   1. Abra a pasta do seu projeto.
   2. Certifique-se de que o seu arquivo .gitignore (ou o mecanismo de ignore do Antigravity) esteja ignorando pastas pesadas de dependências (vendor/, node_modules/). Se a IA tentar ler essas pastas, ela vai misturar o contexto com milhares de linhas de bibliotecas externas e "poluir" o cérebro.
   3. Use o comando Ctrl + Shift + P > Profiles: Switch Profile e selecione o Brain - Projeto A para travar essa pasta a esse perfil de IA específico.

## 4. Criando Atalhos no Debian 13 para Troca Instantânea
Para abrir o Antigravity direto no cérebro correto via terminal no seu Debian:

   1. Abra o terminal e edite seu arquivo de inicialização: nano ~/.bashrc
   2. Adicione os aliases apontando para os perfis do executável do seu editor (se o comando dele for antigravity ou similar):
   
   alias brain-a="antigravity --profile 'Brain - Projeto A'"
   alias brain-b="antigravity --profile 'Brain - Projeto B'"
   
   3. Salve com Ctrl+O, Enter, Ctrl+X e recarregue com source ~/.bashrc.


-----------------------------------
# 📝 VS Code / Antigravity - Como separar contas do Gemini (perfis)

> Os Perfis (Profiles) do VS Code são a solução ideal para separar contextos. Eles isolam completamente as extensões ativas, históricos de chats, contas Google e até preferências visuais. 
Nota sobre o "Antigravity": Se você estiver se referindo ao fork focado em IA baseado no VS Code (ou editores similares como o Cursor), o mecanismo de perfis funciona exatamente da mesma maneira, pois eles herdam a arquitetura base do VS Code. 
Abaixo está o tutorial passo a passo para criar e alternar entre ambientes isolados:
------------------------------
## Passo 1: Criar um Novo Perfil Isolado

   1. Abra o seu VS Code.
   2. Clique no ícone de Engrenagem (⚙️) no canto inferior esquerdo e selecione Profiles > Create Profile... (Ou use o atalho Ctrl + Shift + P e digite Profiles: Create Profile).
   3. No campo que aparecer no topo, dê um nome claro ao perfil (Ex: Projeto Corporativo ou Projeto Pessoal).
   4. Em Copy from, mude de "Default" para Empty Profile (Perfil Vazio).
   * Por que vazio? Isso garante que nenhuma extensão de IA ou token de login antigo seja herdado por padrão, criando um ambiente 100% limpo.
   5. Clique em Create. Uma nova janela do VS Code totalmente "limpa" se abrirá. 

## Passo 2: Conectar o Gemini Específico Deste Perfil
Neste novo perfil vazio, as suas extensões e logins globais não existem. [1, 2] 

   1. Vá até a aba de Extensões (Ctrl + Shift + X).
   2. Instale a extensão de IA que você utiliza (ex: Gemini Code Assist, Google Cloud Code, etc.).
   3. Clique no ícone da extensão ou na barra de status inferior para fazer login. Conecte apenas a conta Google correspondente a este projeto específico.

O token gerado ficará preso exclusivamente ao armazenamento de dados deste perfil. 
## Passo 3: Vincular o Perfil Diretamente à Pasta do Projeto
Para não precisar trocar de perfil manualmente toda vez, você pode forçar uma pasta a sempre abrir com o perfil correto: 

   1. Com o perfil correto ativo, abra a pasta do seu projeto (File > Open Folder).
   2. Abra a Paleta de Comandos (Ctrl + Shift + P) e digite: Profiles: Switch Profile.
   3. Selecione o perfil desejado. O VS Code criará uma associação interna. Da próxima vez que você abrir essa pasta específica (mesmo usando o comando code . no terminal do Debian), ele abrirá automaticamente usando aquele perfil e aquela conta do Gemini. 

## Passo 4 (Opcional): Atalhos Rápidos no Terminal do Debian
Se você gosta de abrir projetos direto pelo terminal do Debian 13, pode criar apelidos (aliases) para abrir o VS Code já no perfil correto.

   1. Abra o arquivo de configuração do seu terminal (ex: nano ~/.bashrc ou nano ~/.zshrc).
   2. Adicione linhas como estas no final do arquivo:
   
   alias code-work="code --profile 'Projeto Corporativo'"
   alias code-personal="code --profile 'Projeto Pessoal'"
   
   3. Salve o arquivo (Ctrl + O, depois Enter e Ctrl + X para sair) e recarregue o terminal com source ~/.bashrc.
   4. Agora, basta rodar code-work . dentro de uma pasta para abrir o editor no perfil de trabalho instantaneamente!

------------------------------
## Como saber qual perfil está ativo?
O VS Code exibe as duas primeiras letras do nome do perfil ativo diretamente no ícone de engrenagem (⚙️) no canto inferior esquerdo. Se você der nomes com cores ou temas diferentes para cada perfil (ex: Tema Escuro para trabalho, Tema Claro para projetos pessoais), a distinção visual fica imediata e evita qualquer erro humano.
Ficou claro o fluxo de criação? Se você estiver usando um fork específico como o Cursor, Anysphere ou outro editor de IA específico e notar alguma diferença nos menus, me avise para ajustarmos o comando exato!

