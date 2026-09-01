# Instalar o servidor web Apache

Para instalar o servidor web Apache no Debian 13 com ambiente KDE, abra o terminal (Ctrl+Alt+T), atualize os pacotes com 
apt update, instale o Apache com apt install apache2, verifique o status com systemctl status apache2, e acesse http://localhost ou o IP da sua máquina no navegador para confirmar a instalação, abrindo a página padrão "It works!". 
1. Atualizar o sistema
```bash
apt update
```
2. Instalar o Apache
```bash
apt install apache2
```
Você pode adicionar -y (ex: apt install apache2 -y) para pular a confirmação. 
3. Verificar o status do serviço
```bash
systemctl status apache2
```
Deve mostrar "active (running)". 
4. Acessar a página padrão

- Localmente: Abra o navegador e digite http://localhost ou http://127.0.0.1.
- Remotamente: Digite o endereço IP da sua máquina no navegador (ex: http://192.168.1.100).