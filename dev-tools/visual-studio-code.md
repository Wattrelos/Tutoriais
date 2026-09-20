
------------------------------
## 🪟 Como Instalar no Windows
Instalar no Windows é muito simples e segue o padrão visual clássico do sistema.

   1. Baixe o Instalador: Acesse o site oficial do [Visual Studio Code](https://code.visualstudio.com/) e clique no botão Download for Windows. O download de um arquivo .exe começará automaticamente. 
   2. Execute o Arquivo: Após o término do download, dê um duplo clique no arquivo baixado (geralmente chamado de VSCodeUserSetup-{versão}.exe). 
   3. Aceite os Termos: Leia e mude a opção para "Eu aceito o acordo", depois clique em Avançar. 
   4. Tarefas Adicionais (Importante): Avance pelas telas mantendo o local padrão. Na tela "Selecionar Tarefas Adicionais", certifique-se de marcar a opção "Adicionar ao PATH". Também é altamente recomendável marcar as opções para "Adicionar a ação 'Abrir com Code' ao menu de contexto do Windows Explorer". Clique em Avançar. 
   5. Conclua a Instalação: Clique em Instalar. Quando a barra carregar, clique em Concluir com a opção "Iniciar o Visual Studio Code" marcada.  

------------------------------
## 🐧 Como Instalar no Linux Debian 13
No Debian 13, você tem duas formas excelentes de realizar a instalação. Escolha a que preferir:
## Método A: Pela Interface Gráfica (Mais fácil)

   1. Acesse o site oficial do Visual Studio Code pelo seu navegador. 
   2. Clique para baixar o pacote .deb (específico para Ubuntu/Debian). 
   3. Assim que o download terminar, abra o gerenciador de arquivos, vá até a pasta Downloads e abra o arquivo baixado usando a sua Central de Softwares padrão do Debian (como o GNOME Software ou Discover). 
   4. Clique em Instalar, digite sua senha de administrador (root) e pronto! O próprio instalador cuidará de adicionar o repositório da Microsoft para atualizações futuras. 

## Método B: Pelo Terminal (Garante atualizações automáticas via APT)
Se prefere usar a linha de comando para gerenciar seus pacotes e configurar o repositório oficial da Microsoft:

   1. Abra o terminal e atualize o índice de pacotes:
   
```bash
   sudo apt update && sudo apt upgrade -y
```   

   2. Instale as dependências necessárias para transferências seguras:
   
```bash
   sudo apt install -y software-properties-common apt-transport-https curl GPG
```
   
   3. Importe a chave GPG oficial da Microsoft para validar o download do aplicativo:
   
```bash
   curl -sSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /usr/share/keyrings/vscode.gpg > /dev/null
```
   
   4. Adicione o repositório estável do VS Code às fontes do seu sistema:
   
```bash
   echo "deb [arch=amd64 signed-by=/usr/share/keyrings/vscode.gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
   
   5. Atualize a lista de pacotes e faça a instalação do editor:

```bash
   sudo apt update
   sudo apt install code -y
```
