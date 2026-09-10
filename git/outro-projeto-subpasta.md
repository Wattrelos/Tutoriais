# git repo dentro de outro

## Cenário:
Um repositório Git dentro de uma subpasta de outro repositório, desde que essa subpasta esteja listada no arquivo .gitignore do repositório pai.

Quando você adiciona o nome da subpasta ao .gitignore principal, o Git do repositório pai passa a ignorar completamente tudo o que acontece dentro dessa subpasta, incluindo a pasta oculta .git da subpasta. Dessa forma, os dois repositórios funcionam de maneira isolada e independente, sem que um interfira nos commits ou no histórico do outro.

## Como configurar passo a passo

1. No repositório pai (pasta principal):
Abra ou crie o arquivo .gitignore na raiz e adicione o nome da subpasta (substitua minha-subpasta pelo nome real):

``` .gitignore
minha-subpasta/
```

2. Crie o repositório filho (na subpasta):
Acesse a subpasta pelo terminal e inicialize o novo repositório local:

``` bash
cd minha-subpasta
git init
```

A partir desse momento, você pode rodar comandos como git status, git add e git commit dentro de cada pasta separadamente, e cada uma responderá apenas ao seu respectivo repositório.
------------------------------
## Alternativa nativa: Git Submodules
Embora a estratégia do .gitignore funcione bem para projetos estritamente isolados, o Git possui uma ferramenta oficial chamada Submódulos (Submodules). 
Ela é ideal se o repositório pai precisar manter uma referência ou conexão com o repositório filho (por exemplo, uma biblioteca ou um submódulo de código compartilhado). Para usá-la, em vez de ignorar a pasta, você executa o seguinte comando na raiz do repositório pai:

git submodule add <URL-do-repositorio-filho> minha-subpasta

Para recomendar a melhor estrutura para o seu caso, me conte:


* O repositório pai precisa saber da existência do projeto filho ou eles são totalmente distintos?
* Você pretende subir ambos para o GitHub em páginas de repositórios diferentes?


