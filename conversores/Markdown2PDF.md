## markdown para pdf

> Para o Debian 13, a forma mais nativa e eficiente de converter arquivos Markdown em PDF em lote é combinando o Pandoc com o Weasyprint (um motor leve e moderno baseado em HTML/CSS) diretamente pelo terminal.
## Passo 1: Instalar as ferramentas
Abra o terminal e instale o Pandoc e o Weasyprint com o gerenciador de pacotes apt:
```bash
sudo apt update && sudo apt install pandoc weasyprint -y
```

## Passo 2: Executar a conversão em lote
Navegue até a pasta onde estão os seus arquivos .md e execute o comando abaixo. Ele vai encontrar todos os arquivos Markdown e criar um PDF correspondente para cada um deles:
```bash
for f in *.md; do pandoc "$f" --pdf-engine=weasyprint -o "${f%.md}.pdf"; done
```

## Alternativa: Unificar tudo em um único PDF
Se em vez de gerar vários PDFs separados você preferir juntar todos os arquivos Markdown em um único arquivo PDF, use este comando:

```bash
pandoc *.md --pdf-engine=weasyprint -o resultado_combinado.pdf
```

Se preferir, me avise se você quer:

* Customizar a aparência do PDF (como margens, fontes e cores) usando uma folha de estilo CSS.
* Uma solução que inclua numeração de páginas ou sumário automático.


