# Baixando imagens no Docker Desktop

No Docker Desktop, você pode buscar e baixar (fazer o pull) imagens prontas de duas formas: diretamente pela interface gráfica (mais visual) ou usando a barra de pesquisa rápida.
Aqui está o passo a passo de como fazer isso:
## Método 1: Pela aba "Images" (Recomendado)

   1. Abra o Docker Desktop na sua tela.
   2. No menu lateral esquerdo, clique em Images.
   3. No topo da tela, haverá um campo de busca ou um botão escrito "Search for images on Docker Hub" (Buscar imagens no Docker Hub).
   4. Digite o nome da imagem que você procura (por exemplo: ubuntu, nginx, mysql ou python).
   5. A interface mostrará os resultados vindos do Docker Hub. As que possuem o selo "Official Image" são as mais seguras e recomendadas.
   6. Clique no botão Pull ao lado da imagem desejada. O Docker Desktop vai baixar a imagem para o seu computador.

------------------------------
## Método 2: Pela barra de busca global (Mais rápido)

   1. No topo da janela do Docker Desktop, clique na barra de pesquisa principal (ou pressione Ctrl + K).
   2. Digite o nome da tecnologia que você quer (ex: postgres).
   3. O painel vai exibir abas com Containers, Images e Volumes. Clique em Images.
   4. Clique no botão Pull para apenas baixar a imagem, ou em Run se você já quiser baixá-la e colocá-la para rodar imediatamente configurando as portas e variáveis de ambiente na tela que vai abrir.

------------------------------
## 💡 Dica de ouro: Como escolher a versão (Tag)
Por padrão, o Docker sempre baixa a versão mais recente da imagem (chamada de latest). Se você precisar de uma versão específica (por exemplo, o Node.js na versão 18):

* Ao pesquisar e clicar na imagem dentro do Docker Desktop, procure pelo menu suspenso ou aba chamada Tags.
* Ali você verá uma lista de todas as versões disponíveis. Basta selecionar a versão que você quer e clicar em Pull.

Você está procurando por alguma imagem específica para o seu projeto (como um banco de dados ou linguagem de programação)? Se quiser, posso te dizer exatamente qual é o nome oficial dela para facilitar sua busca!

