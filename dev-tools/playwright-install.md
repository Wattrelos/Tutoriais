# Instalação e Configuração do Playwright com Google Antigravity no Debian 13

Guia passo a passo para instalar o **Playwright** e integrá-lo ao **Google Antigravity** (IDE e CLI) no **Debian 13 (Trixie)**. 

Essa integração utiliza o protocolo **MCP (Model Context Protocol)**, permitindo que os agentes autônomos do Antigravity controlem navegadores para inspecionar páginas web, tirar capturas de tela, validar fluxos de interface e executar testes automatizados diretamente no seu ambiente de desenvolvimento.

---

## Pré-requisitos

Antes de iniciar, certifique-se de que o sistema possui os seguintes componentes instalados:

- **Debian 13 (Trixie)** com privilégios de `sudo`.
- **Node.js** (versão 18 ou superior) e gerenciador de pacotes **npm**:
  ```bash
  node -v
  npm -v
  ```
- **Google Antigravity IDE** ou **Antigravity CLI** instalado (veja o guia [antigravity-ide.md](antigravity-ide.md)).

---

## Passo 1: Instalar as Dependências do Playwright no Debian 13

No Linux, os navegadores modernos (Chromium, Firefox e WebKit) necessitam de bibliotecas gráficas e de sistema do Debian para funcionar em modo headless e com interface gráfica.

1. **Instale as dependências de sistema exigidas pelos navegadores:**
   O Playwright disponibiliza um utilitário que instala automaticamente via `apt` todas as bibliotecas necessárias:
   ```bash
   sudo npx playwright install-deps
   ```

2. **Baixe os binários do navegador (Chromium):**
   Para a grande maioria das automações e integração com o Antigravity, o Chromium é o navegador padrão e mais estável:
   ```bash
   npx playwright install chromium
   ```

> [!TIP]
> Caso queira instalar todos os navegadores suportados (Chromium, Firefox e WebKit), execute apenas:
> ```bash
> npx playwright install
> ```

3. **(Opcional) Instalar a CLI do Playwright globalmente:**
   Útil para inspecionar rotas, abrir o gerador de código (`codegen`) ou executar testes via terminal:
   ```bash
   sudo npm install -g @playwright/cli@latest
   ```

---

## Passo 2: Conectar o Playwright ao Google Antigravity via MCP

O **Model Context Protocol (MCP)** é o mecanismo pelo qual o Antigravity obtém ferramentas de navegação e automação web.

Você pode configurar o servidor MCP do Playwright por dois métodos:

### Opção 1: Via Arquivo de Configuração Global (Recomendado)

O Antigravity lê as configurações globais de MCP em `~/.gemini/config/mcp_config.json`.

1. Crie o diretório de configurações (caso ainda não exista):
   ```bash
   mkdir -p ~/.gemini/config
   ```

2. Crie ou edite o arquivo `~/.gemini/config/mcp_config.json`:
   ```bash
   nano ~/.gemini/config/mcp_config.json
   ```

3. Insira a configuração do servidor MCP do Playwright:
   ```json
   {
     "mcpServers": {
       "playwright": {
         "command": "npx",
         "args": ["-y", "@playwright/mcp"]
       }
     }
   }
   ```
   *Pressione `Ctrl + O` e `Enter` para salvar, depois `Ctrl + X` para sair.*

### Opção 2: Pela Interface Gráfica do Antigravity IDE

Se você estiver utilizando o **Antigravity IDE**:

1. Abra a IDE do Antigravity.
2. Acesse as configurações com o atalho `Ctrl + ,` ou clique no menu de **Opções Adicionais (...) > MCP Servers**.
3. Clique no botão **Adicionar Servidor MCP** (*Add MCP Server*).
4. Preencha as informações:
   - **Nome / ID:** `playwright`
   - **Comando (Command):** `npx`
   - **Argumentos (Args):** `["-y", "@playwright/mcp"]`
