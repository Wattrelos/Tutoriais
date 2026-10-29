# 🏛️ Apache Cassandra vs MySQL: Uma Comparação Técnica Detalhada

_A diferença fundamental é que o Apache Cassandra é um banco de dados NoSQL distribuído (orientado a colunas/largo escopo), focado em altíssima disponibilidade e escrita massiva, enquanto o MySQL é um banco de dados relacional clássico (RDBMS), estruturado em tabelas com forte consistência (ACID) e ideal para transações complexas. Eles operam sob filosofias de engenharia completamente opostas, o que frequentemente gera mal-entendidos quando profissionais migram de um ecossistema para o outro._

## 📊 Comparação Direta: Cassandra vs. MySQL

| Característica | [Apache Cassandra](https://cloud.google.com/discover/apache-cassandra?hl=pt-BR "O que é o Apache Cassandra?	Google Cloud") | MySQL |
|---|---|---|
| **Paradigma** | NoSQL (Família de Colunas / Chave-Valor) | Relacional (Tabelas, Linhas e Colunas fixas) |
| **Arquitetura** | Descentralizada (Peer-to-Peer, sem nó mestre) | Mestre-Escravo (Master-Replica) |
| **Escalabilidade** | Horizontal (Basta adicionar novas máquinas comuns) | Vertical (Exige máquinas mais potentes; sharding manual) |
| **Consistência** | Eventual / Ajustável (Teorema CAP: foca em AP) | Imediata / Forte (Teorema CAP: foca em CA) |
| **Relacionamentos** | Não aceita JOINs ou chaves estrangeiras | Suporte nativo a JOINs e integridade referencial |
| **Operação mais rápida** | Escrita (Append-only no commit log e MemTable) | Leitura (Uso intensivo de índices estruturados) |
- **Operação mais rápida**: Escrita (Append-only no commit log e MemTable)	Leitura (Uso intensivo de índices estruturados)	


## 🚫 Mitos Comuns


1. "Como o Cassandra usa CQL (Cassandra Query Language), ele funciona igual ao SQL"

* **O Mito**: A sintaxe do CQL é parecida com a do SQL (SELECT * FROM tabela WHERE...), dando a falsa impressão de que basta copiar as queries do MySQL.
* **A Realidade**: No Cassandra, você não pode fazer buscas por qualquer coluna livremente. Toda consulta precisa ser desenhada estritamente em cima da chave de partição (Partition Key) e das chaves de ordenação (Clustering Keys). Consultas sem filtros baseados na chave travam o banco (geram o erro Allow Filtering).

2. "O Cassandra substitui o MySQL porque aguenta mais dados"

* **O Mito**: Cassandra é uma "evolução" do relacional para Big Data.
* **A Realidade**: Eles servem a propósitos distintos. Se o seu sistema precisa de transações financeiras complexas (onde o saldo da Conta A deve diminuir exatamente no mesmo milissegundo em que a Conta B aumenta), o MySQL é infinitamente superior devido à conformidade ACID estrita. O Cassandra sacrifica essa consistência imediata em prol do tempo de atividade global.

3. "Duplicar dados no Cassandra é um erro de arquitetura"

* **O Mito**: A normalização (regrade ouro do MySQL para evitar duplicidade) deve ser aplicada no Cassandra.
* **A Realidade**: No Cassandra, a desnormalização é obrigatória. Se você precisa exibir os dados de três formas diferentes na sua tela, você criará três tabelas diferentes com os mesmos dados, organizados por chaves de busca distintas. Armazenamento é barato; processamento distribuído não.

## ⚠️ Vícios de Desenvolvedores Relacionais (Anti-patterns)

Quem passa anos modelando para o MySQL costuma cometer erros graves ao projetar para o Cassandra:

* **Modelar baseado em entidades e não em queries**: No MySQL, você olha para o mundo real, cria tabelas ("Usuário", "Pedido", "Produto") e depois pensa em como juntá-las com JOINs. No Cassandra, você faz o oposto: primeiro descobre quais telas/queries a aplicação vai precisar e desenha tabelas específicas para responder a cada uma daquelas perguntas com uma única busca.
* **Tentar simular JOINs na aplicação**: Ao perceber que o Cassandra não junta tabelas, o desenvolvedor relacional faz cinco requisições seguidas via código para cruzar dados. Isso destrói a performance distribuída e anula o propósito do NoSQL.
* **Uso excessivo de atualizações (UPDATE) em lote**: No MySQL, modificar dados é trivial. No Cassandra, atualizações frequentes geram um acúmulo de registros deletados ou sobrescritos chamados Tombstones. Se houver muitos tombstones, a performance de leitura degrada drasticamente até derrubar o nó.

Para nos aprofundarmos no cenário ideal para o seu projeto, me conte:
* **Qual é o volume de dados e de acessos por segundo esperado?**
* **A sua aplicação faz mais operações de escrita rápida (logs, IoT, chats) ou de leituras complexas (relatórios, filtros)?**
* **Você precisa de garantias transacionais estritas (ACID)?**

## 🏛️ Onde o Cassandra Domina

_O Apache Cassandra domina exatamente em cenários de gravação massiva, contínua e em tempo real, onde os dados são majoritariamente imutáveis. Ele foi projetado para nunca parar de receber dados, mesmo que partes da infraestrutura global falhem._
_Abaixo estão os cenários onde ele supera quase qualquer banco relacional:_

### 🌟 Em que cenário o Cassandra triunfa

#### 1. Registros Imutáveis e Time-Series (Séries Temporais)

_Dados que são gerados sequencialmente e raramente sofrem alteração (UPDATE) ou deleção (DELETE) são o cenário perfeito para o Cassandra._

* **Por que domina**: A arquitetura dele é append-only (apenas adiciona dados no final). Escrever no Cassandra é basicamente salvar em memória (MemTable) e registrar em um log sequencial no disco (CommitLog). Não há busca por espaço livre em disco nem reorganização de índices na hora da escrita.
* **Exemplos práticos**: Logs de auditoria, históricos de geolocalização de frotas, dados de sensores IoT (telemetria de temperatura, pressão, etc.) e métricas de servidores.

#### 2. Histórico de Transações (Mas não a transação em si)

Existe uma linha fina aqui:

* **Onde ele NÃO deve ser implementado**: No fechamento do carrinho de compras ou na transferência bancária onde o saldo precisa ser atualizado instantaneamente na tabela do usuário (isso é o forte do MySQL).
* **Onde ele domina**: No extrato. Salvar cada linha do histórico de compras de milhões de clientes ao longo de anos. Uma vez que o pagamento foi processado, aquela linha do histórico se torna um registro imutável que raramente será alterado, apenas consultado pelo usuário através do seu ID de cliente.

#### 3. Sistemas de Mensageria e Chats de Larga Escala

* **Por que domina**: Chats geram bilhões de pequenas mensagens por dia. O Cassandra permite estruturar os dados de forma que a Partition Key seja o ID_do_Chat e a Clustering Key seja o Timestamp_da_Mensagem.
* **Resultado**: Buscar o histórico de conversas de um grupo específico é uma operação ultra veloz, pois todas as mensagens daquele chat ficam fisicamente juntas no mesmo nó e ordenadas por tempo.

#### 4. Carrinhos de Compras Globais (Modelo Amazon/Netflix)

* **Por que domina**: Ele prioriza a Disponibilidade em detrimento da consistência imediata (Teorema CAP). Para grandes e-commerces, é melhor o cliente conseguir adicionar um item ao carrinho (mesmo que por alguns milissegundos outro servidor global não saiba disso ainda) do que o site cair ou travar a tela exibindo um erro de conexão com o banco.

#### 📊 Resumo do comportamento do Cassandra

* **Se o dado é estático ou imutável**: Desempenho fantástico e escalabilidade linear infinita.
* **Se o dado muda o tempo todo** (Ex: Contador de likes que atualiza a mesma linha de um post 10.000 vezes por segundo): Péssima escolha. O Cassandra criará milhares de versões do mesmo dado em disco e sobrecarregará o processo de Compaction, degradando o banco.

