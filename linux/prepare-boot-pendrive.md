# Preparar pendrive bootável

No Linux, existem excelentes formas de preparar o seu pendrive bootável. O BalenaEtcher é uma ótima opção visual, mas você também pode usar ferramentas nativas ou o comando dd direto no terminal (que dispensa qualquer instalação).
Abaixo estão as três maneiras mais comuns e seguras de fazer isso no Linux:

------------------------------
## Opção 1: Usando o BalenaEtcher (Interface Gráfica)
Como o BalenaEtcher geralmente é distribuído no Linux como um AppImage, você não precisa instalá-lo de forma tradicional.

   1. Acesse o site oficial e baixe a versão do BalenaEtcher para Linux (será um arquivo com extensão .AppImage).
   2. Abra o gerenciador de arquivos, clique com o botão direito sobre o arquivo baixado e vá em Propriedades.
   3. Na aba de permissões, marque a opção "Permitir executar o arquivo como programa" (ou similar).
   4. Dê um duplo clique no arquivo para abrir o BalenaEtcher.
   5. No aplicativo, clique em Flash from file e selecione a ISO do Debian 13.
   6. Clique em Select target e marque o seu pendrive.
   7. Clique em Flash!. O sistema pedirá a sua senha de administrador (sudo) para iniciar o processo.

------------------------------
## Opção 2: Usando o Gravador de Imagem nativo (GNOME / Mint / Ubuntu)
Se você estiver utilizando uma distribuição com interface GNOME (como Ubuntu, Fedora, Debian) ou o Linux Mint, você já tem um gravador nativo muito simples:

   1. Clique com o botão direito sobre o arquivo ISO do Debian 13.
   2. Selecione "Abrir com -> Gravador de imagem de disco" (ou Gravar imagem em disco).
   3. Na janela que se abre, no campo Destino, selecione cuidadosamente o seu pendrive.
   4. Clique em Iniciar restauração (ou Gravar) e digite sua senha.

------------------------------
## Opção 3: Usando o Terminal com o comando dd (Avançado e Rápido)
Este método funciona em qualquer Linux e não requer instalar nada, mas exige atenção para não selecionar o disco errado.

   1. Conecte o pendrive e abra o terminal.
   2. Descubra o nome do seu pendrive rodando o comando:
   
   lsblk
   
   Identifique o pendrive pelo tamanho (ex: sdb, sdc). Não use o número da partição (como sdb1), use a letra do disco inteiro (sdb).
   3. Navegue até a pasta onde está a ISO (geralmente Downloads) e rode o comando substituindo sdb pela letra correta do seu pendrive:
   
   sudo dd if=debian-13-netinst.iso of=/dev/sdb bs=4M status=progress oflag=sync
   
   4. Aguarde o processo chegar a 100%. Quando o terminal liberar a digitação novamente, o pendrive estará pronto.

Qual dessas opções você prefere seguir? Se escolher o método do terminal ou tiver dúvidas sobre como identificar a letra correta do seu pendrive para não apagar o disco errado, posso te ajudar a verificar o comando com segurança.

