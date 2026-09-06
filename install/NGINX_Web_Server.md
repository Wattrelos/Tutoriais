# Instalação do NGINX no Debian 13 (Trixie)
_Para instalar o servidor web NGINX no Debian 13 (Trixie), você pode utilizar os repositórios oficiais da distribuição, o que torna o processo direto e seguro._

## 1. Atualizar o índice de pacotes
Antes de qualquer instalação, certifique-se de que a lista de pacotes do seu sistema está atualizada: 

```bash
sudo apt update
```

## 2. Instalar o NGINX
Execute o comando abaixo para instalar o servidor web: 

```bash
sudo apt install nginx -y
```

## 3. Configurar o Firewall (UFW)
Se você utiliza o UFW (Uncomplicated Firewall), é necessário liberar as portas de conexões web (HTTP na porta 80 e HTTPS na porta 443). O NGINX registra perfis automáticos no UFW durante a instalação: 
Para liberar tanto o tráfego HTTP quanto o HTTPS, use o perfil 'Nginx Full': 
```bash
sudo ufw allow 'Nginx Full'
```

## 4. Gerenciar o serviço do NGINX
O instalador do Debian ativa e inicia o NGINX automaticamente. Você pode gerenciar o comportamento do serviço usando o systemd: 

* Verificar o status atual do servidor:
```bash
sudo systemctl status nginx
```

* Garantir que o NGINX inicie junto com o sistema (Boot):
```bash
sudo systemctl enable nginx
```

* Recarregar as configurações sem derrubar conexões ativas (útil após alterar arquivos de sites):

sudo systemctl reload nginx


## 5. Validar a Instalação
Para confirmar se tudo está funcionando, abra o seu navegador de preferência e digite o endereço IP do seu servidor ou http://localhost (se estiver testando em ambiente local). A página padrão "Welcome to nginx" do Debian deverá ser exibida. 
------------------------------
## 📂 Diretórios importantes que você deve conhecer:

* /etc/nginx/nginx.conf: O arquivo de configuração principal do NGINX.
* /etc/nginx/sites-available/: Pasta onde você cria os arquivos de configuração para cada site individual (conhecidos como Server Blocks).
* /etc/nginx/sites-enabled/: Pasta que armazena links simbólicos apontando para os sites ativos.
* /var/www/html/: O diretório raiz padrão onde os arquivos do seu site (HTML/CSS) ficam armazenados. 

