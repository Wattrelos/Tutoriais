# Instalação do Composer

## Passo 1: Atualizar o sistema e instalar dependências
- O Composer precisa do PHP via linha de comando (php-cli) e de utilitários de descompactação. Abra o terminal e execute: 
``` bash
sudo apt update
sudo apt install php-cli unzip curl php-mbstring git -y
```

## Passo 2: Baixar o instalador oficial
Baixe o script de instalação oficial diretamente do servidor do [Composer](https://getcomposer.org/): 
``` bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === 'c8b085408188070d5f52bcfe4ecfbee5f727afa458b2573b8eaaf77b3419b0bf2768dc67c86944da1544f06fa544fd47') { echo 'Installer verified'.PHP_EOL; } else { echo 'Installer corrupt'.PHP_EOL; unlink('composer-setup.php'); exit(1); }"
php composer-setup.php
php -r "unlink('composer-setup.php');"
```

## Passo 3: Instalar o Composer Globalmente
- Para que você consiga usar apenas o comando composer em qualquer diretório, instale-o apontando para a pasta de binários públicos do Debian (/usr/local/bin):
``` bash
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

## Passo 4: Limpar os arquivos temporários
- Remova o script instalador que não será mais necessário:
``` bash
rm composer-setup.php
```

## Passo 5: Verificar a instalação
- Teste se o sistema reconhece o comando global executando:

``` bash
composer --version
```
- O terminal deve retornar algo como: Composer version 2.10.2 (ou superior)...

# Ajuste do Composer

## Opção 1: Usar o Composer localmente (Imediato), por exemplo, em ambiente Windows que não tem permissão de administrador

php composer.phar --version

------------------------------
## Opção 2: Tornar o Composer global (Recomendado)
Para poder usar apenas o comando composer de qualquer pasta do seu terminal, mova o arquivo para o diretório de executáveis do sistema e dê permissão de execução:

   1. Mova o arquivo para a pasta global:
   
   sudo mv composer.phar /usr/local/bin/composer
   
   2. Dê permissão de execução:
   
   sudo chmod +x /usr/local/bin/composer
   
   3. Teste novamente:
   
   composer --version

