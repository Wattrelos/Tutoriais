# Instalação do Servidor SSH no Debian 13


# 1. Instalar o servidor SSH
Execute o comando abaixo como root ou com sudo para atualizar a lista de pacotes e instalar o OpenSSH Server:

sudo apt update && sudo apt install openssh-server -y

O serviço será iniciado automaticamente após a instalação. Você pode confirmar se ele está ativo com o comando:

sudo systemctl status ssh

------------------------------
## 2. Descobrir o IP do Debian 13
Você precisará do endereço IP do Debian para se conectar a ele. Descubra executando:

ip a

Procure pela sua interface de rede ativa (ex: eth0 ou enp3s0) e anote o número que aparece após inet (ex: 192.168.1.50).
------------------------------
## 3. Acessar de outra máquina
Vá para o outro computador (que deve estar na mesma rede local) e use um cliente SSH para se conectar.
## A partir de sistemas Linux ou macOS (via Terminal):
Substitua usuario pelo seu nome de usuário no Debian e IP_DO_DEBIAN pelo IP que você anotou:

ssh usuario@IP_DO_DEBIAN

Na primeira conexão, o sistema perguntará se você confia na chave do servidor. Digite yes e depois insira a sua senha.
## A partir do Windows:

* Via Prompt de Comando / PowerShell: Você pode usar o mesmo comando ssh usuario@IP_DO_DEBIAN.
* Via interface gráfica: Baixe e use o programa PuTTY, insira o IP no campo "Host Name" e clique em "Open".

------------------------------
## ⚠️ Notas importantes sobre segurança

* Acesso como Root desativado: Por padrão, o Debian bloqueia o login direto do usuário root via SSH por motivos de segurança. Acesse sempre com seu usuário comum e, se precisar de privilégios, use sudo ou su -.
* Firewall: Se você tiver o UFW ativado no Debian, lembre-se de liberar a porta do SSH executando sudo ufw allow ssh.


