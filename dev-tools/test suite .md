# Instalação de uma suit de testes em ambiente PHP

## Análise Estática

|Ferramenta| Descrição|
|---|---|
|PHPStan| Análise estática de código PHP|
|Psalm| Análise estática de código PHP|
|Rector| Refatoração automatizada de código PHP|

```bash
composer create-project phpstan/phpstan --prefer-dist --dev
./vendor/bin/phpstan analyse src
```

## Testes Automatizados

|Ferramenta| Descrição|
|---|---|
|PHPUnit| Framework de testes unitários para PHP|
|Pest| Framework de testes unitários para PHP|
|Mockery| Framework de mock para PHP|

```bash
composer create-project --dev phpunit/phpunit
./vendor/bin/phpunit --version
```

## Formatadores e Linters


|PHP-CS-Fixer| Formatador de código PHP|
```bash
composer require --dev friendsoftphp/php-cs-fixer
./vendor/bin/php-cs-fixer --version
```

|PHP_CodeSniffer| Formatador de código PHP|
```bash
composer require --dev squizlabs/php_codesniffer
./vendor/bin/phpcs --version
```


 BDD e Linguagem de Negócio

|Ferramenta| Descrição|
|---|---|
|Gherkin| Formato de especificação de comportamento de software|
|Behat| Framework de testes de aceitação para PHP|
    
```bash
composer require --dev behat/behat
./vendor/bin/behat --version
```

## Navegador Headless e Visão Computaciona

|Ferramenta| Descrição|
|---|---|
|Platwrite| Ferramenta de visão computacional para PHP|

```bash
composer require --dev platwrite/platwrite
./vendor/bin/platwrite --version
```

##  Métricas e Integridade de Arquitetura

|Ferramenta| Descrição|
|---|---|
|PHP Insights| Ferramenta de métricas e integridade de arquitetura para PHP|
|Deptrac| Ferramenta de métricas e integridade de arquitetura para PHP|

```bash
composer require --dev phpinsights/phpinsights
./vendor/bin/phpinsights --version
```

```bash
composer require --dev deptrac/deptrac
./vendor/bin/deptrac --version
```

Frameworks Nativos para Criar Agentes em PHP

|Ferramenta| Descrição|
|---|---|
|LLPhant| Framework de desenvolvimento de agentes de IA para PHP|
|PHP Prism| Framework de desenvolvimento de agentes de IA para PHP|

```bash
composer require --dev llphant/llphant
./vendor/bin/llphant --version
```

```bash
composer require --dev php-prism/php-prism
./vendor/bin/php-prism --version
```

