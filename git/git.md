Para adicionar um repositório remoto do GitHub ao seu projeto local, use o comando git remote add origin <URL_DO_REPOSITÓRIO> no seu terminal. [1, 2] 
Aqui está o passo a passo direto para realizar essa configuração:
## 1. Copie a URL no GitHub

   1. Vá até a página do seu repositório no [GitHub](https://github.com/).
   2. Clique no botão verde Code.
   3. Copie a URL gerada (pode ser o link HTTPS ou SSH). [2, 3] 

## 2. Vincule ao seu repositório local
Abra o terminal na pasta do seu projeto local e execute o comando abaixo, substituindo pelo link que você copiou: [1, 4] 
```bash
git remote add origin https://github.com
```

## 3. Verifique se deu certo
Para confirmar se o link foi associado corretamente, digite: 
```bash
git remote -v
```
## Baixar o conteúdo do repositório remoto
```bash
git pull origin main
```
## 4. Envie seus arquivos
Se for a primeira vez que você está enviando arquivos para este repositório remoto, use o comando para fazer o push definindo a branch padrão (geralmente main ou master): [5] 
```bash
git push -u origin main
```
------------------------------
## Dica extra: E se o link estiver errado?
Se você já tiver adicionado um repositório remoto antes e precisar alterar a URL, use o comando de atualização: [2, 6] 

git remote set-url origin <NOVA_URL_DO_REPOSITÓRIO>


