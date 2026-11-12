# Tutorial: PlantUML-AstahUML

> Para transferir um diagrama feito em PlantUML para o Astah UML, utilize o Plugin oficial do PlantUML para Astah, que converte o código textual diretamente em modelos editáveis na ferramenta.

## 1. Requisitos e Instalação do Plugin

O plugin funciona nas versões 10.0 ou superiores do Astah UML e do Astah Professional. 
* Baixe o arquivo do plugin. Acesse a página do Astah PlantUML Plugin ou o repositório oficial no GitHub da ChangeVision para obter o arquivo .jar. 
* Instale no Astah. Arraste o arquivo .jar baixado diretamente para dentro da janela aberta do Astah UML, confirme a instalação na caixa de diálogo e reinicie o programa. 
* Verifique o Graphviz. No Windows o componente já vem embutido, mas usuários de macOS ou Linux precisam instalar o Graphviz manualmente para renderizar diagramas de classes e de máquinas de estado. 

## 2. Como Importar o Código PlantUML

Com o plugin ativo, a conversão é feita dentro da própria interface do Astah: 
* Abra a aba do plugin. No painel inferior do Astah, localize e selecione a aba [PlantUML View].
* Insira o código. Cole o seu script PlantUML no editor de texto da aba e aguarde a pré-visualização validar a sintaxe automaticamente.
* Converta para o Astah. Clique no botão [toAstah △] (ou opção equivalente de importação para o Astah) na barra superior da aba.
* Edite o modelo gerado. O Astah criará um novo diagrama (ou atualizará um diagrama compatível já aberto) com classes, atributos, métodos e relacionamentos nativos prontos para edição. 

## 3. Diagramas Suportados e Alternativas

* Tipos compatíveis. O plugin suporta a importação de diagramas de Classes, Sequência, Casos de Uso, Atividades, Máquina de Estados, Componentes e Estrutura Composta. 
* Alternativa via Inteligência Artificial. Caso você utilize o recurso de Modelagem com IA no Astah, também é possível colar trechos em PlantUML no AI Chat Copilot ou via Servidor MCP para gerar os diagramas diretamente no projeto. 
