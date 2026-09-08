# Instalação do Debian 13

Instalar o Debian 13 (Trixie) utilizando um pendrive bootável e realizando o download dos pacotes diretamente da internet através da imagem oficial Netinst (Network Installer).
Esse método é o mais recomendado, pois gera uma mídia leve (cerca de 750 MB) e garante que o seu sistema seja instalado com as versões mais recentes dos pacotes. 
------------------------------
## Passo 1: Baixar a imagem Netinst

   1. Acesse a página oficial de [Instalação pela Rede do Debian](https://www.debian.org/distrib/netinst.pt.html).
   2. Baixe a imagem ISO correspondente à arquitetura do seu processador (na maioria dos computadores modernos, escolha amd64 para sistemas de 64 bits). [1, 3] 

## Passo 2: Criar o Pendrive Bootável
Para gravar a ISO no pendrive de forma correta, você precisará de uma ferramenta de gravação:

* 
* BalenaEtcher: Excelente opção visual e multiplataforma (funciona em Windows, macOS e Linux).
* Rufus: Ótima ferramenta caso você esteja criando o pendrive a partir do Windows. Se usar o Rufus, selecione o pendrive, a ISO do Debian e prefira gravar no modo Imagem DD se o modo ISO apresentar problemas de detecção. [4] 
* 

⚠️ Atenção: Esse processo apagará permanentemente todos os arquivos que estiverem dentro do pendrive. [3] 

## Passo 3: Inicializar pelo Pendrive (Boot)

   1. Conecte o pendrive no computador que receberá o Debian 13.
   2. Ligue o computador pressionando a tecla de menu de boot da sua placa-mãe (geralmente F12, F11, F8 ou Esc).
   3. Selecione o seu pendrive na lista de dispositivos de inicialização. Se o seu computador for recente, prefira a inicialização em modo UEFI. [5] 

## Passo 4: O Processo de Instalação (Via Rede)
Ao iniciar pelo pendrive, a tela do instalador do Debian será exibida: [2, 6] 

   1. Escolha o instalador: Selecione Graphical install (Instalação Gráfica) para um processo mais intuitivo guiado pelo mouse. [2, 6] 
   2. Idioma e Teclado: Selecione Português do Brasil e o layout do seu teclado (geralmente Português Brasileiro ABNT2). [6] 
   3. Configuração de Rede: O instalador tentará configurar a internet automaticamente via cabo (DHCP) ou exibirá as redes Wi-Fi disponíveis. Uma conexão ativa é obrigatória neste ponto, pois os arquivos do sistema serão baixados agora. [1] 
   4. Contas e Senhas: Defina o nome da máquina, a senha do usuário administrador (root) e crie a sua conta de usuário comum com uma nova senha. [6] 
   5. Particionamento do Disco:
   * Para iniciantes, a opção Assistido - usar o disco inteiro configurará tudo automaticamente.
      * Para quem faz dual-boot com o Windows ou quer criar partições separadas (como /home), escolha a opção Manual. [6, 7] 
   6. Configuração do Gerenciador de Pacotes: O instalador perguntará se deseja utilizar um "espelho de rede" (Network Mirror). Responda Sim. Escolha o país (Brasil) e selecione um servidor da lista (como deb.debian.org ou o de alguma universidade de sua preferência) para garantir downloads rápidos. [6] 
   7. Seleção de Software (Desktop): Graças à instalação via rede, nesta tela você poderá marcar qual interface gráfica deseja instalar por padrão (como GNOME, XFCE, KDE Plasma, entre outras). Deixe marcado também Utilitários padrão do sistema. [6] 
   8. Finalização: O instalador vai baixar os pacotes da interface escolhida e os componentes do sistema operacional. Por fim, o carregador de inicialização GRUB será instalado no disco rígido. [6] 

Ao terminar, remova o pendrive quando solicitado e reinicie o computador para entrar no seu novo Debian 13!

