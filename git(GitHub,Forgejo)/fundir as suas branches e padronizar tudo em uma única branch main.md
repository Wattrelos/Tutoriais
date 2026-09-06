# Padronizar tudo em uma única branch main

_Para fundir as suas branches e padronizar tudo em uma única branch main, o processo mais seguro e limpo é renomear a sua branch local master para main, puxar o histórico da main remota (se houver), unificar os históricos e depois atualizar o GitHub._

Imagine a seguinte situação: Você tem duas branches: uma branch master local (que você acabou de criar) e uma branch main remota (que já existe no GitHub). O que você quer fazer é padronizar tudo em uma única branch main. Como fazer?

Aqui está o passo a passo com os comandos que você deve executar no seu terminal:

## 1. Renomear a branch local de master para main

Se você estiver atualmente na branch master, mude o nome dela localmente para main com o comando:

```bash
git branch -m master main
```

## 2. Unificar os históricos (Se a main remota já tiver arquivos)
Para garantir que você não vai perder nada que já esteja na main do GitHub, baixe as atualizações e junte os históricos:gi

```bash
git fetch origin
git checkout main
git merge origin/main --allow-unrelated-histories
```

Nota: O --allow-unrelated-histories é usado caso a master local e a main remota tenham sido criadas de formas diferentes e não compartilhem o mesmo commit inicial.
## 3. Enviar a nova main para o GitHub e definir como padrão
Envie a sua branch main local revisada para o repositório remoto:

```bash
git push -u origin main
```

## 4. Limpar a branch master antiga (Opcional)
Se você não precisa mais da branch master que ficou no GitHub e quer manter apenas a main, você pode deletá-la remotamente:

```bash
git push origin --delete master
```

------------------------------
## O que fazer caso o GitHub bloqueie a exclusão da master?
Se você receber um erro ao tentar deletar a master no último passo, significa que ela está configurada como a branch padrão (default) no site do GitHub. Para resolver:

   1. Acesse o seu repositório no GitHub.
   2. Vá em Settings (Configurações) > Branches.
   3. Na seção Default branch, clique nas setas alternadas e mude a branch padrão de master para main.
   4. Clique em Update e confirme. Depois disso, você poderá rodar o comando de deletar a master normalmente.


