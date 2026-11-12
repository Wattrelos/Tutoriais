# Instalação do MARP

> Para instalar o Marp (Markdown Presentation Ecosystem) no Debian 13 (Trixie), a forma mais direta é utilizar o Marp CLI via gerenciador de pacotes npm ou através do binário independente (standalone). Caso utilize o Visual Studio Code, também é possível instalar a extensão oficial para pré-visualização em tempo real.
> Para entender na prática como funciona o fluxo de criação de slides e as ferramentas do ecossistema Marp, confira estes vídeos:

Método 1. Instalação via npm (Recomendado)

Este método exige o Node.js (versão 18 ou superior) instalado no sistema.
1. Atualize os repositórios e instale o Node.js e o npm. Execute no terminal:
```bash
sudo apt update
sudo apt install nodejs npm -y
```
2. Instale o Marp CLI globalmente. Utilize o comando abaixo para deixar o comando marp disponível em todo o sistema:
```bash
sudo npm install -g @marp-team/marp-cli
```
3. Verifique a instalação. Confirme se a ferramenta responde corretamente:

```bash
marp --version
```

Método 2. Binário Independente (Sem Node.js)

Caso prefira não instalar o Node.js no Debian 13, você pode baixar o executável compilado da página oficial do @marp-team/marp-cli no GitHub.
1. Baixe e extraia a versão para Linux. Execute os comandos abaixo no terminal:
```bash
wget https://github.com
tar -xzf marp-cli-linux.tar.gz
```
2. Mova o binário para o diretório do sistema. Isso permite executar o comando de qualquer pasta:
```bash
sudo mv marp /usr/local/bin/
rm marp-cli-linux.tar.gz
```

Dependência para Exportação (PDF, PPTX e Imagens)

Para converter arquivos Markdown apenas para HTML, nenhuma dependência extra é necessária. Porém, para exportar apresentações em PDF, PowerPoint (PPTX) ou imagens, o Marp exige um navegador compatível instalado no sistema (como Firefox, Chromium ou Google Chrome).
• Instalação rápida de navegador no Debian. Caso não tenha um navegador gráfico instalado, instale o Chromium pelo repositório oficial:
```bash
sudo apt install chromium -y
```

Uso no Visual Studio Code (Opcional)

Se você edita seus arquivos no VS Code, não precisa usar o terminal para gerar os slides.
• Extensão oficial. Abra o editor, acesse a aba de extensões (Ctrl+Shift+X), busque por Marp for VS Code e clique em instalar. A extensão permite visualizar os slides lado a lado enquanto você digita e exportar diretamente pela interface gráfica.

Teste Rápido

Crie um arquivo chamado apresentacao.md com o seguinte conteúdo:
markdown
---
marp: true
theme: default
paginate: true
---

# Primeiro Slide
Apresentação criada com Marp no Debian 13.

---

## Segundo Slide
* Item 1
* Item 2
```
Para converter o arquivo para PDF pelo terminal, execute:

```bash
marp apresentacao.md --pdf
```
Você pretende usar o Marp principalmente pelo terminal para automação ou integrado a um editor como o VS Code ou Obsidian? Posso mostrar como configurar temas CSS personalizados para seus slides.


Para visualizar suas apresentações do Marp em tempo real enquanto você edita o arquivo Markdown, a forma mais rápida é usar o modo de pré-visualização (--preview) ou o modo servidor (--server) pelo terminal, além da opção integrada no VS Code.

1. Visualização em Tempo Real pelo Terminal (Marp CLI)

O Marp CLI possui um servidor embutido que abre o seu navegador padrão e atualiza os slides automaticamente toda vez que você salva o arquivo .md.
• Modo Preview (Arquivo único): Abre diretamente a apresentação no navegador com recarregamento automático (live reload).
```bash
marp -p apresentacao.md
```
(A flag -p é um atalho para --preview.)
• Modo Server (Pasta de arquivos): Inicia um servidor local na porta 8080 que lista todos os arquivos Markdown do diretório atual. Ideal se você trabalha com várias apresentações ao mesmo tempo.
```bash
marp -s .
```
(Depois de rodar o comando, acesse http://localhost:8080 no navegador.)
• Modo Watch (Conversão contínua sem abrir navegador): Se você já abriu o arquivo .html gerado no navegador e quer apenas que ele seja atualizado a cada salvamento:
```bash
marp -w apresentacao.md
```

2. Visualização Direta no Visual Studio Code

Se você utiliza o VS Code com a extensão Marp for VS Code, a visualização acontece lado a lado dentro do próprio editor, sem precisar rodar comandos no terminal.
1. Abra o arquivo Markdown. Certifique-se de que o cabeçalho marp: true está nas primeiras linhas do documento.
2. Ative a pré-visualização. Clique no ícone de Abrir Visualização ao Lado (uma lupa sobre uma página no canto superior direito do editor) ou pressione o atalho Ctrl + K V.
3. Navegue pelos slides. A janela lateral exibirá os slides formatados em tempo real conforme você digita.

Dicas Úteis durante a Visualização

• Modo Apresentação no Navegador: Ao visualizar pelo navegador (via marp -p ou marp -s), passe o mouse sobre a parte inferior da tela para acessar a barra de controles do Marp, que permite entrar em tela cheia, ativar a visão do apresentador (com notas e cronômetro) ou navegar entre os slides.
• Arquivos locais e imagens: Se a sua apresentação carregar imagens locais ou fontes externas que não aparecem no preview, adicione a flag --allow-local-files ao comando:
```bash
marp -p --allow-local-files apresentacao.md
```
