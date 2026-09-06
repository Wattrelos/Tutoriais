Para instalar o Redis e integrá-lo ao PHP 8.4 com Apache2 no Debian 13 (Trixie), siga o passo a passo abaixo executando os comandos no seu terminal:
## 1. Instalar o servidor do Redis
Atualize os repositórios do sistema e instale o pacote oficial: 

sudo apt update
sudo apt install -y redis-server

Ative o serviço para iniciar automaticamente com o sistema e execute-o agora: 

sudo systemctl enable --now redis-server

(Opcional) Teste se o servidor está respondendo digitando redis-cli ping. Ele deve retornar PONG. [2, 3] 
## 2. Instalar a extensão do Redis para o PHP 8.4
Como você está utilizando o PHP 8.4 (geralmente instalado via repositório de Ondřej Surý no Debian), instale o módulo correspondente e a ferramenta de gerenciamento: 

sudo apt install -y php8.4-redis

Ative a extensão especificamente para a sua versão do PHP: 

sudo phpenmod -v 8.4 redis

## 3. Reiniciar o Apache2
Para que o Apache reconheça a nova extensão instalada no ecossistema do PHP, reinicie o servidor web: 

sudo systemctl restart apache2

## 4. Verificar se a instalação funcionou
Para ter certeza de que o Apache e o PHP carregaram o módulo do Redis com sucesso, execute o comando abaixo no terminal: 

php -m | grep redis

Se a palavra redis for exibida na tela, a instalação foi concluída perfeitamente.