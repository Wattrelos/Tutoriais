# CQRS + Event-Driven Architecture

> É ótimo abordar esses conceitos agora, pois eles mudam completamente a forma como pensamos sobre sistemas e bancos de dados quando o projeto começa a crescer. Vamos simplificar esses dois termos usando analogias do mundo real para que você entenda como eles se conectam perfeitamente ao seu "livro-ata".

## 1. O que é CQRS? (Separação de Responsabilidades)

A sigla CQRS significa Command Query Responsibility Segregation (Segregação de Responsabilidade de Comando e Consulta).
* **A ideia central**: Em vez de usar o mesmo modelo e o mesmo banco de dados para salvar dados e buscar dados, você divide o sistema em dois caminhos totalmente independentes.
💡 **A analogia do Restaurante**:
Pense em um restaurante movimentado:
* **O Comando (Escrita)**: É a Cozinha. A única preocupação deles é receber pedidos novos e preparar a comida o mais rápido possível (regras rígidas, sequência exata, append-only). Eles não atendem clientes perguntando o preço do prato.
* **A Consulta (Leitura)**: É o Cardápio na mesa. O cliente não olha para a cozinha para saber o que tem; ele olha para um papel impresso (um dado estático, mastigado, fácil de ler).

No seu projeto:
* **A Cozinha** é o seu banco "livro-ata" (PostgreSQL ou ScyllaDB). Ele só recebe os INSERTs de pedidos e logs.
* **O Cardápio** é um banco de leitura rápido (como o OpenSearch ou Redis). Ele tem o resumo pronto em formato JSON para o cliente ver na tela instantaneamente.

## 2. O que é a Arquitetura Baseada em Eventos? (Event-Driven)

Se você separou a Escrita da Leitura, surge um problema: Como o "Cardápio" se atualiza quando a "Cozinha" termina um prato? É aí que entram os Eventos.
Em um sistema tradicional, um componente avisa o outro diretamente (chamada síncrona). Na arquitetura baseada em eventos, o sistema reage a acontecimentos passados ("coisas que já aconteceram e não mudam").
💡 **A analogia do Alto-falante do Aeroporto**:
Em vez de a companhia aérea ligar individualmente para cada passageiro avisando que o voo mudou de portão, ela simplesmente anuncia no sistema de som: "Voo 123 alterado para o Portão 5".
Quem tiver interesse nessa informação (o passageiro, a equipe de limpeza do portão, a segurança) escuta o anúncio e toma uma ação.

No seu código, funciona assim:
1. O cliente finaliza um pedido.
2. A sua aplicação grava no "livro-ata" e imediatamente publica uma mensagem em um "alto-falante" (gerenciadores de fila como RabbitMQ ou Apache Kafka, que rodam no Linux):
* **Evento**: "Pedido #772 foi concluído com sucesso!"
3. O seu banco de leitura está "ouvindo" esse alto-falante. Ele captura a mensagem, monta o JSON desnormalizado de Cliente ➔ Pedido ➔ Produto e salva na tabela de busca rápida.

## Por que isso resolve o seu problema de "Arquivo Morto"?

Utilizando esses dois padrões combinados, você ganha uma flexibilidade incrível para gerenciar o ciclo de vida dos seus dados:

* **O banco de leitura fica leve**: Nele você mantém apenas os dados dos últimos 3 meses (o período de trocas e garantias). Como ele roda em ferramentas focadas em busca rápida (como OpenSearch), os filtros por cliente e produto serão instantâneos. Passados 3 meses, você pode apagar o dado desse banco sem medo.
* **O livro-ata fica protegido**: Os dados de 1 ou 2 anos atrás continuam salvos de forma permanente, imutável e sequencial no seu banco principal (ou movidos para discos rígidos baratos). Se um cliente precisar de uma auditoria antiga, você busca diretamente no "livro-ata" (uma consulta que pode demorar um pouco mais, mas que acontece raramente).

