# Guia Completo: Instalação do Rust no Debian 13 e Ecossistema de Qualidade para Agentes de IA

Este guia orienta a instalação e configuração do ambiente de desenvolvimento **Rust** no **Debian 13 (Trixie)**, além de apresentar as ferramentas equivalentes ao ecossistema PHP (PHPUnit, Pest, PHPStan, Behat/Gherkin, Playwright) que potencializam o desenvolvimento autônomo e assistido por **Agentes de IA** (como o Google Antigravity e Gemini).

---

## 1. Preparação do Sistema (Debian 13)

O Rust compila código nativo e depende de bibliotecas C e ferramentas essenciais de compilação (*linker*, GCC e cabeçalhos de sistema).

### 1.1 Atualizar os repositórios do sistema
Abra o terminal e garanta que os pacotes do Debian 13 estejam atualizados:

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Instalar dependências essenciais
Instale as ferramentas de compilação, o utilitário `curl`, `git` e bibliotecas SSL:

```bash
sudo apt install -y build-essential curl gcc pkg-config libssl-dev git
```

> [!NOTE]
> O pacote `build-essential` fornece o `gcc`, `make` e as bibliotecas C padrão (`glibc`) necessárias para a vinculação (*linking*) dos binários gerados pelo Rust.

---

## 2. Instalação Oficial do Rust via Rustup

A forma recomendada de instalar o Rust em qualquer distribuição Linux é através do **Rustup**, o instalador e gerenciador oficial de toolchains.

> [!IMPORTANT]
> **Evite** instalar o Rust via `apt install rustc cargo`. Os pacotes dos repositórios Debian costumam ser defasados e não oferecem facilidade de alternar canais (*stable*, *beta*, *nightly*) nem instalar componentes essenciais como `rust-analyzer` e `clippy`.

### 2.1 Executar o script do Rustup
Execute o instalador oficial:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Quando o assistente interativo for exibido:
1. Pressione **`1`** (opção padrão: `Proceed with standard installation (default)`).
2. Pressione **Enter**.

### 2.2 Configurar as variáveis de ambiente
O Rustup instala os binários em `~/.cargo/bin`. Ative as variáveis na sessão atual:

```bash
source "$HOME/.cargo/env"
```

*(O instalador já adiciona automaticamente essa linha ao teu `~/.bashrc` ou `~/.profile`).*

### 2.3 Validar a instalação
Verifique se o compilador (`rustc`) e o gerenciador de pacotes/construção (`cargo`) estão operacionais:

```bash
rustc --version
cargo --version
rustup --version
```

---

## 3. Componentes Oficiais Indispensáveis

Para preparar o ambiente para desenvolvimento profissional e integração com IDEs e Agentes de IA, instale os seguintes componentes:

```bash
# Language Server Protocol (usado pelo VS Code, Neovim e agentes de IA)
rustup component add rust-analyzer

# Linter oficial e análise estática avançada
rustup component add clippy

# Formatador oficial de código segundo os padrões da comunidade
rustup component add rustfmt
```

---

## 4. Comparativo: Ecossistema PHP vs. Ecossistema Rust

Para quem vem do ecossistema PHP profissional, o Rust não apenas possui alternativas diretas como também se destaca pela **confiabilidade em tempo de compilação**.

Abaixo está o mapeamento dos recursos comuns no PHP para seus equivalentes em Rust:

| Categoria | PHP | Rust | Papel no Desenvolvimento |
| :--- | :--- | :--- | :--- |
| **Testes Unitários / Integração** | PHPUnit | **`cargo test`** (Nativo) | Testes unitários integrados na linguagem com execução paralela nativa. |
| **Testes Expressivos / BDD** | Pest PHP | **`rstest`** / **`pretty_assertions`** | Sintaxe expressiva, testes parametrizados (data providers), fixtures e diffs visuais coloridos. |
| **Testes de Snapshot** | Spatie Snapshot / Pest Snapshot | **`insta`** | Testes de snapshot com revisão interativa via CLI. |
| **Mocking / Fakes** | Mockery / PHPUnit Mocks | **`mockall`** | Criação automática de mocks para traits e structs via macros. |
| **Análise Estática & Tipagem** | PHPStan / Psalm (Nível 0-8) | **`rustc`** + **`cargo clippy`** | O compilador garante tipagem e segurança de memória em tempo de build; o Clippy identifica más práticas e otimizações. |
| **BDD & Gherkin** | Behat | **`cucumber-rs`** | Suporte completo a arquivos `.feature` com Gherkin para testes de aceitação e comportamento. |
| **E2E & Automação de Browser** | Playwright (PHP) / Panthère | **`playwright-rust`** / **`thirtyfour`** | Automação e testes de ponta a ponta controlando navegadores reais (Chromium, Firefox, WebKit). |
| **Análise de Vulnerabilidades** | Composer Audit / LocalPHPCheck | **`cargo-audit`** | Varredura de dependências contra o banco de vulnerabilidades da RustSec. |
| **Execução Contínua em Dev** | Pest `--watch` / PHP-Watcher | **`cargo-watch`** | Reexecuta testes e compilação a cada arquivo alterado. |

