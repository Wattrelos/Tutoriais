# Instalação do Composer no Debian 13

> Guia passo a passo para instalar o Composer globalmente no Debian 13.

## 1. Atualizar o sistema e instalar dependências

Atualize a lista de pacotes e instale o PHP CLI junto com as ferramentas e extensões essenciais:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install php-cli php-curl php-mbstring php-xml git unzip curl -y
```

## 2. Baixar e verificar o instalador do Composer
Para garantir a segurança, baixe o script de instalação oficial e valide a assinatura digital diretamente com a chave mais recente do [Composer](https://getcomposer.org/download/):

```bash
# 1. Baixar o instalador
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
```
```bash
# 2. Obter a assinatura oficial mais recente
HASH="$(curl -sS https://composer.github.io/installer.sig)"
```
```bash
# 3. Verificar integridade
php -r "if (hash_file('sha384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```
> Atenção: Só continue se o retorno for Installer verified.

## 3. Instalar o Composer Globalmente
Execute o instalador configurando-o para disponibilizar o comando composer para todo o sistema dentro do diretório de binários do Debian 13: 

```bash
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

## 4. Limpar os arquivos temporários
Remova o script de configuração que não é mais necessário:
```bash
php -r "unlink('composer-setup.php');"
```

## 5. Validar a instalação
Por fim, verifique se o comando está respondendo corretamente e exiba a versão instalada:
```bash
composer --version
```

Se encontrar qualquer erro durante o processo, me informe:

* Qual é a versão exata do PHP instalada no seu contêiner?
* O terminal do Antigravity está rodando em um ambiente local ou remoto?

Dessa forma, posso diagnosticar problemas de rede ou dependências ausentes de maneira mais precisa.