5. Salve as alterações.
6. Reinicie a IDE ou recarregue a janela (`Ctrl + Shift + P` > *Developer: Reload Window*). O indicador ao lado de `playwright` deverá ficar verde, indicando conexão ativa.

---

## Passo 3: Configurar Habilidades (Skills) no Antigravity

As habilidades (*Skills*) ensinam ao agente boas práticas e comandos específicos de automação com o Playwright.

No ecossistema Antigravity, as habilidades são descobertas nos seguintes locais:
- **No Projeto (Workspace):** Na pasta `.agents/skills/` na raiz do seu projeto.
- **Globalmente (Para todos os projetos):** Na pasta `~/.gemini/config/skills/`.

Para adicionar uma habilidade global do Playwright:

```bash
# Cria o diretório de habilidades globais
mkdir -p ~/.gemini/config/skills

# Baixa os modelos de habilidades do Playwright (opcional)
npx degit microsoft/playwright-cli/skills/playwright-cli ~/.gemini/config/skills/playwright-cli
```

---

## Passo 4: Validar e Usar no Antigravity

Com o servidor MCP ativo, o agente do Antigravity terá acesso a um conjunto de ferramentas de navegador (`browser_navigate`, `browser_click`, `browser_take_screenshot`, `browser_snapshot`, etc.).

1. Abra o painel de chat do agente no Antigravity (`Ctrl + L`).
2. Teste a integração com comandos em linguagem natural:
   - *"Abra o navegador e acesse https://example.com para validar se a conexão está funcionando."*
   - *"Tire um print da página inicial da aplicação local em http://localhost:80 e salve nos artefatos."*
   - *"Inspecione o formulário de login e verifique quais seletores estão disponíveis."*

> [!NOTE]
> Quando o agente executa ações no navegador, uma janela do Chromium pode ser aberta de forma gerenciada pelo MCP ou executada em segundo plano gerando relatórios de texto e capturas de tela.

---

## Passo 5: Criar um Projeto de Testes com Playwright (Opcional)

Se além da automação do agente você desejar criar suítes de testes automatizados E2E (End-to-End) em TypeScript ou JavaScript no seu projeto:

1. Acesse a pasta do seu projeto:
   ```bash
   cd /caminho/do/seu/projeto
   ```

2. Inicialize o Playwright Test:
   ```bash
   npm init playwright@latest
   ```
   *O assistente interativo perguntará a linguagem (TypeScript/JavaScript), onde salvar os testes (`tests/`) e se deseja instalar os navegadores.*

3. Para rodar a suíte de testes:
   ```bash
   npx playwright test
   ```

4. Para abrir o relatório interativo dos testes:
   ```bash
   npx playwright show-report
   ```

---

## Resolução de Problemas (Troubleshooting)

### 1. Erro de bibliotecas compartilhadas ausentes (`libgbm.so`, `libasound.so`, etc.)
Se ao iniciar o navegador ocorrer um erro indicando bibliotecas ausentes:
```bash
sudo npx playwright install-deps
```
Certifique-se de executar com `sudo`, pois esse comando instala pacotes via gerenciador `apt` do Debian.

### 2. Falha de download de binários (HTTP 404 ou restrições de rede)
Se o download automático dos navegadores via Playwright falhar por instabilidade de rede ou bloqueio de CDN:
1. Instale o pacote do Chromium do próprio Debian:
   ```bash
   sudo apt update
   sudo apt install -y chromium chromium-sandbox
   ```
2. Ou force o download específico do Chromium pelo Playwright:
   ```bash
   npx playwright install --with-deps chromium
   ```

### 3. O servidor MCP não conecta (indicador cinza ou vermelho)
1. Verifique se o arquivo `~/.gemini/config/mcp_config.json` possui sintaxe JSON válida:
   ```bash
   cat ~/.gemini/config/mcp_config.json
   ```
2. Teste a execução manual do servidor MCP no terminal para verificar se há algum erro de inicialização:
   ```bash
   npx -y @playwright/mcp
   ```
   *Se o comando iniciar aguardando mensagens de entrada JSON/RPC, o pacote está funcionando perfeitamente.*
