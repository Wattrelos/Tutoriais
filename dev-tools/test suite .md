# 🧪 Guia Completo da Suíte de Testes, Qualidade e Ferramentas para Agentes em PHP

> **Contexto:** Este guia complementa o tutorial [🤖 Ferramentas como Superpoderes para Agentes de IA](../agents/ferramentas-que-ajudam-agents.md) e adapta a suíte de testes e ferramentas de análise estática para o nosso ambiente real de desenvolvimento (**Debian 13**, **PHP 8.4 runtime / 8.2+**, **Slim 4 / beta-engine** e **Google Antigravity**).

---

## 🧭 Visão Geral: Dois Tipos de Ferramentas

Ao equipar nosso ambiente e os agentes de IA com ferramentas, encontramos dois tipos fundamentais de pacotes:

1. **Ferramentas de Linha de Comando (CLI Tools):**
   - Geram executáveis diretos dentro de `./vendor/bin/` (ou via `npx` no ecossistema Node).
   - O desenvolvedor e os agentes executam no terminal para obter feedback booleano (passou/falhou): `pest`, `phpstan`, `php-cs-fixer`, `phpunit`, `behat`.
2. **Bibliotecas e SDKs de Código (In-Code Libraries):**
   - **Não** geram executáveis em `./vendor/bin/`.
   - São invocadas dentro do código PHP via classes e métodos (`Mockery`, `LLPhant`, `Prism`).
   - Sua instalação é verificada via `composer show <pacote>`.

> [!IMPORTANT]
> **Atenção ao instalar pacotes em projetos existentes:**
> - Nunca use `composer create-project` para adicionar ferramentas a um projeto que já existe (isso tenta criar uma nova aplicação em outra pasta).
> - Utilize sempre **`composer require --dev <vendor>/<pacote>`**.

---

## ⚡ Mapa de Ferramentas no Nosso Projeto (`agsonhos2`)

No nosso projeto atual (`/var/www/html/agsonhos2`), o ecossistema está padronizado e pronto para uso pelo time e pelos agentes de IA:

| Categoria | Ferramenta | Pacote / Origem | Executável / Validação | Status no Projeto |
| :--- | :--- | :--- | :--- | :--- |
| **Testes Modernos (TDD)** | **Pest PHP** | `pestphp/pest` | `./vendor/bin/pest` | 🟢 Ativo (437+ testes) |
| **Testes Unitários Legados** | **PHPUnit** | `phpunit/phpunit` | `./vendor/bin/phpunit` | 🟢 Ativo (`tests/Unit`) |
| **Mocks e Dublês** | **Mockery** | `mockery/mockery` | `composer show mockery/mockery` | 🟢 Ativo no Composer |
| **Análise Estática** | **PHPStan** | `phpstan/phpstan` | `./vendor/bin/phpstan` | 🟢 Ativo (`phpstan.neon`) |
| **Linter e Formatação** | **PHP-CS-Fixer** | `friendsofphp/php-cs-fixer` | `./vendor/bin/php-cs-fixer` | 🟢 Ativo (`.php-cs-fixer.dist.php`) |
| **BDD & Aceitação** | **Behat + Mink** | `behat/behat` | `./vendor/bin/behat` | 🟢 Ativo (`features/`) |
| **E2E & Visão Web** | **Playwright** | `@playwright/test` (Node) | `npx playwright test` | 🟢 Configurado (`package.json`) |

---

## 🚀 Scripts Rápidos no `composer.json`

Para padronizar os comandos e facilitar a execução tanto por humanos quanto por agentes de IA, o `composer.json` do projeto dispõe dos seguintes atalhos:

```bash
# Executa todos os testes com Pest (saída visual moderna)
composer test

# Executa apenas os testes unitários via PHPUnit
composer test:unit

# Executa apenas os testes de integração
composer test:integration

# Executa análise estática de tipos no backend
composer phpstan

# Verifica conformidade de estilo de código (PSR-12) sem alterar arquivos
composer cs-check

# Aplica correções automáticas de estilo em backend/src e tests/
composer cs-fix

# Executa cenários de aceitação em Gherkin
composer behat
```

