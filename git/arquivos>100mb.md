# Upload de arquivos acima de 100MB
É possível fazer o upload, mesmo que o arquivo ultrapasse o limite padrão de 100 MB do GitHub para arquivos individuais.
O GitHub impede commits de arquivos maiores que 100 MB via Git tradicional, mas você pode contornar essa limitação utilizando o Git Large File Storage (Git LFS) ou enviando o arquivo por meio dos Releases (caso ele não precise ficar diretamente na árvore de código).
Abaixo estão as duas principais formas de resolver isso:

Opção 1: Usar o Git LFS (Recomendado para manter no repositório)

O Git LFS substitui arquivos grandes por ponteiros de texto dentro do repositório, armazenando o conteúdo real em um servidor separado do GitHub.
1. Baixe e instale o Git LFS em sua máquina (via git-lfs.com).
2. Abra o terminal na pasta do seu repositório local.
3. Ative o LFS no repositório:bash
git lfs install
Use o código com cuidado.
4. Associe a extensão do seu arquivo RAW (por exemplo, .cr2, .nef, .arw, .dng ou .raw):bash
git lfs track "*.raw"
Use o código com cuidado.
5. Adicione o arquivo de configuração criado pelo LFS:bash
git add .gitattributes
Use o código com cuidado.
6. Faça o commit e push do seu arquivo normalmente:bash
git add imagem.raw
git commit -m "Adiciona imagem RAW usando Git LFS"
git push origin main
Use o código com cuidado.
Nota: O plano gratuito do GitHub inclui 1 GB de armazenamento Git LFS e 1 GB de largura de banda mensal.

Opção 2: Fazer upload nos Releases (Para arquivos de até 2 GB)

Se você precisa da imagem apenas como um anexo ou download para os usuários (e ela não precisa ser rastreada no histórico do código), você pode publicá-la em um "Release".
• Cada arquivo anexado a um Release pode ter até 2 GB.
• Como fazer: Acesse a página inicial do seu repositório no GitHub, clique em Releases no menu lateral direito, crie um novo Release ("Create a new release") e arraste o arquivo RAW diretamente para a área de upload de binários.

