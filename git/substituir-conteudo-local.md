# Substituir conteúdo local pelo do servidor

Se você deseja descartar completamente todas as suas alterações locais e deixar o seu repositório exatamente igual ao que está no GitHub (remoto), você deve usar o comando git reset.
Antes de executar, tenha certeza absoluta, pois esse processo não pode ser desfeito e você perderá qualquer código local que não foi enviado ao servidor.
Execute os seguintes comandos no seu terminal:

## 1. Baixa as atualizações mais recentes do servidor (sem mesclar ainda)

```bash
git fetch origin
```
# 2. Força o seu ramo local a ficar idêntico ao ramo do servidor
```bash
git reset --hard origin/main
```
(Substitua main pelo nome do seu branch atual, como master ou dev, se for o caso).
------------------------------
## Passo Adicional (Opcional)
O comando acima substitui os arquivos que já existiam, mas se você tiver arquivos novos criados localmente que nunca foram enviados ao GitHub, eles continuarão na pasta como arquivos "não rastreados" (untracked).
Para apagar esses arquivos novos e deixar a pasta 100% limpa, execute:

# Remove arquivos e pastas novos que não existem no repositório remoto
```bash
git clean -fd
```