---

## 🧰 Detalhamento por Categoria

---

### 1. Análise Estática de Código (O Juiz da Tipagem)

Analisadores estáticos inspecionam o código em busca de bugs de tipagem, métodos inexistentes ou retornos inconsistentes sem precisar rodar a aplicação.

#### A. PHPStan (Recomendado)
- **Instalação:**
  ```bash
  composer require --dev phpstan/phpstan
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/phpstan --version
  ```
- **Execução:**
  ```bash
  # Analisa o código do backend utilizando o arquivo de configuração phpstan.neon
  ./vendor/bin/phpstan analyse
  
  # Ou apontando diretamente com nível de rigor (0 a 9)
  ./vendor/bin/phpstan analyse backend/src --level=4
  ```

#### B. Psalm
- **Instalação:**
  ```bash
  composer require --dev vimeo/psalm
  ```
- **Execução:**
  ```bash
  ./vendor/bin/psalm --init
  ./vendor/bin/psalm
  ```

#### C. Rector (Refatoração Automatizada de AST)
- **Instalação:**
  ```bash
  composer require --dev rector/rector
  ```
- **Execução:**
  ```bash
  ./vendor/bin/rector process backend/src --dry-run
  ```

---

### 2. Testes Automatizados e TDD (O Motor de Auto-Cura)

A suíte de testes fornece a verdade binária que os modelos de IA precisam para validar suas soluções.

#### A. Pest PHP
Construído sobre o PHPUnit, oferece sintaxe elegante e amigável para agentes de IA:
- **Instalação:**
  ```bash
  composer require --dev pestphp/pest --with-all-dependencies
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/pest --version
  ```
- **Execução:**
  ```bash
  # Roda toda a suíte
  ./vendor/bin/pest

  # Roda apenas um arquivo específico
  ./vendor/bin/pest tests/Unit/EmailMessageTest.php

  # Filtra testes por nome
  ./vendor/bin/pest --filter="EmailMessage"
  ```

#### B. PHPUnit
O padrão tradicional da indústria PHP:
- **Instalação:**
  ```bash
  composer require --dev phpunit/phpunit
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/phpunit --version
  ```
- **Execução:**
  ```bash
  ./vendor/bin/phpunit -c phpunit.xml tests/Unit
  ```

#### C. Mockery (Dublês de Testes)
> [!NOTE]
> O **Mockery é uma biblioteca interna de código**, não uma ferramenta CLI de terminal. Portanto, **não existe** o comando `./vendor/bin/mockery`.

- **Instalação:**
  ```bash
  composer require --dev mockery/mockery
  ```
- **Como verificar se está instalado:**
  ```bash
  composer show mockery/mockery
  ```
- **Exemplo de uso nos testes:**
  ```php
  use Mockery;
  use App\Domain\Customer\CustomerRepositoryInterface;

  it('salva o cliente usando mock', function () {
      $mock = Mockery::mock(CustomerRepositoryInterface::class);
      $mock->shouldReceive('save')->once()->andReturnTrue();

      expect($mock->save())->toBeTrue();
  });
  ```

---

### 3. Formatadores e Linters (A Guia de Estilo)

Garantem conformidade estrita com as normas PSR-12 e PER Coding Style 2.0.

#### A. PHP-CS-Fixer
- **Instalação:**
  ```bash
  composer require --dev friendsofphp/php-cs-fixer
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/php-cs-fixer --version
  ```
- **Execução:**
  ```bash
  # Apenas inspeciona e exibe diferenças (sem alterar arquivos)
  ./vendor/bin/php-cs-fixer fix --dry-run --diff

  # Aplica as correções automáticas de formatação
  ./vendor/bin/php-cs-fixer fix
  ```

#### B. PHP_CodeSniffer
- **Instalação:**
  ```bash
  composer require --dev squizlabs/php_codesniffer
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/phpcs --version
  ```
- **Execução:**
  ```bash
  # Inspeciona violações
  ./vendor/bin/phpcs --standard=PSR12 backend/src
  
  # Aplica correções automáticas
  ./vendor/bin/phpcbf --standard=PSR12 backend/src
  ```

