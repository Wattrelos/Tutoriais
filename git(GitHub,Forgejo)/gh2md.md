# Baixar todos os issues do GitHub e convertê-los em arquivos Markdown (.md).
_Como o GitHub não oferece essa opção de forma nativa (ele apenas exporta para formatos como CSV/TSV por padrão), você precisa utilizar ferramentas de terceiros, extensões ou scripts de automação._
Aqui estão as melhores maneiras de fazer isso:
------------------------------

## Ferramentas de Linha de Comando (CLI) e Web
Existem utilitários de código aberto criados pela comunidade para automatizar essa extração.

* 
* [issue2md](https://github.com/bigwhite/issue2md): É uma ferramenta CLI (linha de comando) que converte issues, discussões ou pull requests do GitHub para o formato Markdown. Ela permite incluir reações, links de usuários e salvar tudo em arquivos locais de forma estruturada.

------------------------------
## Scripts Personalizados (Python / Node.js)
Se você precisa de uma estrutura de arquivos muito específica, a melhor saída é escrever um script simples utilizando a [API REST do GitHub](https://docs.github.com/pt/rest).
Um fluxo básico em Python, por exemplo, consiste em:

   1. Fazer uma requisição para o endpoint GET /repos/{owner}/{repo}/issues.
   2. Percorrer a lista de dados retornada (título, corpo, comentários).
   3. Utilizar uma biblioteca para salvar strings de texto diretamente em arquivos com a extensão .md.

-----------------------------------
## Exemplo usando Python e a biblioteca `PyGithub`
Se você não quiser usar ferramentas de terceiros e preferir ter controle total sobre o processo, pode criar um script simples em Python para extrair as issues usando a biblioteca PyGithub.

### 1. Instalação
Primeiro, instale a biblioteca necessária:

```bash
pip install PyGithub
```

### 2. Código do Script
Crie um arquivo Python (ex: `export_issues.py`) com o seguinte conteúdo. Lembre-se de substituir `seu_token_aqui`, `nome_do_usuario` e `nome_do_repositorio` pelos valores corretos.

```python
import os
from github import Github
from datetime import datetime

# --- CONFIGURAÇÕES ---
# Substitua pelo seu Token do GitHub (gerado em https://github.com/settings/tokens)
TOKEN = "seu_token_aqui"
# Substitua pelo nome de usuário e repositório que deseja baixar
OWNER = "nome_do_usuario"
REPO = "nome_do_repositorio"
# Diretório onde os arquivos .md serão salvos
OUTPUT_DIR = "issues_export"
# --------------------

def main():
    # Cria o diretório de saída se não existir
    if not os.path.exists(OUTPUT_DIR):
        os.makedirs(OUTPUT_DIR)

    # Autenticação com a API
    print("Conectando ao GitHub...")
    g = Github(TOKEN)
    repo = g.get_user(OWNER).get_repo(REPO)
    
    print(f"Baixando Issues de {OWNER}/{REPO}...")
    issues = repo.get_issues(state='all')

    count = 0
    for issue in issues:
        count += 1
        filename = os.path.join(OUTPUT_DIR, f"issue_{issue.number}.md")
        
        with open(filename, 'w', encoding='utf-8') as f:
            # Cabeçalho Markdown
            f.write(f"# Issue {issue.number}: {issue.title}\n")
            f.write(f"**Status:** {issue.state}\n")
            f.write(f"**Criado em:** {issue.created_at}\n")
            f.write(f"**Autor:** {issue.user.login}\n\n")
            
            # Corpo da Issue
            f.write("---\n\n")
            if issue.body:
                f.write(issue.body)
            else:
                f.write("_Sem descrição._\n")
                
            # Comentários (se houver)
            if issue.comments > 0:
                f.write("\n\n---\n\n### Comentários\n\n")
                for comment in issue.get_comments():
                    f.write(f"#### Comentário de {comment.user.login} em {comment.created_at}\n")
                    f.write(f"\n{comment.body}\n\n")
                    
    print(f"Concluído! Foram exportados {count} issues para a pasta '{OUTPUT_DIR}'.")

if __name__ == "__main__":
    main()
```

### 3. Execução
Salve o arquivo e rode-o:

```bash
python export_issues.py
```

Isso criará uma pasta `issues_export` contendo um arquivo `.md` para cada issue do repositório especificado, completo com títulos, descrições e comentários.
