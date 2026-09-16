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

# Gerar o arquivo composer.json

Para gerar o arquivo composer.json na raiz da sua árvore de pastas, você deve utilizar o comando interativo do próprio Composer diretamente no terminal. Esse comando criará a estrutura básica do arquivo de forma automática.
Siga os passos abaixo:
## 1. Navegue até a pasta raiz do seu projeto
Abra o terminal e entre na pasta principal (a raiz que contém toda a sua árvore de diretórios vazios):

```bash
cd /caminho/para/sua/pasta-principal
```

## 2. Inicie a configuração do Composer
Execute o comando de inicialização:

```bash
composer init
```

## 3. Responda às perguntas no terminal
O Composer iniciará um assistente interativo fazendo algumas perguntas. Se você não souber o que colocar em alguma delas, basta apertar Enter para aceitar o valor padrão (ou deixar em branco) e ajustar depois.
As principais perguntas são:

* Package Name: O nome do seu projeto no formato seu-usuario/nome-do-projeto (ex: dev/minha-estrutura).
* Description: Uma breve descrição do que o projeto faz.
* Author: Seu nome e e-mail (geralmente ele puxa as configurações globais do Git).
* Minimum Stability: A estabilidade mínima dos pacotes (pode deixar em branco ou digitar stable).
* Package Type: O tipo de projeto. Para aplicações comuns, digite project.
* License: A licença do projeto (ex: MIT ou deixe em branco).

## 4. Definir dependências (Opcional neste momento)
O assistente perguntará se você deseja procurar e instalar dependências (composer require) interativamente:

* Você verá a mensagem: Would you like to define your dependencies (require) interactively [yes]?
* Digite no (ou n), pois você pode adicionar os pacotes depois conforme o projeto evoluir.
* Ele fará a mesma pergunta para dependências de desenvolvimento (require-dev), digite no novamente.

## 5. Confirmar a geração do arquivo
No final, ele mostrará uma prévia do seu arquivo composer.json e perguntará:

* Do you confirm generation [yes]?
* Pressione Enter (ou digite yes).

Pronto! O arquivo composer.json será criado na raiz do seu projeto.
------------------------------
## Dica extra: Configurando o Autoload para a sua árvore de pastas
Como você já criou uma árvore de pastas, você provavelmente precisará mapeá-las para que o PHP encontre suas classes automaticamente. Abra o arquivo composer.json recém-criado em um editor de texto e adicione o bloco autoload (usando o padrão PSR-4).
Por exemplo, se você tem uma pasta chamada src/ onde ficará o seu código:
```bash
{
    "name": "dev/minha-estrutura",
    "type": "project",
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    },
    "require": {}
}
```
Depois de salvar o arquivo editado, rode o comando composer dump-autoload no terminal para ativar o mapeamento.
```bash
composer dump-autoload



