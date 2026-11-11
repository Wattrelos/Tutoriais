# Criar diagramas de arquivos e pastas navegáveis para o github

> Para ter fonte monoespaçada (largura fixa) e links clicáveis ao mesmo tempo no GitHub, você deve substituir as três crases pela tag HTML `<pre>` (texto pré-formatado) combinada com `<a>` ou `<code>`. O bloco de código tradicional com crases trata tudo como texto puro, mas a tag `<pre>` preserva todos os espaços, usa fonte monoespaçada e permite tags HTML internas.

1. Usando `<pre>` com `<code>` e `<a>` (Mais fiel ao visual de bloco de código)

O GitHub permite o uso das tags `<pre>`, `<code>` e `<a href="...">`. Essa combinação cria exatamente o mesmo fundo cinza e a mesma fonte fixa do bloco de três crases, mas torna os nomes clicáveis.
Como escrever no seu README.md:
```html
<pre><code>meu-projeto/
├── <a href="/src">src/</a>
│   ├── <a href="/src/components">components/</a>
│   │   ├── <a href="/src/components/Button.js">Button.js</a>
│   │   └── <a href="/src/components/Header.js">Header.js</a>
│   └── <a href="/src/index.js">index.js</a>
└── <a href="/README.md">README.md</a>
</code></pre>
```

2. Usando apenas `<pre>` (Sem fundo cinza)

Se você preferir que o diagrama fique direto no fundo branco/escuro da página, sem a caixa cinza ao redor, basta remover a tag `<code>`. A largura fixa dos caracteres continua garantida.
```html
<pre>
meu-projeto/
├── <a href="src">src/</a>
│   └── <a href="/src/index.js">index.js</a>
└── <a href="/README.md">README.md</a>
</pre>
```

## ⚠️ Regras importantes para funcionar no GitHub

* Use HTML para os links: Dentro da tag `<pre>`, o GitHub não processa a sintaxe Markdown `[texto](url)`. Você deve obrigatoriamente usar `<a href="caminho">texto</a>`.
* Evite linhas em branco dentro do `<pre>`: Deixar uma linha totalmente vazia dentro de um bloco HTML pode fazer o interpretador do GitHub fechar a tag `<pre>` antes da hora e quebrar o restante do diagrama. Se precisar de uma linha vazia para espaçamento, coloque pelo menos um espaço em branco nela.
* Caminhos relativos funcionam: Você pode apontar o `href` direto para os arquivos do seu repositório usando `./pasta/arquivo.ext` ou para links externos com `https://....`.
Você gostaria de adicionar descrições ou notas de rastreabilidade alinhadas ao lado de cada arquivo (como colunas de texto) ou precisa automatizar a geração desse HTML a partir das pastas do seu projeto?
