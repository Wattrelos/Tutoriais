# 📗 Apache Cassandra: Boas Práticas de Modelagem e Uso

> **A Analogia do Livro de Ata:**  
> A essência arquitetural do Cassandra é baseada em **registros imutáveis e sequenciais** — exatamente como um **Livro de Ata** oficial. Cada entrada é inserida no final, datada, e nunca é rasurada ou arrancada. Se algo precisa ser corrigido, registra-se uma nova entrada referenciando a anterior. Essa é a melhor metáfora para compreender o comportamento ideal deste banco de dados.

O Cassandra foi projetado nativamente para cenários onde você **insere dados continuamente** (*Write-Heavy*) e os **lê rapidamente por chave de acesso** (*Key-Based Reads*), evitando ao máximo atualizações (`UPDATE`) e exclusões (`DELETE`).

Existem dois motivos técnicos fundamentais que explicam por que tratar o Cassandra como uma **Ata** é o cenário perfeito de uso:

---

## 1. Como o Cassandra Funciona Internamente

### 1.1 Escrita Ultrarrápida: O Motor LSM-Tree (Log-Structured Merge-Tree)

O Cassandra **não procura** um espaço livre no disco para gravar ou atualizar uma linha. O caminho de escrita é radicalmente diferente dos bancos relacionais:

```
Caminho de Escrita no Cassandra:

[Aplicação] ── INSERT/UPDATE ──▸ [Nó Cassandra]
                                        │
                    ┌───────────────────┤
                    ▼                   ▼
            ┌─────────────┐    ┌───────────────┐
            │ Commit Log  │    │   MemTable    │
            │ (WAL Disco) │    │  (Memória RAM)│
            └─────────────┘    └───────┬───────┘
                                       │ (Flush periódico)
                                       ▼
                              ┌─────────────────┐
                              │    SSTable       │
                              │ (Arquivo Imutável│
                              │   no Disco)      │
                              └─────────────────┘
                                       │ (Compactação em background)
                                       ▼
                              ┌─────────────────┐
                              │ SSTable Merged   │
                              │ (Versões antigas │
                              │  são descartadas)│
                              └─────────────────┘
```

1. **Commit Log (Write-Ahead Log):** O dado é primeiro gravado sequencialmente em um arquivo de log no disco para garantir durabilidade em caso de falha de energia.
2. **MemTable (Memória RAM):** Simultaneamente, o dado é inserido em uma estrutura ordenada na memória. É aqui que as leituras mais recentes são atendidas com velocidade de nanossegundos.
3. **SSTable (Sorted String Table):** Quando a MemTable atinge um limite de tamanho, todo o seu conteúdo é despejado (*flushed*) no disco como um **arquivo imutável** — a SSTable. Uma vez gravada, a SSTable **nunca é modificada**.
4. **Compactação (*Compaction*):** Em segundo plano, o Cassandra mescla (*merge*) múltiplas SSTables, descartando versões obsoletas de registros e consolidando os dados em arquivos maiores e mais eficientes.

> [!TIP]
> **A consequência prática:** Se você executar um `UPDATE` sobre uma linha existente, o Cassandra **não localiza e sobrescreve** o registro antigo. Ele simplesmente grava uma **nova versão** com um *timestamp* mais recente em uma nova SSTable. Quem lê sempre recebe a versão com o *timestamp* mais alto. Atualizações frequentes geram acúmulo de versões antigas no disco até que a compactação as elimine — consumindo CPU e I/O desnecessariamente.

### 1.2 O Terrível Problema dos Tombstones (Exclusão de Dados)

Quando você deleta um dado no Cassandra (`DELETE`), ele **não apaga** o registro imediatamente. Como as SSTables são **imutáveis**, apagar algo exigiria reescrever arquivos gigantescos a todo momento.

Em vez de apagar, o Cassandra grava um marcador especial chamado **Tombstone** (Lápide):

```
SSTable Original:                     SSTable após DELETE:
┌──────────────────────────┐          ┌──────────────────────────┐
│ user_123 │ Pedido #500   │          │ user_123 │ Pedido #500   │
│ user_123 │ Pedido #501   │          │ user_123 │ Pedido #501   │
│ user_123 │ Pedido #502   │          │ user_123 │ Pedido #502   │
└──────────────────────────┘          │ user_123 │ ☠ TOMBSTONE   │◄── "Pedido #502 foi
                                      │          │  (deletado em │    deletado no horário X"
                                      │          │   2026-09-30) │
                                      └──────────────────────────┘
```

