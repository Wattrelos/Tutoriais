# Instalação e Configuração do Behat no PHP

Este guia demonstra como instalar e configurar o **Behat** com **Mink** em um ambiente PHP moderno (compatível com Symfony 7+), utilizando os pacotes atualizados da comunidade.

---

## 🛠️ Passo 1: Instalar as Dependências

Na raiz do seu projeto (onde está localizado o `composer.json`), execute o comando abaixo para instalar o Behat, o Mink, a extensão de compatibilidade moderna, o driver BrowserKit e o cliente HTTP do Symfony:

```bash
composer require --dev behat/behat behat/mink friends-of-behat/mink-extension behat/mink-browserkit-driver symfony/http-client
```

> **Nota:** Utilizamos o pacote `friends-of-behat/mink-extension` para garantir compatibilidade com versões modernas do Symfony/PHP.

---

## 📁 Passo 2: Inicializar a Estrutura de Pastas

Para criar a estrutura padrão de diretórios e arquivos de contexto do Behat, execute:

```bash
./vendor/bin/behat --init
```

Esse comando criará automaticamente a seguinte estrutura no seu projeto:

```text
meu-projeto/
└── features/
    └── bootstrap/
        └── FeatureContext.php  <-- Onde você escreverá o código PHP dos testes
```

---

## ⚙️ Passo 3: Criar o Arquivo de Configuração (`behat.yml`)

Crie o arquivo `behat.yml` na raiz do seu projeto com o seguinte conteúdo:

```yaml
default:
  suites:
    default:
      paths:    [ "%paths.base%/features" ]
      contexts: [ FeatureContext ]
  extensions:
    Behat\MinkExtension:
      base_url: http://localhost/
      sessions:
        default:
          browserkit_http: ~
```

> **Atenção:** 
> - A extensão é configurada sob a chave `Behat\MinkExtension` (namespace PHP interno mantido pelo pacote `friends-of-behat/mink-extension`).
> - A opção da sessão para o driver do BrowserKit HTTP é `browserkit_http`.

---

## 🧪 Passo 4: Atualizar o Contexto de Testes

Para utilizar os recursos de navegação web do Mink dentro do seu código PHP, altere a classe em `features/bootstrap/FeatureContext.php` para estender `RawMinkContext`:

```php
<?php

use Behat\MinkExtension\Context\RawMinkContext;

class FeatureContext extends RawMinkContext
{
    /**
     * @Given que eu estou na página inicial
     */
    public function queEuEstouNaPaginaInicial()
    {
        // Visita a URL definida no base_url do behat.yml
        $this->visitPath('/');
    }

    /**
     * @Then eu devo ver o texto :texto
     */
    public function euDevoVerOTexto($texto)
    {
        // Verifica se o texto existe na página atual
        $this->assertSession()->pageTextContains($texto);
    }
}
```

---

## 📝 Passo 5: Criar seu Primeiro Cenário Gherkin

Crie um arquivo de funcionalidade dentro da pasta `features/`, por exemplo, `features/home.feature`:

```gherkin
# language: pt
Funcionalidade: Acessar a Página Inicial

  Cenário: Verificar o título do site
    Dado que eu estou na página inicial
    Então eu devo ver o texto "Bem-vindo"
```

---

## 🚀 Passo 6: Executar os Testes

Para rodar a sua suíte de testes do Behat, execute no terminal:

```bash
./vendor/bin/behat
```
