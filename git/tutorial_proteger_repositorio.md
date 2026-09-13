# Tutorial Prático: Como Proteger Repositórios Git Contra Vazamento de Credenciais e Segredos

> **Objetivo:** Implementar um ecossistema de defesa em camadas (*Defense in Depth*) que analisa automaticamente o código e **impede commits e pushes** sempre que detectar chaves de API, senhas, tokens privados ou credenciais sensíveis.

---

## 1. O Problema: Por Que a Atenção Humana Sempre Falha

Em projetos de software, o vazamento acidental de credenciais é um dos erros humanos mais frequentes e com maior potencial destrutivo. Basta um desenvolvedor cansado às 23h rodar:

```bash
git add .
git commit -m "Ajustes finais da integração com gateway de pagamento"
git push origin main
```

Se um token de produção (Stripe, AWS, OpenAI, banco de dados ou chave JWT) estiver acidentalmente inserido no código ou no `.env`, robôs automatizados na internet detectam o segredo em **menos de 60 segundos** após o envio para o GitHub.

### O Princípio da Segurança: Automatizar o Bloqueio no "Gargalo"
Não se combate erro humano com "atenção redobrada"; combate-se com **automação e portões de segurança (*Quality Gates*)**. A estratégia ideal divide-se em **três linhas de defesa**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AS 3 LINHAS DE DEFESA                             │
├─────────────────────────────────────────────────────────────────────────┤
│ 1. LOCAL (Máquina do Dev):       Git Pre-Commit Hook (Gitleaks / Husky) │
│    ↳ Bloqueia o "git commit" antes de gravar no histórico local.        │
├─────────────────────────────────────────────────────────────────────────┤
│ 2. SERVIDOR REMOTO (GitHub):     GitHub Push Protection                 │
│    ↳ Bloqueia o "git push" se algum commit contiver credenciais.        │
├─────────────────────────────────────────────────────────────────────────┤
│ 3. PIPELINE DE INTEGRAÇÃO (CI):  GitHub Actions Secret Scanner          │
│    ↳ Audita Pull Requests e branches contra vazamentos de terceiros.   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Linha de Defesa Remota: Ativando o GitHub Push Protection

O bloqueio que você vivenciou com o erro `GH013: Repository rule violations found` é fornecido pelo **GitHub Secret Scanning & Push Protection**.

### Como Funciona o Push Protection?
O GitHub mantém um catálogo com centenas de expressões regulares e formatos oficiais de tokens de mais de 100 parceiros globais (Stripe, AWS, Google Cloud, Microsoft, Slack, OpenAI, Twilio, etc.).

Quando você executa `git push`, o GitHub analisa todos os commits recebidos no momento da transação. Se detectar qualquer padrão de chave viva:
1. **Rejeita a transação de push imediatamente.**
2. Exibe no terminal a lista de arquivos e linhas onde a credencial foi encontrada.
3. Fornece uma URL exclusiva para liberação em caso de falso positivo justificado.

### Como Ativar no Seu Repositório GitHub

> [!NOTE]
> Em **repositórios públicos**, o Secret Scanning e o Push Protection são **gratuitos** e podem ser habilitados diretamente nas configurações. Em repositórios privados, o recurso está disponível para contas individuais e organizações que utilizem GitHub Advanced Security.

Siga os passos abaixo:

1. Acesse a página do repositório no GitHub (`https://github.com/SeuUsuario/SeuRepositorio`).
2. Clique na aba **Settings** (Configurações) no menu superior.
3. No menu lateral esquerdo, na seção **Security**, clique em **Code security and analysis**.
4. Localize o bloco **Secret scanning**:
   - Clique em **Enable** em *Secret scanning*.
   - Marque a caixa ou clique em **Enable** em **Push protection**.

```
[Settings] ➔ [Code security and analysis]
   ├── Secret scanning  ──────────► [ Enable ]
   └── Push protection ──────────► [ Enable ] (Bloqueia o push no terminal)
```

Pronto! A partir desse momento, nenhum desenvolvedor conseguirá dar `push` em nenhuma branch contendo tokens reconhecidos.

---

## 3. Linha de Defesa Local: Impedindo o Commit Antes que Aconteça

Embora o GitHub Push Protection seja excelente, ele atua apenas no momento do `git push`. Se ele bloquear o envio, o commit sensível **já foi gravado na sua máquina**, obrigando você a reescrever o histórico (`git rebase` ou `git commit --amend`) para poder subir seu código.