> [!CAUTION]
> **O Efeito Cascata dos Tombstones:**  
> Se o sistema realizar muitos `DELETE`s, as SSTables ficarão repletas de *tombstones*. Quando uma consulta `SELECT` for executada, o Cassandra será forçado a ler milhares de marcadores de exclusão na memória antes de entregar o dado real ao cliente, degradando drasticamente a performance. Em casos extremos, isso causa erros de `ReadTimeoutException` ou `TombstoneOverwhelmingException`.
>
> **Regra prática:** Se o Cassandra tiver que pular mais de **1.000 tombstones** durante uma leitura, ele emite avisos críticos nos logs. Acima de **100.000**, a consulta é abortada por padrão.

### 1.3 A Chave Primária Composta: Partição + Ordenação

A chave primária no Cassandra é **a decisão mais importante** de toda a modelagem. Ela é dividida em dois componentes:

```sql
PRIMARY KEY ((partition_key), clustering_col_1, clustering_col_2)
--            ▲                ▲
--            │                └── Clustering Key: define a ORDEM FÍSICA
--            │                    dos dados dentro da partição no disco.
--            │
--            └── Partition Key: define EM QUAL NÓ do cluster
--                os dados são armazenados (via hash Murmur3).
```

| Componente | Função | Analogia |
| :--- | :--- | :--- |
| **Partition Key** | Determina **qual nó** do cluster armazena o dado. Todos os registros com a mesma chave de partição ficam fisicamente juntos no mesmo nó. | O **número do livro de ata** (ex: Ata da Conta #12345). |
| **Clustering Key** | Determina a **ordem física** dos registros **dentro da partição** no disco (ASC ou DESC). | A **data e hora** de cada registro dentro daquele livro de ata. |

---

## 2. A Regra de Ouro: Modelagem Orientada a Consultas (*Query-Driven Modeling*)

> [!IMPORTANT]
> **No Cassandra, você NÃO modela pensando nas entidades (como faria no PostgreSQL com normalização). Você modela pensando nas perguntas que a aplicação precisa responder.**
>
> Para cada consulta distinta que a UI ou a API precisa realizar, você cria **uma tabela específica** para respondê-la com eficiência máxima.

Isso significa que a **desnormalização (redundância de dados)** é uma prática **esperada e necessária**, não um antipadrão:

| Banco Relacional (PostgreSQL) | Cassandra |
| :--- | :--- |
| Dados normalizados em múltiplas tabelas. | Dados desnormalizados em tabelas dedicadas por consulta. |
| `JOIN` combina dados na hora da leitura. | **Não existem `JOIN`s.** Os dados já estão pré-combinados na gravação. |
| Uma tabela serve a múltiplas consultas. | Uma tabela serve idealmente a **uma única consulta**. |
| Otimizado para flexibilidade de consulta. | Otimizado para **velocidade de leitura** em padrões de acesso conhecidos. |

**Exemplo concreto:** Em um banco relacional, para saber o nome do cliente que fez uma avaliação de produto, você faria um `JOIN` com a tabela de usuários. No Cassandra, como não há `JOIN`, você grava o `customer_name` **diretamente dentro** da tabela de avaliações na hora da inserção. Se o cliente mudar de nome, a avaliação antiga preserva o nome original (comportamento de *snapshot*).

---

## 3. Exemplos Práticos: Onde o Cassandra Brilha (Padrão Ata)

### 3.1 Histórico de Transações Financeiras (Programa de Fidelidade / Extrato)

**Pergunta de Negócio:** *"Quais foram as compras do Cliente X, ordenadas das mais recentes para as mais antigas, para calcular o nível de pontos?"*

```sql
CREATE KEYSPACE IF NOT EXISTS ecommerce WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'datacenter1': 3
};

CREATE TABLE ecommerce.customer_transactions (
    customer_id   uuid,           -- Partition Key: agrupa todas as compras do mesmo cliente
    created_at    timestamp,      -- Clustering Key: ordena fisicamente por data (DESC)
    transaction_id timeuuid,      -- Identificador único do lançamento
    order_id      uuid,           -- Referência ao pedido no banco transacional
    transaction_type text,        -- 'PURCHASE', 'REFUND', 'CASHBACK', 'POINTS_EARNED'
    amount        decimal,
    currency      text,
    points_earned int,
    description   text,
    PRIMARY KEY ((customer_id), created_at, transaction_id)
) WITH CLUSTERING ORDER BY (created_at DESC, transaction_id DESC);
```

**Consulta otimizada (buscar os 20 lançamentos mais recentes):**
```sql
SELECT * FROM ecommerce.customer_transactions
WHERE customer_id = 018e3a2b-7c1e-7f32-8df2-36c57f5c9001
LIMIT 20;
```

> [!NOTE]
> **Por que funciona como Ata:** Cada compra gera um novo registro. Você **nunca** edita o valor de uma compra passada; se houver um estorno, insere-se uma **nova transação** com `transaction_type = 'REFUND'` e `amount` negativo. O histórico permanece íntegro e auditável.

---

### 3.2 Avaliações de Produtos (Monitoramento de Qualidade)

**Pergunta de Negócio:** *"Quais são as avaliações mais recentes do Produto Y para verificar se ele está recebendo reclamações recorrentes?"*

```sql
CREATE TABLE ecommerce.product_reviews (
    product_id    uuid,           -- Partition Key: agrupa avaliações por produto
    created_at    timestamp,      -- Clustering Key: ordena por data (DESC)
    review_id     timeuuid,       -- Identificador único da avaliação
    customer_id   uuid,
    customer_name text,           -- Desnormalizado: evita a necessidade de JOIN
    rating        int,            -- Notas de 1 a 5
    title         text,
    comment       text,
    PRIMARY KEY ((product_id), created_at, review_id)
) WITH CLUSTERING ORDER BY (created_at DESC, review_id DESC);
```

**Consulta otimizada (últimas 10 avaliações do produto):**
```sql
SELECT rating, customer_name, title, comment, created_at
FROM ecommerce.product_reviews
WHERE product_id = 018e3a2b-7c1e-7f32-8df2-36c57f5c9001
LIMIT 10;
```

> [!NOTE]
> **Por que funciona como Ata:** Os clientes dão notas que entram na **linha do tempo do produto**. O `customer_name` é gravado junto ao comentário (desnormalização), eliminando a necessidade de `JOIN`. O analista de estoque lê as avaliações mais recentes de forma sequencial e ultrarrápida para gerar alertas de qualidade.

---

### 3.3 Registros de HoneyPots (Segurança da Informação / *Threat Intelligence*)

**Pergunta de Negócio:** *"Quais tentativas de ataque ocorreram contra o HoneyPot da rede DMZ nas últimas 24 horas, agrupadas por IP de origem?"*

Um HoneyPot (servidor armadilha) exposto na internet pode sofrer **milhares de requisições maliciosas por segundo** (ataques de força bruta, varreduras de portas, injeção de exploits). Um banco relacional travaria com o volume de escrita. O Cassandra aguenta essa carga de forma linear.

```sql
CREATE TABLE security.honeypot_intrusion_logs (
    honeypot_id   text,           -- Partition Key: identifica qual armadilha capturou o evento
    occurred_at   timestamp,      -- Clustering Key: ordena por tempo (DESC)
    event_id      timeuuid,
    source_ip     inet,           -- IP do atacante
    source_port   int,
    target_port   int,            -- Porta visada no honeypot
    protocol      text,           -- 'TCP', 'UDP', 'ICMP'
    attack_type   text,           -- 'BRUTE_FORCE', 'PORT_SCAN', 'SQL_INJECTION'
    payload_hash  text,           -- Hash SHA-256 do payload para assinatura de malware
    country_code  text,           -- Geolocalização do IP via GeoIP
    PRIMARY KEY ((honeypot_id), occurred_at, event_id)
) WITH CLUSTERING ORDER BY (occurred_at DESC, event_id DESC)
  AND default_time_to_live = 7776000; -- TTL padrão de 90 dias
```

> [!NOTE]
> **Por que funciona como Ata:** Cada tentativa de ataque gera um log imutável. O analista de segurança consulta esses registros por período para identificar assinaturas de malware e padrões de ataque (*Threat Intelligence*). O `default_time_to_live` garante expurgo automático após 90 dias sem nenhum `DELETE` manual.

---

### 3.4 Log Centralizado de Eventos de Microsserviços

**Pergunta de Negócio:** *"Quais erros críticos ocorreram no serviço de pagamento nos últimos 15 minutos?"*

Grandes arquiteturas de microsserviços geram **gigabytes de logs por minuto** (`UserLoggedIn`, `CartAbandoned`, `PaymentFailed`). Centralizar esses eventos no Cassandra permite buscas rápidas por serviço e período.

```sql
CREATE TABLE observability.service_event_logs (
    service_name  text,           -- Partition Key: ex: 'checkout-service', 'catalog-api'
    event_date    date,           -- Parte da Partition Key: evita partições gigantes (bucket diário)
    occurred_at   timestamp,      -- Clustering Key
    event_id      timeuuid,
    log_level     text,           -- 'DEBUG', 'INFO', 'WARN', 'ERROR', 'FATAL'
    message       text,
    trace_id      text,           -- Correlação com tracing distribuído (Jaeger/OpenTelemetry)
    metadata      map<text, text>,
    PRIMARY KEY ((service_name, event_date), occurred_at, event_id)
) WITH CLUSTERING ORDER BY (occurred_at DESC, event_id DESC)
  AND default_time_to_live = 2592000; -- TTL de 30 dias
```

> [!TIP]
> **Bucket Temporal na Partition Key:** Observe que usamos `(service_name, event_date)` como chave de partição composta. Sem o campo `event_date`, um serviço com meses de operação acumularia milhões de linhas em uma única partição, violando o limite saudável de ~100MB por partição. O bucket diário mantém cada partição enxuta e performática.

---

### 3.5 Trilhas de Auditoria (*Compliance*: LGPD, GDPR, SOX)

**Pergunta de Negócio:** *"Quem alterou o nível de acesso do usuário administrativo X e quando isso aconteceu?"*

Para auditorias regulatórias (LGPD, GDPR, normas bancárias), é **obrigatório** registrar *quem alterou o quê no sistema*, e esse registro **nunca pode ser apagado ou adulterado**.

```sql
CREATE TABLE audit.entity_change_logs (
    entity_type   text,           -- 'USER', 'PRODUCT', 'ORDER', 'PAYMENT'
    entity_id     uuid,
    occurred_at   timestamp,
    event_id      timeuuid,
    actor_id      uuid,           -- Quem realizou a ação
    actor_name    text,           -- Desnormalizado para auditoria legível
    action        text,           -- 'CREATE', 'UPDATE', 'DELETE', 'ACCESS_LEVEL_CHANGED'
    field_changed text,           -- Qual campo foi alterado
    old_value     text,           -- Valor anterior (serializado)
    new_value     text,           -- Valor novo (serializado)
    ip_address    inet,
    user_agent    text,
    PRIMARY KEY ((entity_type, entity_id), occurred_at, event_id)
) WITH CLUSTERING ORDER BY (occurred_at DESC, event_id DESC);
```

> [!IMPORTANT]
> **Observação sobre imutabilidade e Tombstones:** Note que esta tabela de auditoria **não possui TTL** — os registros de *compliance* devem ser retidos pelo tempo que a legislação exigir. Se alguém tentar fraudar o sistema deletando registros, os *tombstones* resultantes degradarão a performance de leitura, gerando alertas visíveis nos logs do cluster. Isso torna a arquitetura **naturalmente resiliente a adulterações**.

---

### 3.6 Geolocalização em Tempo Real (Rastreamento de Frotas / Delivery)

**Pergunta de Negócio:** *"Qual é a posição atual e o trajeto recente do entregador #789?"*

Aplicativos de delivery e corrida coletam coordenadas GPS a cada 3–5 segundos. O volume de gravação é imenso e os dados antigos perdem relevância rapidamente.

```sql
CREATE TABLE logistics.driver_location_history (
    driver_id     uuid,
    tracking_date date,           -- Bucket diário para evitar partições gigantes
    recorded_at   timestamp,
    latitude      double,
    longitude     double,
    speed_kmh     float,
    heading       float,          -- Direção em graus (0-360)
    accuracy_m    float,          -- Precisão do GPS em metros
    PRIMARY KEY ((driver_id, tracking_date), recorded_at)
) WITH CLUSTERING ORDER BY (recorded_at DESC)
  AND default_time_to_live = 604800; -- TTL de 7 dias
```

> [!NOTE]
> **Por que funciona como Ata:** Cada ponto GPS gravado é uma entrada imutável no livro de ata do trajeto. O histórico vira a "ATA do trajeto feito". Com TTL de 7 dias, posições antigas são expurgadas automaticamente sem custo operacional.

---

## 4. Casos de Uso do Mundo Real: Quem Usa Cassandra em Escala?

| Empresa | Caso de Uso | Escala |
| :--- | :--- | :--- |
| **Apple** | Armazenamento de dados do iCloud e infraestrutura de backend. | 150.000+ nós Cassandra. |
| **Netflix** | Histórico de visualizações, dados de personalização e métricas de streaming. | Trilhões de registros. |
| **Discord** | Armazenamento de bilhões de mensagens de chat organizadas por canal (`channel_id`). | 177+ nós, 12 trilhões de mensagens (migrado para ScyllaDB em 2023). |
| **Uber** | Rastreamento de corridas em tempo real, precificação dinâmica e dados de mapa. | Múltiplos clusters com petabytes de dados. |
| **Nubank** | Histórico de transações financeiras e extrato de clientes. | Milhões de transações diárias. |

---

## 5. Antipadrões Fatais: O que NUNCA Fazer no Cassandra

> [!CAUTION]
> Os erros abaixo são os mais comuns em equipes que tentam usar o Cassandra como se fosse um banco relacional:

| # | Antipadrão | Por que é destrutivo? | O que fazer? |
| :--- | :--- | :--- | :--- |
| 1 | **`UPDATE` frequente de linhas existentes** | Cada `UPDATE` grava uma nova versão da linha em uma nova SSTable. Acumula dados duplicados até a compactação, desperdiçando disco e CPU. | Modele como *Append-Only*: insira novas linhas para representar mudanças de estado. |
| 2 | **`DELETE` em massa para "limpar" dados antigos** | Gera milhares de *tombstones* que degradam as leituras. | Use `TTL` (Time-to-Live) na inserção para expurgo automático e silencioso. |
| 3 | **Usar `ALLOW FILTERING` em consultas** | Força uma varredura em **todos os nós** do cluster (*full cluster scan*). Equivale a um `SELECT * FROM tabela` sem índice no PostgreSQL. | Crie uma **tabela desnormalizada dedicada** para a consulta específica. |
| 4 | **Criar índices secundários em colunas de alta cardinalidade** | Índices secundários no Cassandra são locais a cada nó e não distribuídos. Consultas exigem broadcast para todos os nós, gerando latências imprevisíveis. | Use tabelas materializadas ou tabelas invertidas para padrões de acesso alternativos. |
| 5 | **Partições sem limite de crescimento** | Uma partição que cresce indefinidamente (ex: todos os logs do sistema inteiro em uma única partição) ultrapassa o limite saudável de ~100MB. | Aplique **bucket temporal** na chave de partição: `((entity_id, year_month), created_at)`. |
| 6 | **Gravar valores `NULL` frequentemente** | Cada `NULL` no Cassandra é internamente tratado como um *tombstone*. | Omita a coluna na cláusula `INSERT` em vez de atribuir `NULL` explicitamente. |

---

## 6. Checklist de Produção para Apache Cassandra

Antes de promover um cluster Cassandra para produção, audite os seguintes itens:

* [ ] **Topologia de Cluster:** Mínimo de 3 nós com `NetworkTopologyStrategy` e fator de replicação `RF = 3`.
* [ ] **Nível de Consistência:** `LOCAL_QUORUM` para leitura e escrita como padrão seguro (garante leitura consistente sem sacrificar disponibilidade total).
* [ ] **TTL Configurado:** Todas as tabelas de dados efêmeros (logs, telemetria, cache de posição) possuem `default_time_to_live` ou inserções com `USING TTL`.
* [ ] **Monitoramento de Tombstones:** Alertas no Prometheus/Grafana para queries que varram mais de 500 tombstones por operação.
* [ ] **Tamanho de Partição:** Nenhuma partição deve ultrapassar 100MB de disco ou 100.000 células. Use bucket temporal na chave de partição para dados de alta ingestão.
* [ ] **Compactação Adequada:** Estratégia de compactação configurada conforme o padrão de acesso:
  * `SizeTieredCompactionStrategy (STCS)` — Padrão para *write-heavy*.
  * `TimeWindowCompactionStrategy (TWCS)` — Ideal para séries temporais com TTL.
  * `LeveledCompactionStrategy (LCS)` — Melhor para *read-heavy* com dados que raramente expiram.
* [ ] **Reparos Periódicos:** `nodetool repair` agendado semanalmente para garantir consistência entre réplicas.
* [ ] **Discos NVMe / SSD:** O Cassandra depende fortemente de I/O sequencial rápido. Discos mecânicos (HDD) são inaceitáveis para produção.

---

## 7. Referências Técnicas e Leitura Complementar

1. **Jeff Carpenter & Eben Hewitt** – *Cassandra: The Definitive Guide — Distributed Data at Web Scale* (O'Reilly, 3ª edição).
2. **Documentação Oficial do Apache Cassandra** – *Data Modeling Best Practices* (cassandra.apache.org).
3. **ScyllaDB University** – Curso gratuito de modelagem e administração compatível com CQL (university.scylladb.com).
4. **Discord Blog** – *How Discord Stores Trillions of Messages* (Migração de Cassandra para ScyllaDB e lições aprendidas).
5. **Martin Kleppmann** – *Designing Data-Intensive Applications*, Cap. 3: *Storage and Retrieval — LSM-Trees and SSTables*.
6. **Boas Práticas na Arquitetura de Persistência Poliglota** – [Tutorial Complementar](file:///var/www/html/tutoriais/databases/boas-praticas-uso-bd.md).
