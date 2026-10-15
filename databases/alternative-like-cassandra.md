# 🏛️ Persistência Poliglota com Wide-Column Store (ScyllaDB): Estudo Arquitetural, Crítica Técnica e Implementação

> **Resumo Executivo:**  
> Este estudo analisa a viabilidade, os benefícios reais, os custos operacionais ocultos e as armadilhas da adoção de **Persistência Poliglota** utilizando um banco de dados Wide-Column (**ScyllaDB / Cassandra**) para atuar como "Livro-Ata" imutável e arquivo morto no ecossistema do **Alpha Engine** (Rust, MariaDB, Redis, RabbitMQ sobre Debian 13).

---

## 📑 Sumário

1. [Visão Geral do Cenário e Motivação](#1-visão-geral-do-cenário-e-motivação)
2. [Revisão Crítica de Engenharia (O "Reality Check")](#2-revisão-crítica-de-engenharia-o-reality-check)
   - 2.1 O que o MariaDB suporta antes de precisar de NoSQL
   - 2.2 Os 4 Custos Ocultos de um Banco Wide-Column
   - 2.3 O Perigo do "Dual-Write Problem" e Inconsistência de Dados
3. [Comparativo Tecnológico: ScyllaDB vs Cassandra vs MariaDB Particionado](#3-comparativo-tecnológico-scylladb-vs-cassandra-vs-mariadb-particionado)
4. [A Arquitetura de Persistência Poliglota Recomendada](#4-a-arquitetura-de-persistência-poliglota-recomendada)
   - 4.1 O Fluxo CQRS Desacoplado
   - 4.2 O Padrão Transactional Outbox (A Cura do Dual-Write)
5. [Modelagem Prática em CQL: O Livro-Ata Imutável](#5-modelagem-prática-em-cql-o-livro-ata-imutável)
   - 5.1 O Tabu da Modelagem NoSQL: Queries First
   - 5.2 DDL da Tabela de Auditoria com TWCS (Time-Window Compaction)
   - 5.3 Evitando Partições Gigantes (Bucketing Temporal)
6. [Implementação no Ecossistema Rust (Driver Oficial `scylla`)](#6-implementação-no-ecossistema-rust-driver-oficial-scylla)
7. [Matriz de Decisão & Roadmap Recomendado](#7-matriz-de-decisão--roadmap-recomendado)

---

## 1. Visão Geral do Cenário e Motivação

No comércio eletrônico e plataformas de serviços B2B/B2C, os dados de pedidos possuem ciclos de vida bem distintos:

1. **Ciclo Quente e Transacional (0 a 30 dias):** Pagamentos, emissão fiscal, faturamento, picking, rastreamento logístico e disputas imediatas. Exige **ACID estrito**, integridade relacional e consultas dinâmicas.
2. **Ciclo Morno de Leitura (30 a 90 dias):** Trocas, devoluções (RMA), garantias e histórico recente do cliente. Consultas pontuais por cliente ou número de pedido.
3. **Ciclo Frio / Arquivo Morto (90 dias a 5 anos):** Auditoria fiscal, cumprimento da LGPD, livro-ata imutável e registros históricos. **Quase 100% de escrita (*Append-Only*)** e raríssima leitura, mas cujo volume de dados brutos (JSONs, logs de auditoria, eventos) pode inflar tabelas relacionais em dezenas de gigabytes.

A proposta de **Persistência Poliglota** busca alocar cada carga de trabalho no motor de banco de dados onde sua física de armazenamento é mais eficiente.

---

## 2. Revisão Crítica de Engenharia (O "Reality Check")

> [!WARNING]
> **Postura de Par de Programação Sênior:**  
> A introdução de uma nova tecnologia no stack deve ser justificada por métricas reais de escala ou gargalos de I/O comprovados, e não apenas por fascínio arquitetural (*Resume-Driven Development*). Abaixo estão as verdades técnicas sem rodeios sobre essa escolha.

### 2.1 Até onde o MariaDB/MySQL aguenta sozinho?
Existe um mito comum de que "o banco relacional não aguenta logs ou histórico de pedidos". Isso é falso na esmagadora maioria dos casos:
* O MariaDB com **InnoDB bem configurado e índices B-Tree compostos** lida com facilidade com **10 a 50 milhões de linhas** de pedidos e auditoria com tempos de resposta sub-10ms para buscas por chave primária ou índice indexado.
* Recursos nativos como **Particionamento por Faixa Temporal (`PARTITION BY RANGE (YEAR(created_at))`)** permitem que o MariaDB isole dados de anos anteriores em arquivos físicos separados (`.ibd`), tornando a exclusão ou arquivamento de partições antigas uma operação instantânea (`DROP PARTITION`) sem bloquear a tabela.
* **Conclusão:** Se o seu volume de pedidos for inferior a centenas de transações por segundo (TPS), o MariaDB **não está sofrendo nem sofrerá sobrecarga** se for mantido com índices corretos e expurgo/compactação periódica.

### 2.2 Os 4 Custos Ocultos do ScyllaDB/Cassandra

1. **Sobrecarga Brutal de Infraestrutura (Hardware):**
   * **Cassandra (Java):** Demanda uma JVM dedicada com 8 GB a 16 GB de RAM só para o processo do banco. Em servidores compartilhados (onde rodam Axum, MariaDB, Redis e RabbitMQ), o Cassandra causa contenção grave de memória.
   * **ScyllaDB (C++ / Seastar):** É infinitamente superior em consumo de recursos que o Cassandra (sem pausas de Garbage Collector), mas adota a arquitetura *Thread-per-core* com *Direct I/O* assíncrono (AIO). Ele foi projetado para "dominar" a CPU e o disco da máquina. Rodar ScyllaDB no mesmo nó onde o MariaDB precisa de I/O pode estrangular o banco relacional.
2. **Rigidez Extrema de Modelagem (Queries First):**
   * No MariaDB: você pode criar uma tabela e depois fazer `SELECT ... WHERE cliente_id = ? AND total > 500 ORDER BY created_at`.
   * No ScyllaDB/Cassandra: **Não existe consulta dinâmica sem índice específico criado previamente.** Se você tentar filtrar por uma coluna que não seja parte da Partition Key ou Clustering Key, a consulta falhará com erro (`ALLOW FILTERING` é estritamente proibido em produção por derrubar o cluster). Você é obrigado a duplicar tabelas para cada caso de consulta (*Table per Query*).
3. **Ausência de Transações Multi-Tabela e ACID Global:**
   * Bancos Wide-Column adotam consistência eventual (teorema CAP - sistema AP). Não existem foreign keys, nem triggers, nem commits transacionais entre diferentes tabelas.
4. **Curva de Aprendizado e Operação (DBA Overhead):**
   * É necessário monitorar reparos de anti-entropia (*nodetool repair*), compactação de SSTables, *tombstones* (marcas de exclusão que degradam a leitura) e distribuição de tokens no anel.

### 2.3 O Perigo do "Dual-Write Problem"

Se a sua aplicação fizer:
```rust
// ❌ ANTI-PATTERN: Dual-Write Ingênuo
maria_db.insert_order(&order).await?;
scylla_db.insert_ledger(&order).await?; // Se o Scylla falhar aqui, seus bancos divergiram!
```
Se a segunda gravação falhar (timeout de rede, nó ocupado, crash), o MariaDB terá o pedido, mas o ScyllaDB não terá o registro de auditoria. Tentar sincronizar dois bancos na camada de aplicação sem um protocolo distribuído (2PC) cria **inconsistência silenciosa**.

---

## 3. Comparativo Tecnológico: ScyllaDB vs Cassandra vs MariaDB Particionado

| Critério | Apache Cassandra 5.0 | ScyllaDB Open Source | MariaDB 10.11+ (Particionado) |
| :--- | :--- | :--- | :--- |
| **Linguagem / Runtime** | Java 17 (JVM) | C++20 (Framework Seastar) | C / C++ nativo |
| **Consumo Mínimo de RAM** | 8 GB a 16 GB recomendados | 2 GB a 4 GB (Escala com os cores) | 1 GB a 2 GB (Buffer Pool configurável) |
| **Latência P99** | Média a Alta (interrupções por GC) | Ultrabaixa e previsível (< 1ms) | Baixa a moderada |
| **Suporte ao Driver Rust** | Crate `cassandra-cpp` ou `cdrs-tokio` | Crate oficial **`scylla`** (Nativa Rust) | Crate oficial **`sqlx`** (Nativa Rust) |
| **Complexidade Operacional** | Alta (Tuning de JVM, GC, Trixie Python) | Média/Alta (Configuração de CPU/Disco) | Mínima (Já dominada no stack) |
| **Arquivamento em Disco Barato** | Manual via scripts ou S3 | **Hybrid/Tiered Storage nativo** | Particionamento com tabelas compactadas |

**Veredito:** Se o projeto realmente for adotar um banco Wide-Column, **o ScyllaDB é amplamente superior ao Cassandra** para o ecossistema Debian 13 e integração nativa com Rust.

---

## 4. A Arquitetura de Persistência Poliglota Recomendada

Para evitar sobrecarga no MariaDB e eliminar o *Dual-Write Problem*, a arquitetura deve operar sob o padrão **CQRS + Transactional Outbox Pattern**.

### 4.1 Topologia de Responsabilidades

```
                       ┌────────────────────────────────────────┐
                       │           Cliente / Navegador          │
                       └───────────────────┬────────────────────┘
                                           │
                        Requisição HTTP    ▼
                       ┌────────────────────────────────────────┐
                       │            API Rust (Axum)             │
                       └──────┬───────────────────────────┬─────┘
                              │                           │
                   [Escrita]  │                           │  [Leitura Rápida < 90d]
                              ▼                           ▼
        ┌───────────────────────────────────┐    ┌─────────────────────────────────┐
        │       MariaDB (Transacional)      │    │              Redis              │
        │  - Pedidos (Estado Atual)         │    │  - JSON Agregado do Pedido      │
        │  - Tabela outbox_events (ACID)    │    │  - Expiração automática TTL 90d │
        └─────────────────┬─────────────────┘    └────────────────┬────────────────┘
                          │                                       │
                Consome   │                                       │ Cache Miss (> 90d)
                Outbox    ▼                                       ▼
        ┌───────────────────────────────────┐    ┌─────────────────────────────────┐
        │        RabbitMQ (Broker)          │    │     ScyllaDB (Livro-Ata Frio)   │
        └─────────────────┬─────────────────┘    │  - Histórico Imutável Bruto     │
                          │                      │  - Particionado por Tempo/TWCS  │
            Evento        ▼                      └─────────────────────────────────┘
        ┌───────────────────────────────────┐
        │      Worker Rust Tokio (Worker)   │
        │  - Grava no ScyllaDB (Append-Only)│
        │  - Popula Cache no Redis          │
        └───────────────────────────────────┘
```

### 4.2 O Padrão Transactional Outbox (Eliminando o Dual-Write)

1. A API Axum grava o pedido e o evento na mesma transação local no MariaDB:
   ```sql
   START TRANSACTION;
   INSERT INTO orders (id, customer_id, total, status) VALUES (...);
   INSERT INTO outbox_events (aggregate_type, event_name, payload) VALUES (order, order.completed, {...});
   COMMIT;
   ```
2. Um processo leve lê os eventos da tabela `outbox_events` e os despacha para o **RabbitMQ**.
3. O **Worker** consome do RabbitMQ e executa duas tarefas desacopladas:
   * Insere o log bruto no **ScyllaDB** (Append-Only permanente).
   * Constrói a projeção desnormalizada no **Redis** com TTL de 90 dias.
4. **Benefício:** Se o ScyllaDB ficar offline temporariamente, as mensagens aguardam com segurança na fila do RabbitMQ. O MariaDB **nunca trava** e nenhuma informação é perdida.

---

## 5. Modelagem Prática em CQL: O Livro-Ata Imutável

No ScyllaDB, a modelagem de um livro de ata de pedidos deve respeitar duas regras de ouro:
1. **Evitar Partições Gigantes (Partition Hotspots):** Uma partição no ScyllaDB nunca deve exceder 100 MB de dados. Se colocarmos todos os pedidos de um cliente sob uma única Partition Key `customer_id`, clientes corporativos (B2B) com milhares de pedidos degradarão a partição.
2. **Utilizar Compactor TWCS (Time-Window Compaction Strategy):** Como os registros de auditoria nunca sofrem `UPDATE` ou `DELETE`, o TWCS cria arquivos SSTable imutáveis agrupados por períodos (ex: 30 dias), sem reescrever dados antigos.

### 5.1 Script DDL Completo em CQL

```sql
-- 1. Criação do Keyspace com fator de replicação
CREATE KEYSPACE IF NOT EXISTS alpha_ledger 
WITH replication = {
    class: SimpleStrategy, 
    replication_factor: 1
};

-- 2. Tabela de Livro-Ata com Bucketing Temporal
CREATE TABLE IF NOT EXISTS alpha_ledger.order_audit_trail (
    customer_id uuid,
    bucket_year_month int,          -- Ex: 202610 (Garante limites saudáveis de partição)
    event_timestamp timestamp,
    order_id uuid,
    event_type text,                -- ORDER_CREATED, PAYMENT_APPROVED, DISPATCHED
    actor_id uuid,                  -- ID do usuário ou sistema que provocou o evento
    client_ip text,
    payload_snapshot text,          -- JSON consolidado contendo snapshot total do pedido
    PRIMARY KEY ((customer_id, bucket_year_month), event_timestamp, order_id)
) WITH CLUSTERING ORDER BY (event_timestamp DESC, order_id ASC)
AND compaction = {
    class: TimeWindowCompactionStrategy,
    compaction_window_unit: DAYS,
    compaction_window_size: 30,
    timestamp_resolution: MILLISECONDS
};
```

### 5.2 Como essa modelagem responde às consultas da aplicação:

1. **Consulta Instantânea (Recentes do Mês):**
   ```sql
   SELECT * FROM alpha_ledger.order_audit_trail 
   WHERE customer_id = ? AND bucket_year_month = 202610;
   ```
   *Retorna os pedidos mais recentes primeiro (`event_timestamp DESC`) em frações de milissegundo, lendo de uma única partição contígua em disco.*

2. **Consulta de Histórico Antigo (Arquivo Morto de 1 ano atrás):**
   ```sql
   SELECT * FROM alpha_ledger.order_audit_trail 
   WHERE customer_id = ? AND bucket_year_month = 202510;
   ```
   *Lê cirurgicamente o bucket antigo sem varrer a tabela inteira e sem acordar o MariaDB.*

---

## 6. Implementação no Ecossistema Rust (Driver Oficial `scylla`)

O ScyllaDB possui suporte de primeira classe em Rust através da crate **`scylla`**, desenvolvida e mantida pela própria ScyllaDB Corporation. Ela é 100% assíncrona, construída sobre o **Tokio** e é *Shard-Aware* (estabelece conexões diretamente com a CPU/core responsável por aquela partição).

### 6.1 Adição ao `Cargo.toml`

```toml
[dependencies]
scylla = "0.15"
tokio = { version = "1.0", features = ["full"] }
uuid = { version = "1.0", features = ["v4"] }
chrono = "0.4"
```

### 6.2 Exemplo de Gravação Assíncrona no Worker Rust

```rust
use scylla::transport::session::Session;
use scylla::SessionBuilder;
use std::sync::Arc;
use uuid::Uuid;
use chrono::Utc;

pub struct ScyllaLedgerRepository {
    session: Arc<Session>,
}

impl ScyllaLedgerRepository {
    pub async fn new(uri: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let session = SessionBuilder::new()
            .known_node(uri)
            .build()
            .await?;
        Ok(Self { session: Arc::new(session) })
    }

    pub async fn record_audit_event(
        &self,
        customer_id: Uuid,
        order_id: Uuid,
        event_type: &str,
        payload_json: &str,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let now = Utc::now();
        // Gera o bucket temporal (Ex: 202610)
        let bucket = (now.format("%Y%m").to_string()).parse::<i32>()?;

        let query = "
            INSERT INTO alpha_ledger.order_audit_trail 
            (customer_id, bucket_year_month, event_timestamp, order_id, event_type, payload_snapshot)
            VALUES (?, ?, ?, ?, ?, ?)
        ";

        self.session
            .query_unpaged(query, (
                customer_id,
                bucket,
                now.timestamp_millis(),
                order_id,
                event_type,
                payload_json
            ))
            .await?;

        Ok(())
    }
}
```

---

## 7. Matriz de Decisão & Roadmap Recomendado

### 💡 Recomendações Práticas para o Projeto Atual

| Situação do Projeto | Recomendação Arquitetural | Justificativa |
| :--- | :--- | :--- |
| **Volume Atual (Milhares a Dezenas de Milhares de Pedidos/mês)** | **Mantenha tudo no MariaDB + Redis + RabbitMQ** | O MariaDB opera com sobra de folga. Adicionar um banco NoSQL neste momento aumentará os custos de infraestrutura e complexidade sem ganho perceptível. |
| **Volume Médio (Logs textuais e auditorias consumindo > 100 GB)** | **Particionamento Nativo no MariaDB + S3/MinIO** | Aplique `PARTITION BY RANGE` no MariaDB e exporte JSONs de pedidos de mais de 1 ano para armazenamento de objetos econômico. |
| **Alto Volume (> 500k eventos/dia ou telemetria em tempo real)** | **Adotar ScyllaDB como Livro-Ata via Worker RabbitMQ** | Neste ponto, a física de gravação do ScyllaDB se paga, absorvendo gigabytes de logs contínuos com custo zero de locking no MariaDB. |

### 🛑 Checklist: O Que NUNCA Fazer

1. **Nunca faça queries relacionais no ScyllaDB:** Não tente fazer `JOIN`, consultas sem partition key ou somatórios arbitrários.
2. **Nunca use Cassandra se puder usar ScyllaDB no Linux:** O consumo excessivo de memória da JVM do Cassandra é um passivo técnico desnecessário para quem roda em servidores Linux modernos.
3. **Nunca grave diretamente na API sem mensageria:** Use sempre o RabbitMQ como amortecedor. Se o banco NoSQL passar por manutenção ou compactação pesada, sua API HTTP permanecerá rápida e resiliente.
