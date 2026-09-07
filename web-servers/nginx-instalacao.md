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

# 2. Configuração do site no Nginx

## Migração da configuração do Apache para o Nginx
Se você está migrando do Apache2, lembre-se que o Nginx não lê arquivos .htaccess. Toda regra de reescrita que antes ficava oculta no .htaccess agora precisa ser tratada diretamente pelo Nginx através da diretiva try_files $uri $uri/ /index.php?$query_string;.

```nginx
server {
    listen 80;
    server_name meusite.local www.meusite.local localhost;

    # Ponto de entrada do projeto (pasta pública)
    root /var/www/html/meusite/public_html;
    
    # Arquivos padrão
    index index.php index.html index.htm;

    # Limite para upload de imagens/arquivos
    client_max_body_size 64M;

    # Logs
    access_log /var/log/nginx/meusite-access.log;
    error_log /var/log/nginx/meusite-error.log;

    # Bloqueia acesso direto à rota /admin (pois o admin usa pasta ofuscada)
    location ^~ /admin {
        return 403;
    }

    # Roteamento do Slim Framework (envia o que não for estático para o index.php)
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Processamento PHP via PHP-FPM 8.4
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    }

    # Bloqueio de arquivos e diretórios ocultos (.env, .git, etc.)
    location ~ /\. {
        deny all;
    }

    # Cache de arquivos estáticos (CSS, JS, imagens, fontes)
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|webp|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
        try_files $uri =404;
    }
}

```
## O que mudou na conversão?

* DocumentRoot virou root: Aponta exatamente para o mesmo diretório público.
* AllowOverride All virou try_files: O Nginx não lê arquivos .htaccess. Toda regra de reescrita que antes ficava oculta no .htaccess agora precisa ser tratada diretamente pelo Nginx através da diretiva try_files $uri $uri/ /index.php?$query_string;.
* Options Indexes (Listagem de arquivos): Por padrão, o Nginx desativa a listagem de arquivos por segurança (retorna erro 403 se não houver um index.html ou index.php). Se você realmente precisar listar os arquivos no navegador como o Apache fazia, adicione a linha autoindex on; dentro do bloco location /.

## 3. Ativar o site
Após criar o arquivo dentro do diretório sites-available, você precisa ativar o site criando um link simbólico para a pasta sites-enabled e reiniciar o Nginx.
Siga estes 3 passos sequenciais no terminal:

## 1. Ativar o site (Criar o link simbólico)
O Nginx só lê os sites que estão na pasta sites-enabled. Execute o comando abaixo substituindo meusite pelo nome exato do arquivo que você criou:
```bash
sudo ln -s /etc/nginx/sites-available/meusite /etc/nginx/sites-enabled/
```

## 2. Testar a configuração
Antes de reiniciar o servidor, certifique-se de que não há nenhum erro de sintaxe (como a falta de um ponto e vírgula ;):
```bash
sudo nginx -t
```
Se tudo estiver correto, você verá a mensagem: syntax is ok e test is successful.
## 3. Recarregar o Nginx
Se o teste passou, aplique as novas configurações sem derrubar o servidor:
```bash
sudo systemctl reload nginx
```

------------------------------
# Passo Extra (Importante para domínios locais .local)
Como você está usando o domínio meusite.local, o seu computador precisa saber para onde enviar esse endereço. Se você for testar no mesmo servidor (ou se estiver usando uma máquina virtual/WSL na mesma máquina), adicione o domínio ao arquivo hosts do seu sistema operacional:

* No Linux / macOS: Edite o arquivo /etc/hosts (sudo nano /etc/hosts) e adicione a linha:

127.0.0.1 meusite.local www.meusite.local

* No Windows: Edite o arquivo C:\Windows\System32\drivers\etc\hosts como Administrador e adicione a mesma linha acima.

Ao acessar http://meusite.local no navegador, o site carregou corretamente ou você se deparou com algum erro (como 403 Forbidden ou 502 Bad Gateway)? Edições extras podem ser necessárias dependendo do tipo do projeto.


Sim, há um ajuste importante sobre como você está acessando o site. O comportamento que você viu é perfeitamente normal:
Ao acessar http://localhost/, o Nginx entrega a página padrão dele ("Welcome to nginx!") porque o arquivo de configuração original chamado default está capturando essa requisição.
Para ver o seu projeto, você deve fazer o seguinte:
## 1. Acesse pelo domínio correto
No seu navegador, você deve digitar http://meusite.local (ou http://meusite.local) em vez de localhost. O Nginx usa a linha server_name meusite.local; para saber qual pasta abrir. Se você digitar localhost, ele não reconhece o site.
## 2. Certifique-se de que configurou o arquivo hosts
Se você digitar http://meusite.local e o navegador der um erro de "Não foi possível encontrar este site", significa que o seu sistema operacional ainda não sabe que esse nome aponta para o seu próprio computador.
Você precisa adicionar a linha abaixo no arquivo hosts da sua máquina (veja como no passo anterior):

127.0.0.1 meusite.local www.meusite.local

------------------------------
## Dica Bônus: Desativar a página padrão do Nginx (Opcional)
Se você quiser que o seu site apareça mesmo quando você digitar http://localhost, você precisa desativar o site padrão do Nginx e dizer para o seu arquivo aceitar o localhost.

   1. Remova o link do site padrão:
   
   sudo rm /etc/nginx/sites-enabled/default
   
   2. Adicione o localhost no seu arquivo (Abra /etc/nginx/sites-available/meusite e mude a linha server_name):
   
   server_name meusite.local www.meusite.local localhost;
   
   3. Recarregue o Nginx:
   
   sudo systemctl reload nginx
   
   
Ao digitar http://meusite.local diretamente na barra de endereços, o seu site abriu corretamente ou apareceu alguma página de erro?



