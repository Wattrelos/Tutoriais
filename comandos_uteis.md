O comando correto para copiar uma pasta inteira no Debian 13 (incluindo subpastas e arquivos ocultos) sem sobrescrever os arquivos já existentes no destino é o cp com as opções -a (arquivamento/preservação) e -n (não sobrescrever).
Execute o seguinte comando no terminal:

```bash
cp -an /caminho/da/pasta_origem/. /caminho/da/pasta_destino/
```

## O que cada parte do comando faz:

* -a (archive): Copia de forma recursiva (tudo dentro da pasta), preservando links simbólicos, permissões, donos e atributos dos arquivos. Ele garante que os arquivos ocultos (que começam com .) sejam copiados.
* -n (no-clobber): Impede que arquivos no destino sejam sobrescritas. Se um arquivo já existir, ele será pulado silenciosamente.
* /. (ponto após a origem): É o segredo para copiar o conteúdo da pasta de origem (incluindo os itens ocultos) diretamente para dentro da pasta de destino, em vez de copiar a pasta mãe em si.

Se você precisar ver exatamente quais arquivos estão sendo copiados na tela, me avise para eu te mostrar como ativar o modo detalhado ou como criar um log dessa cópia.