A melhor prática é **impedir a gravação do commit localmente** através de um **Git Hook de Pre-Commit**.

### A Ferramenta Padrão Ouro: Gitleaks

O [Gitleaks](https://github.com/gitleaks/gitleaks) é uma ferramenta open-source escrita em Go, extremamente rápida, feita especificamente para detectar credenciais não criptografadas e segredos em repositórios Git.

#### Passo 1: Instalar o Gitleaks na sua máquina

- **Linux (Debian / Ubuntu / Pop!_OS via script ou binário):**
  ```bash
  # Via script oficial:
  curl -sSfL https://raw.githubusercontent.com/gitleaks/gitleaks/master/scripts/install.sh | sudo sh -s -- -b /usr/local/bin
  
  # Ou se você usa Homebrew no Linux:
  brew install gitleaks
  ```

- **macOS:**
  ```bash
  brew install gitleaks
  ```

- **Windows:**
  ```powershell
  winget install Gitleaks.Gitleaks
  # ou via Scoop:
  scoop install gitleaks
  ```

- **Verificar se está instalado:**
  ```bash
  gitleaks version
  ```

---

#### Passo 2: Configurar o Hook Nativo do Git (`.git/hooks/pre-commit`)

Dentro de qualquer repositório Git local, existe uma pasta oculta chamada `.git/hooks/`. O Git executa qualquer script executável nomeado como `pre-commit` antes de confirmar uma alteração.

Crie ou edite o arquivo `.git/hooks/pre-commit` na raiz do seu projeto:

```bash
cat << 'EOF' > .git/hooks/pre-commit
#!/bin/bash
# ==============================================================================
# Hook de Pre-commit: Verificação de Segredos com Gitleaks
# ==============================================================================

if ! command -v gitleaks &> /dev/null; then
    echo "⚠️  [AVISO]: O utilitário 'gitleaks' não está instalado no seu ambiente."
    echo "Recomendamos instalá-lo para evitar vazamentos acidentais de credenciais."
    exit 0
fi

echo "🔍 Analisando arquivos em staging em busca de credenciais e chaves..."

# Executa o Gitleaks apenas nos arquivos preparados para o commit (--staged)
gitleaks protect --staged -v

RESULT=$?

if [ $RESULT -ne 0 ]; then
    echo ""
    echo "❌ [BLOQUEIO DE SEGURANÇA]: Foram detectadas credenciais no commit!"
    echo "Remova o segredo ou configure um placeholder antes de tentar novamente."
    exit 1
fi

echo "✅ Nenhum segredo detectado. Prosseguindo com o commit."
exit 0
EOF

# Tornar o hook executável:
chmod +x .git/hooks/pre-commit
```

---

#### Passo 3 (Opcional, mas Recomendado): Compartilhar os Hooks com a Equipe

Por padrão, a pasta `.git/hooks/` **não é enviada para o repositório remoto**. Para que todos os desenvolvedores da equipe usem a mesma regra automaticamente, você pode versionar os hooks:

1. Crie uma pasta versionada chamada `.githooks`:
   ```bash
   mkdir -p .githooks
   cp .git/hooks/pre-commit .githooks/pre-commit
   chmod +x .githooks/pre-commit
   ```

2. Configure o Git do projeto para ler os hooks dessa pasta:
   ```bash
   git config core.hooksPath .githooks
   ```

3. Adicione essa instrução ao `README.md` ou automatize em um script de setup inicial (como `npm install` ou `composer install`).

---

### Alternativa: Usando o Framework Multiplataforma `pre-commit`

Se a sua equipe prefere uma solução padronizada via arquivo de configuração YAML:

1. Instale o framework:
   ```bash
   pip install pre-commit
   ```

2. Crie na raiz do repositório o arquivo `.pre-commit-config.yaml`:
   ```yaml
   repos:
     - repo: https://github.com/gitleaks/gitleaks
       rev: v8.18.2
       hooks:
         - id: gitleaks
   ```

3. Instale o hook no repositório:
   ```bash
   pre-commit install
   ```

Toda vez que você rodar `git commit`, o hook do Gitleaks baixará e executará a análise automaticamente.

---

## 4. Linha de Defesa na Nuvem: Auditoria Contínua com GitHub Actions

E se alguém clonar o repositório em uma máquina nova sem os hooks locais configurados? A terceira linha de defesa garante que nenhum Pull Request seja integrado se contiver credenciais.

Crie o arquivo `.github/workflows/secret-scan.yml`:

```yaml
name: "Segurança - Varredura de Credenciais"

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  gitleaks:
    name: "Auditoria com Gitleaks"
    runs-on: ubuntu-latest
    steps:
      - name: Checkout do Código com Histórico Completo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Executar Gitleaks Action Oficial
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Se alguém enviar um Pull Request com chaves expostas, o teste falhará e o botão de *Merge* ficará bloqueado.

---

## 5. Como Lidar com Bloqueios: Falsos Positivos vs. Chaves Reais

Quando o scanner bloquear seu commit ou push, identifique primeiro a natureza do bloqueio:

### Caso A: É uma Chave Real (Erro Crítico)

> [!CAUTION]
> **REGRA DE OURO DA SEGURANÇA:**  
> Uma vez que uma chave real foi commitada em um repositório que já subiu para o GitHub, **considere-a 100% comprometida**. Mesmo que você apague o commit, ela já pode ter sido indexada por robôs.

**O que fazer:**
1. **Revogue a Chave Imediatamente:** Entre no painel do serviço (Stripe, AWS, etc.) e desative/delete a credencial exposta. Crie uma nova.
2. **Mova a Nova Chave para o `.env`:** Nunca insira credenciais no código-fonte.
3. **Limpe o Commit Local:**
   ```bash
   # Remove a chave do arquivo e adiciona ao staging
   git add caminho/do/arquivo.ext

   # Emenda a alteração no último commit
   git commit --amend --no-edit
   ```

---

### Caso B: É um Falso Positivo (Exemplo Didático ou Teste)

Foi exatamente o que aconteceu quando bloqueamos o exemplo didático da Stripe em `docs/documentos_para_a_faculdade/Importancia de proteger credenciais 2.md`.

Existem 3 maneiras de tratar:

#### 1. Usar Placeholders Não-Conformes (Melhor Abordagem)
Em exemplos de código, **nunca use prefixos de chaves vivas seguidos de strings alfanuméricas reais**. 
- ❌ **Evite:** `'sk_live_51Mz000000000000000000000000000'` (mesmo cheia de zeros, o regex identifica como chave de produção).
- ✅ **Use:** `'sk_live_<TOKEN_SECRETO_STRIPE_AQUI>'` ou `'<SUA_CHAVE_PRIVADA>'` (caracteres como `<` e `>` quebram o padrão regex dos scanners e deixam claro que é um modelo).

#### 2. Adicionar Marcação de Isenção no Próprio Código
O Gitleaks permite ignorar uma linha específica usando um comentário inline:

```javascript
// gitleaks:allow
const chaveDeTeste = "sk_live_exemplo_que_precisa_ficar_aqui";
```

#### 3. Configurar um Arquivo `.gitleaksignore`
Crie um arquivo `.gitleaksignore` na raiz do projeto contendo as assinaturas dos falsos positivos conhecidos ou caminhos de arquivos de testes e documentação:

```text
# Ignorar arquivos puramente acadêmicos de documentação
docs/documentos_para_a_faculdade/*
```

---

## 6. Checklist de Higiene para o Desenvolvedor

Para garantir que seu projeto esteja sempre seguro e em conformidade com as melhores práticas da indústria:

- [ ] **`.gitignore` configurado desde o `git init`:** Garanta que `.env`, `.env.local`, `*.pem`, `*.key` e `id_rsa` estejam ignorados.
- [ ] **`.env.example` preenchido apenas com chaves vazias:** Forneça a estrutura de variáveis sem valores sensíveis.
- [ ] **Hook de Pre-Commit ativo:** Gitleaks configurado localmente impedindo `git commit` com segredos.
- [ ] **GitHub Push Protection habilitado:** Bloqueador de segurança remoto ativo nas configurações do repositório.
- [ ] **CI de auditoria contínua:** Workflow de Gitleaks ativo para Pull Requests.
- [ ] **Variáveis de Ambiente na Nuvem:** Use os gerenciadores de segredos nativos da plataforma de hospedagem (GitHub Secrets, AWS Secrets Manager, Vercel/Render Environment Variables).

---

> **Conclusão:** A segurança de um repositório não depende de sorte nem da infalibilidade da memória humana. Com a tríade **Pre-Commit Hook + GitHub Push Protection + CI/CD**, qualquer tentativa de envio de credenciais é interceptada e neutralizada automaticamente.
