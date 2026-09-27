# 📂 Ambiente Separado para Cada Projeto

Para isolar completamente o chat, os pensamentos e o contexto do Model Context Protocol (MCP) entre projetos diferentes, a melhor estratégia é descentralizar a configuração do MCP, tirando as diretrizes do escopo global da IDE e trazendo-as para o escopo local de cada repositório.
Abaixo está o passo a passo para blindar seus projetos no Debian 13, garantindo que cada PC (ou pasta) use apenas o seu respectivo "brain".
------------------------------
## 1. Criar Configurações Locais por Projeto
Em vez de usar o arquivo global do MCP, crie um arquivo de configuração específico dentro da raiz de cada projeto.
No Antigravity IDE / VS Code, crie a pasta oculta e o arquivo de configuração local:

# Na raiz do Projeto A (no PC 1)
mkdir -p .vscode
touch .vscode/mcp.json
# Na raiz do Projeto B (no PC 2)
mkdir -p .vscode
touch .vscode/mcp.json

## 2. Isolar as Ferramentas e "Brains" no mcp.json
Dentro de cada .vscode/mcp.json, defina apenas as ferramentas, servidores MCP e caminhos que pertencem àquele projeto específico.
Exemplo para o Projeto A (E-commerce / PHP):

{
  "mcpServers": {
    "php-local-server": {
      "command": "node",
      "args": ["/caminho/para/o/mcp-server-php/index.js"],
      "env": {
        "PROJECT_ROOT": "${workspaceFolder}",
        "DB_CONNECTION": "mysql_local"
      }
    }
  }
}

Exemplo para o Projeto B (Outro contexto / Performance):

{
  "mcpServers": {
    "rust-performance-analyzer": {
      "command": "cargo",
      "args": ["run", "--bin", "mcp-analyzer"],
      "env": {
        "PROJECT_ROOT": "${workspaceFolder}"
      }
    }
  }
}   

O Antigravity prioriza o .vscode/mcp.json do workspace ativo, ignorando o do outro projeto.
## 3. Blindar os Perfis da IDE (Isolamento de Chat e Histórico)
Para garantir que as abas de chat, histórico de prompts e indexação de arquivos (embeddings/pensamentos) não se misturem, utilize os Perfis (Profiles) nativos da IDE.

   1. No canto inferior esquerdo, clique no ícone de Engrenagem ⚙️.
   2. Selecione Perfis (Profiles) > Criar Perfil...
   3. Crie um perfil chamado Projeto-Ecommerce e outro chamado Projeto-Secundario.
   4. Associe cada pasta de projeto ao seu respectivo perfil.

💡 O que isso faz: Cada perfil mantém um banco de dados de cache, extensões ativas, histórico do chat de IA e estado de janelas totalmente separado. Mesmo que você use a mesma conta da assinatura, a IA não lerá o cache do outro perfil.

## 4. Isolar Variáveis de Ambiente no Debian 13
Nunca misture credenciais ou chaves de IA nos arquivos globais do sistema como o /etc/environment ou ~/.bashrc. Utilize arquivos .env locais em cada máquina.

   1. Certifique-se de que o .env está incluído no seu .gitignore global ou local para não vazar dados.
   2. Adicione um arquivo .env.example na raiz para o assistente saber quais chaves ele precisa ler naquele PC, sem misturar os tokens de produção ou desenvolvimento de cada projeto.

------------------------------
## Resumo do Fluxo de Trabalho por PC
Ao abrir a IDE no PC 1, ela carregará o Perfil A, lerá o .vscode/mcp.json da pasta e usará o banco de dados local de pensamentos do Projeto A. Ao ligar o PC 2, o ambiente estará 100% limpo com o contexto do Projeto B.