---

### 4. BDD e Linguagem de Negócio (A Ponte com o Cliente)

Permite escrever regras de negócio em formato **Gherkin** (`Dado / Quando / Então`) que tanto humanos quanto LLMs compreendem sem ambiguidade.

#### Behat + Mink
- **Instalação das dependências modernas:**
  ```bash
  composer require --dev behat/behat behat/mink friends-of-behat/mink-extension behat/mink-browserkit-driver symfony/http-client
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/behat --version
  ```
- **Execução dos cenários:**
  ```bash
  ./vendor/bin/behat
  ```
> Veja o guia passo a passo completo em [behat.md](behat.md).

---

### 5. Navegador Headless e Visão Web (Os Olhos do Agente)

> [!WARNING]
> O **Playwright não é um pacote PHP** (não existe `platwrite/platwrite` no Composer). O Playwright é mantido pela Microsoft no ecossistema **Node.js** e opera em nível de sistema operacional controlando instâncias de Chromium, Firefox e WebKit.

#### Playwright
- **Instalação e configuração de sistema:**
  ```bash
  # No Debian/Ubuntu, instala as dependências de sistema para os navegadores
  sudo npx playwright install-deps
  npx playwright install chromium
  ```
- **Execução dos testes E2E do projeto (`agsonhos2`):**
  ```bash
  npm run test:e2e
  npm run test:e2e:smoke
  ```
- **Integração com Agentes de IA via MCP:**
  Permite que agentes autônomos naveguem pelo frontend, inspecionem o DOM e capturem screenshots.
  > Veja o tutorial detalhado de integração em [playwright-install.md](playwright-install.md).

---

### 6. Métricas e Integridade de Arquitetura

Ferramentas para impedir código espaguete e monitorar a complexidade do sistema.

#### A. PHP Insights
- **Instalação:**
  ```bash
  composer require --dev nunomaduro/phpinsights
  ```
- **Execução:**
  ```bash
  ./vendor/bin/phpinsights analyse backend/src
  ```

#### B. Deptrac
Valida que regras de camadas (ex: Clean Architecture / Hexagonal) sejam respeitadas:
- **Instalação:**
  ```bash
  composer require --dev deptrac/deptrac
  ```
- **Verificação de versão:**
  ```bash
  ./vendor/bin/deptrac --version
  ```
- **Execução:**
  ```bash
  ./vendor/bin/deptrac analyse
  ```

---

### 7. Bibliotecas para Criação de Agentes de IA em PHP

Se você estiver desenvolvendo funcionalidades com Inteligência Artificial dentro da aplicação PHP:

#### A. LLPhant (Generative AI framework para PHP)
- **Instalação:**
  ```bash
  composer require theodo-group/llphant
  ```
- **Uso:** Framework inspirado em LangChain para orquestrar LLMs, vetores e Function Calling diretamente em PHP.

#### B. Prism PHP
- **Instalação:**
  ```bash
  composer require echolabsdev/prism
  ```
- **Uso:** Camada unificada para invocar OpenAI, Anthropic, Gemini, Ollama com ferramentas e respostas estruturadas.

---

## 🤖 Como Instruir os Agentes de IA a Usar Essas Ferramentas

Para garantir que os agentes sigam o ciclo autônomo de auto-cura (*Self-Healing*), inclua em suas diretrizes de projeto (ex: `GEMINI.md`, `AGENTS.md` ou regras do workspace):

```markdown
### 🛠️ Protocolo de Qualidade Obrigatório para o Agente

1. **Ciclo TDD:**
   - Antes de implementar alterações, crie ou adapte os testes com Pest em `tests/`.
   - Execute `composer test` e garanta que todos os testes passem.
   
2. **Análise Estática:**
   - Após alterar arquivos PHP, execute: `composer phpstan`.
   - Se o PHPStan apontar inconsistências de tipo ou classes indefinidas, analise e corrija autonomamente.

3. **Padronização de Código:**
   - Antes de concluir sua resposta, execute: `composer cs-fix`.
   - Garanta que todo código novo obedeça às diretrizes da PSR-12.
```