---

## 5. Detalhamento e Uso dos Equivalentes no Rust

### 5.1 Testes Unitários: O Sistema Nativo do Rust (vs. PHPUnit)
Diferente do PHP, onde você precisa instalar o PHPUnit via Composer, o Rust já vem com testes embutidos no núcleo da linguagem e no Cargo:

```rust
// src/calculadora.rs
pub fn somar(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_somar() {
        assert_eq!(somar(2, 3), 5);
    }
}
```

Executando os testes:
```bash
cargo test
```

### 5.2 Testes Expressivos e Parametrizados: `rstest` e `pretty_assertions` (vs. Pest PHP)
O [rstest](https://crates.io/crates/rstest) traz recursos semelhantes aos testes com dataset/providers do Pest PHP, além de fixtures limpas.

Adicione ao `Cargo.toml`:
```toml
[dev-dependencies]
rstest = "0.24"
pretty_assertions = "1.4"
```

Exemplo de teste parametrizado (equivalente ao `with()` do Pest):
```rust
use rstest::rstest;
use pretty_assertions::assert_eq;

#[rstest]
#[case(2, 2, 4)]
#[case(3, 5, 8)]
#[case(-1, 1, 0)]
fn test_somas_multiplas(#[case] a: i32, #[case] b: i32, #[case] esperado: i32) {
    assert_eq!(a + b, esperado);
}
```

### 5.3 Mocks: `mockall` (vs. Mockery)
Para simular comportamentos de serviços externos, banco de dados ou APIs, o `mockall` gera mocks automáticos a partir de `traits`:

```toml
[dev-dependencies]
mockall = "0.13"
```

```rust
use mockall::automock;

#[automock]
pub trait Notificador {
    fn enviar_email(&self, destinatario: &str, mensagem: &str) -> bool;
}

#[test]
fn test_notificacao() {
    let mut mock = MockNotificador::new();
    mock.expect_enviar_email()
        .with(mockall::predicate::eq("admin@teste.com"), mockall::predicate::always())
        .times(1)
        .returning(|_, _| true);

    assert!(mock.enviar_email("admin@teste.com", "Olá!"));
}
```

### 5.4 Análise Estática: `cargo clippy` (vs. PHPStan / Psalm)
No PHP, ferramentas como o PHPStan inferem tipos dinâmicos para prevenir erros em tempo de execução. No Rust:
1. **O compilador (`rustc`)** é o nível máximo de rigor de tipos, concorrência e gerenciamento de memória (sem Garbage Collector).
2. **O `cargo clippy`** atua como o super-linter, identificando código não-idiomático, gargalos de performance e potenciais bugs lógicos.

Execução:
```bash
cargo clippy
```

Para aplicar correções automáticas:
```bash
cargo clippy --fix
```

### 5.5 BDD com Gherkin: `cucumber-rs` (vs. Behat)
Se teu time utiliza especificações em linguagem natural (Given/When/Then), o [cucumber-rs](https://crates.io/crates/cucumber) oferece paridade completa com o Behat.

Estrutura de diretórios:
```text
meu_projeto/
├── tests/
│   ├── features/
│   │   └── login.feature
│   └── cucumber.rs
```

Exemplo do arquivo `tests/features/login.feature`:
```gherkin
Feature: Autenticação de Usuário
  Scenario: Login com credenciais válidas
    Given que o usuário possui conta ativa
    When ele submete o login com usuário e senha corretos
    Then ele deve receber um token de acesso válido
```

Exemplo da implementação dos passos (`tests/cucumber.rs`):
```rust
use cucumber::{given, when, then, World};

#[derive(Debug, Default, World)]
pub struct AppWorld {
    usuario_ativo: bool,
    token_recebido: Option<String>,
}

#[given("que o usuário possui conta ativa")]
fn usuario_ativo(world: &mut AppWorld) {
    world.usuario_ativo = true;
}

#[when("ele submete o login com usuário e senha corretos")]
fn submeter_login(world: &mut AppWorld) {
    if world.usuario_ativo {
        world.token_recebido = Some("jwt.token.valido".to_string());
    }
}

#[then("ele deve receber um token de acesso válido")]
fn verificar_token(world: &mut AppWorld) {
    assert!(world.token_recebido.is_some());
}

#[tokio::main]
async fn main() {
    AppWorld::run("tests/features").await;
}
```

### 5.6 Testes E2E e Automação de Navegador: `playwright-rust` e `thirtyfour` (vs. Playwright PHP)
Para testar aplicações web de ponta a ponta controlando navegadores:

- **[playwright-rust](https://crates.io/crates/playwright)**: Wrapper direto do Playwright oficial.
- **[thirtyfour](https://crates.io/crates/thirtyfour)**: Biblioteca assíncrona para WebDriver / Selenium e navegadores headless.

Exemplo com `thirtyfour` (Chromium / Firefox):
```toml
[dev-dependencies]
thirtyfour = "0.34"
tokio = { version = "1", features = ["full"] }
```

```rust
use thirtyfour::prelude::*;

#[tokio::test]
async fn test_navegador() -> WebDriverResult<()> {
    let caps = DesiredCapabilities::chrome();
    let driver = WebDriver::new("http://localhost:9515", caps).await?;

    driver.goto("https://antigravity.dev").await?;
    let elem = driver.find(By::Tag("h1")).await?;
    assert!(!elem.text().await?.is_empty());

    driver.quit().await?;
    Ok(())
}
```

---

## 6. Por que o Rust é o Ambiente Ideal para Agentes de IA (Antigravity & Gemini)

O ecossistema Rust oferece vantagens incomparáveis para o desenvolvimento assistido por agentes de IA:

1. **Mensagens de Diagnóstico Extremamente Prescritivas:**
   Ao contrário de linguagens interpretadas, o `rustc` e o `clippy` indicam exatamente onde está o erro, o porquê dele ocorrer e trazem sugestões diretas com `help: try: ...`. Os agentes de IA conseguem interpretar esses erros e autocorrigir o código de forma precisa e autônoma.

2. **Tipagem e Contratos Estritos (Zero Alucinação de Tipos):**
   Agentes de IA raramente introduzem bugs de ponteiro nulo (`null pointer`) ou erros de tipo inesperados no Rust, pois o compilador impede a geração do binário caso haja qualquer inconsistência.

3. **Ciclo de Feedback Rápido com `cargo check`:**
   O comando `cargo check` valida a integridade de todo o projeto sem gerar os binários finais, permitindo que os agentes testem hipóteses em frações de segundo.

4. **Ferramentas de Suporte Recomendadas para Agentes:**
   Instale utilitários complementares via `cargo`:
   ```bash
   # Test runner moderno, rápido e com saída estruturada (ideal para CI e Agentes)
   cargo install cargo-nextest --locked

   # Observador de arquivos para testes contínuos
   cargo install cargo-watch --locked

   # Auditoria de segurança de dependências
   cargo install cargo-audit --locked
   ```

---

## 7. Comandos Essenciais do Dia a Dia

```bash
# Criar um novo projeto executável
cargo new meu_app --bin

# Criar uma nova biblioteca
cargo new minha_lib --lib

# Compilar e verificar erros rapidamente
cargo check

# Executar testes unitários e de integração
cargo test

# Executar testes com o cargo-nextest (mais rápido e limpo)
cargo nextest run

# Executar o linter de boas práticas
cargo clippy

# Formatar o código de acordo com o padrão oficial
cargo fmt

# Atualizar o compilador Rust e ferramentas para a versão mais recente
rustup update
```

---

## Conclusão

Com o Rust instalado no **Debian 13**, você tem em mãos uma das linguagens mais robustas da atualidade. A transição das práticas de qualidade do PHP (PHPUnit, Pest, PHPStan, Behat) para o Rust é natural e amplamente suportada pela comunidade, proporcionando aos **Agentes de IA (Antigravity/Gemini)** um ecossistema com feedback rigoroso, testes de alta velocidade e código previsível.
