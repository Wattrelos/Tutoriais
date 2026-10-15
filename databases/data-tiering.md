# 🧊 Data Tiering: Arquitetura de Dados em Camadas (Hot, Warm e Cold)

> **O Princípio do Ciclo de Vida da Informação:**  
> Mais de 80% do volume de dados em aplicações corporativas tem mais de 6 meses de vida e quase nunca é consultado. No entanto, se mantido no banco de dados operacional principal, esse histórico consome a maior parte dos custos de infraestrutura e causa mais de 90% da degradação de performance. **Data Tiering** (armazenamento em camadas) é a disciplina de arquitetura que move os dados para o lugar certo, no momento certo e pelo menor custo — mantendo a conformidade e a disponibilidade.

---

## Sumário
1. [O que é Data Tiering e Por Que Ele é Vital?](#1-o-que-é-data-tiering-e-por-que-ele-é-vital)
2. [As Três Camadas Estratégicas (Hot, Warm e Cold)](#2-as-três-camadas-estratégicas-hot-warm-e-cold)
3. [Cenário Real: E-commerce e Conformidade Fiscal (CTN e Ajuste SINIEF)](#3-cenário-real-e-commerce-e-conformidade-fiscal-ctn-e-ajuste-sinief)
4. [Arquitetura de Referência: Do Checkout ao Arquivo de Longo Prazo](#4-arquitetura-de-referência-do-checkout-ao-arquivo-de-longo-prazo)
5. [Tecnologias de Ponta para a Camada Morna: Por que ScyllaDB / Cassandra?](#5-tecnologias-de-ponta-para-a-camada-morna-por-que-scylladb--cassandra)
6. [Estratégias Práticas de Migração e Purga (Sem Derrubar a Produção)](#6-estratégias-práticas-de-migração-e-purga-sem-derrubar-a-produção)
7. [Modelagem Prática de Dados nas Três Camadas](#7-modelagem-prática-de-dados-nas-três-camadas)
8. [Camada de Acesso Unificado: Roteamento Transparente na Aplicação](#8-camada-de-acesso-unificado-roteamento-transparente-na-aplicação)
9. [Políticas de Ciclo de Vida no Object Storage (S3 Lifecycle)](#9-políticas-de-ciclo-de-vida-no-object-storage-s3-lifecycle)
10. [Armadilhas Comuns e Checklist Operacional](#10-armadilhas-comuns-e-checklist-operacional)
11. [Referências e Leituras Oficiais](#11-referências-e-leituras-oficiais)

---

## 1. O que é Data Tiering e Por Que Ele é Vital?

**Data Tiering** (ou Hierarquização de Armazenamento em Camadas) consiste em classificar e migrar automaticamente os dados entre diferentes tecnologias e mídias de armazenamento com base em três fatores principais:
1. **Frequência de Acesso:** Quantas vezes o dado é lido por minuto, dia ou ano.
2. **Requisitos de Performance (SLA):** Se a resposta precisa ocorrer em sub-milissegundos ou se pode aguardar alguns segundos/minutos.
3. **Custo por Gigabyte:** Otimizar o investimento em hardware e armazenamento em nuvem.

```
Frequência de Acesso & Custo/GB
▲
│    [Hot Tier]        SSD NVMe / RAM      Latência: < 5ms      Custo: $$$$
│    (Dados Quentes)   MySQL / Postgres    Escrita/Leitura ativa
│
│    [Warm Tier]       SSD Standard / EBS  Latência: 10-50ms    Custo: $$
│    (Dados Mornos)    ScyllaDB / NoSQL    Leituras analíticas / Imutáveis
│
│    [Cold Tier]       Object Storage      Latência: Segundos   Custo: $
│    (Dados Frios)     S3 / Parquet        Guarda fiscal / Auditoria
│
│    [Frozen / Deep]   Archive Storage     Latência: Horas      Custo: ¢
▼    (Arquivo Morto)   Glacier / Tape      Conformidade legal estrita
└─────────────────────────────────────────────────────────────────────────────► Idade do Dado (Tempo)
```

### O Efeito "Bomba-Relógio" nos Bancos Relacionais
Quando uma aplicação cresce sem uma política de Data Tiering, ela inevitavelmente sofre com a degradação cumulativa:
* **Poluição do Buffer Pool (Memória RAM):** O motor de banco de dados (ex: InnoDB no MySQL) precisa alocar páginas de dados antigos do disco para a memória RAM durante varreduras ou relatórios, expulsando dados quentes do checkout e elevando a latência geral.
* **Índices B-Tree Gigantescos:** A cada milhão de linhas inseridas, as árvores de índices ficam mais profundas. Inserções de novos pedidos tornam-se progressivamente mais lentas por causa das divisões de páginas (*page splits*).
* **Backups e Restaurações Inviáveis (RTO/RPO):** Realizar `mysqldump` ou snapshots de um banco de 5 TB pode levar dezenas de horas, inviabilizando planos de recuperação de desastres (*Disaster Recovery*).
* **Escalação Vertical Proibitiva:** Aumentar a máquina com centenas de gigabytes de RAM e SSDs NVMe empresariais para guardar dados de pedidos de 3 anos atrás é um enorme desperdício de capital.

---

## 2. As Três Camadas Estratégicas (Hot, Warm e Cold)

A divisão padrão de arquitetura de dados moderna utiliza três (ou quatro) camadas bem definidas:

| Característica | Camada Quente (Hot) | Camada Morna (Warm) | Camada Fria (Cold) | Arquivo Morto (Frozen/Deep) |
| :--- | :--- | :--- | :--- | :--- |
| **Idade do Dado** | 0 a 90 dias | 3 meses a 1 ano | 1 ano a 5 anos | Mais de 5 anos |
| **Padrão de Uso** | CRUD frequente, atualizações de status | Apenas leitura (*Append-Only*, imutável) | Consultas esporádicas de auditoria e suporte | Somente restauração sob demanda judicial |
| **SLA de Latência** | Sub-milissegundos a < 10 ms | 5 ms a 50 ms | 1 segundo a 5 segundos | Minutos a várias horas |
| **Tecnologias Típicas** | Redis, MySQL, PostgreSQL, MariaDB | ScyllaDB, Apache Cassandra, Elasticsearch | AWS S3, Cloud Storage, Apache Parquet, MinIO | AWS S3 Glacier Deep Archive, Fita LTO |
| **Tipo de Armazenamento** | Memória RAM / SSD NVMe de alta IOPS | SSD balanceado / Storages em nós distribuídos | Object Storage em nuvem / Discos HDD | Object Storage de arquivamento profundo |
| **Custo por GB** | Muito Alto ($$$$) | Moderado ($$) | Baixo ($) | Mínimo / Centavos (¢) |

### 2.1 Camada Quente (Hot Tier)
* **Objetivo:** Garantir a experiência em tempo real dos usuários e processar transações financeiras com garantias estritas de integridade (ACID).
* **Exemplos no E-commerce:** Carrinho de compras ativo, sessões de usuário, estoque em tempo real, pedidos recentes nos estados *Em Separação*, *Em Trânsito* ou dentro do prazo legal de devolução/arrependimento (7 a 30 dias).

### 2.2 Camada Morna (Warm Tier)
* **Objetivo:** Prover capacidade massiva de leitura rápida por chave, sem consumir os recursos de CPU e memória do banco relacional principal.
* **Exemplos no E-commerce:** Pedidos finalizados e entregues entre 3 e 12 meses atrás, extratos de faturamento, histórico de tickets de suporte concluídos.
* **Premissa Fundamental:** Os dados nesta camada são **imutáveis**. Ninguém altera o valor ou os itens de um pedido de 8 meses atrás; ele é apenas consultado.

### 2.3 Camada Fria (Cold Tier) e Arquivo Morto (Frozen)
* **Objetivo:** Atender exigências fiscais, contábeis e judiciais com o menor custo financeiro possível.
* **Exemplos no E-commerce:** Arquivos XML originais de NF-e e CT-e, recibos fiscais eletrônicos, trilhas completas de auditoria (*Audit Trail*) de transações concluídas há mais de 1 ano.
* **Mídia:** Armazenamento em arquivos colunares comprimidos (Apache Parquet) gravados em Object Storage (ex: AWS S3 ou MinIO local), com regras de transição automática para classes de arquivamento profundo.

---

## 3. Cenário Real: E-commerce e Conformidade Fiscal (CTN e Ajuste SINIEF)

No Brasil, o Data Tiering não é apenas uma recomendação de performance de software: é uma **necessidade de conformidade legal**. A legislação tributária obriga as organizações a armazenar dados transacionais e documentos fiscais eletrônicos por anos.

```
Ciclo Legal do Pedido e Documento Fiscal:
┌─────────────────┐      ┌─────────────────────────┐      ┌──────────────────────────┐
│  0 a 30 dias    │ ───► │   31 dias a 5 anos      │ ───► │  Após 5 anos fiscais     │
│  Operação Viva  │      │   Guarda Obrigatória    │      │  Decadência / Descarte   │
│ (Trocas / SAC)  │      │  (CTN Art. 173/174)     │      │  (Purga Definitiva)      │
└─────────────────┘      └─────────────────────────┘      └──────────────────────────┘
      [HOT]                        [WARM / COLD]                     [PURGE / EXPIRADO]
```

### 3.1 Prazos Legais e Fundamentação Tributária
* **Regra Geral para Empresas Privadas (5 Anos):**  
  Conforme o **Código Tributário Nacional (CTN)**, artigos 173 e 174, o prazo para a Fazenda Pública constituir o crédito tributário e cobrar judicialmente tributos prescreve em **5 anos** (contados a partir do primeiro dia do exercício seguinte àquele em que o lançamento poderia ter sido efetuado ou da data da emissão do documento).
  * Documentos abrangidos: Notas Fiscais Eletrônicas (**NF-e**), Notas Fiscais de Consumidor Eletrônicas (**NFC-e**), Conhecimentos de Transporte Eletrônico (**CT-e**) e seus respectivos arquivos **XML com assinatura digital**.
* **O Ajuste SINIEF nº 02/2025 (Prazos de 11 anos):**  
  O Ajuste SINIEF nº 2/2025 determinou que a guarda dos documentos fiscais eletrônicos pelo prazo de **11 anos (132 meses)** é obrigação estrita dos **órgãos públicos e administrações tributárias** (Receita Federal do Brasil, Secretarias da Fazenda dos Estados e do DF), mantendo-se para os contribuintes privados a regra tradicional de guarda de **5 anos**.
* **Exceções que Exigem Guarda Prolongada (> 5 a 10 anos):**
  * **Processos Administrativos ou Judiciais Pendentes:** Caso uma autuação ou processo esteja em curso, todos os documentos envolvidos devem ser conservados até a certidão de trânsito em julgado definitiva da decisão.
  * **Documentos Previdenciários e Trabalhistas:** Livros de registro, comprovantes de depósitos de FGTS e documentos associados a empréstimos e financiamentos de longo prazo podem exigir guarda por até 10 ou 30 anos.

> [!IMPORTANT]
> **O Desafio no Banco Relacional:**  
> Uma operação de e-commerce média gera facilmente 50.000 pedidos/dia. Em 5 anos, isso totaliza **mais de 90 milhões de pedidos** com seus itens, pagamentos e eventos fiscais. Tentar reter esse histórico no MySQL resulta em colapso do Buffer Pool, queries de checkout lentas e custo estratosférico de SSDs.

---

## 4. Arquitetura de Referência: Do Checkout ao Arquivo de Longo Prazo

Abaixo está o fluxo arquitetural de ponta a ponta demonstrando a divisão de responsabilidades e as ferramentas integradas:

```
[Aplicação Cliente / Web / App]
              │
              ▼
   ┌───────────────────────┐
   │  API Gateway / Router │ ◄── Identifica a idade da consulta (Data do Pedido)
   └──────────┬────────────┘
              │
     ┌────────┴──────────────┬────────────────────────┐
     │ Consulta < 90 dias    │ Consulta 90d a 1 ano   │ Consulta > 1 ano / Auditoria
     ▼                       ▼                        ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────┐
│   CAMADA HOT     │    │   CAMADA WARM    │    │       CAMADA COLD        │
│                  │    │                  │    │                          │
│ MySQL / MariaDB  │    │ ScyllaDB / CQL   │    │ Object Storage (S3/MinIO)│
│ (SSD NVMe)       │    │ (Nós Sharded C++)│    │ Arquivos Parquet / XMLs  │
└────────┬─────────┘    └────────┬─────────┘    └─────────────┬────────────┘
         │                       ▲                            ▲
         │ (Job ETL / CDC)       │                            │
         └───────────────────────┘                            │
                   │ (Compactação Periódica / Archive Job)    │
                   └──────────────────────────────────────────┘
```

### O Fluxo dos Dados no Ciclo de Vida:
1. **0 a 90 dias (Hot Tier):** O pedido nasce no **MySQL**. Sofre atualizações de status (autorizado, faturado, despachado, entregue, cancelado, estornado). Os dados residem em SSDs ultra-rápidos.
2. **Após 90 dias (Transição Hot ➔ Warm):** Uma vez entregue e decorrido o prazo legal de troca, o pedido é congelado. Um processo assíncrono (Job ETL em lote ou Change Data Capture) replica o registro completo em formato imutável para o **ScyllaDB**.
3. **Purga Segura no Hot:** Após confirmar a integridade e replicação no ScyllaDB, o registro é removido do MySQL via descarte de partição ou exclusão fracionada em lotes.
4. **Após 1 ano (Transição Warm ➔ Cold):** O pedido e seus respectivos arquivos XML de notas fiscais são exportados em arquivos colunares comprimidos (**Apache Parquet**) ou compactados em `.tar.zst` para o **Amazon S3 / MinIO**, reduzindo o custo de armazenamento a uma fração de centavo.

---

## 5. Tecnologias de Ponta para a Camada Morna: Por que ScyllaDB / Cassandra?

Para atuar como a **Camada Morna (Arquivo Vivo)**, o motor de banco de dados precisa ter características específicas que os bancos relacionais não oferecem:
* Escrita sequencial ultrarrápida (capacidade de absorver milhões de registros arquivados sem travar).
* Escalabilidade horizontal linear (basta adicionar nós baratos para aumentar a capacidade).
* Consultas por chave primária (`pedido_id` ou `cliente_id`) em milissegundos.

### 5.1 O Motor LSM-Tree: Gravação sem Gargalos de Disco
Diferente dos bancos B-Tree (como MySQL/Postgres), bancos baseados em **LSM-Tree** (como ScyllaDB e Apache Cassandra) gravam em estruturas sequenciais:
1. **CommitLog** (disco sequencial) + **MemTable** (memória RAM).
2. Quando a MemTable atinge o limite, gera um despejo no disco chamado **SSTable** (arquivo ordenado e 100% imutável).
3. Essa característica torna a ingestão em lote (*batch insert*) ordens de grandeza mais rápida e com zero contenção de bloqueios (*locks*).

### 5.2 ScyllaDB vs. Apache Cassandra: Por que o ScyllaDB se Destaca no Linux/Debian?
O **ScyllaDB** foi desenvolvido especificamente como um substituto direto (*drop-in replacement*) para o Apache Cassandra, reescrevendo o núcleo em **C++ Moderno** sobre o framework assíncrono de altíssimo desempenho **Seastar**:

```
Comparativo de Arquitetura:
┌──────────────────────────────────────┐    ┌──────────────────────────────────────┐
│           Apache Cassandra           │    │               ScyllaDB               │
├──────────────────────────────────────┤    ├──────────────────────────────────────┤
│ • Linguagem: Java (JVM)              │    │ • Linguagem: C++20 nativo (Seastar)  │
│ • Garbage Collection (GC Pauses)     │    │ • Zero GC (Gerenciamento manual)     │
│ • Bloqueios de concorrência global   │    │ • Arquitetura Shard-per-Core         │
│ • Drivers: CQL compatível            │    │ • 100% compatível com drivers CQL    │
│ • Consumo de memória alto            │    │ • Extrai 100% da CPU e SSD NVMe      │
│ • Pacotes Debian genéricos           │    │ • Repositórios APT oficiais Debian   │
└──────────────────────────────────────┘    └──────────────────────────────────────┘
```

#### Vantagens Críticas do ScyllaDB:
1. **Eliminação de Pausas por Garbage Collector:** No Cassandra tradicional em Java, coletas de lixo de memória (*Stop-the-World GC*) podem introduzir picos de latência de centenas de milissegundos ou segundos. No ScyllaDB, isso não existe.
2. **Arquitetura Shard-per-Core:** Cada núcleo de processador executa sua própria thread isolada, com sua própria memória e filas de I/O assíncronas (via `io_uring` no Linux), eliminando travas de sincronização (*mutex contention*).
3. **Compatibilidade Total com CQL:** Nenhuma linha de código de aplicação precisa ser alterada. Os mesmos drivers para Java, Python, Go, Node.js e PHP feitos para Cassandra conectam-se nativamente ao ScyllaDB.
4. **Suporte Nativo a Pacotes Debian:** A ScyllaDB mantém repositórios oficiais APT testados para as distribuições estáveis do Debian e Ubuntu.

### 5.3 O Segredo da Compactação para Arquivamento: TWCS
Ao modelar dados arquivados no ScyllaDB/Cassandra, a escolha da estratégia de compactação é crucial. Em vez de usar `SizeTieredCompactionStrategy` (STCS), deve-se configurar a **TimeWindowCompactionStrategy (TWCS)**:
* A TWCS agrupa dados que pertencem à mesma janela temporal (ex: 1 mês) na mesma SSTable física.
* Como dados históricos nunca mais sofrem `UPDATE`, aquela SSTable torna-se estável para sempre. O banco nunca mais gastará ciclos de CPU compactando arquivos antigos!

---

## 6. Estratégias Práticas de Migração e Purga (Sem Derrubar a Produção)

Muitas equipes cometem o erro grave de tentar limpar o banco relacional executando consultas ingênuas:
```sql
-- ⚠️ ERRO GRAVE EM PRODUÇÃO: NUNCA FAÇA ISSO!
DELETE FROM orders WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
```
Esse comando causará:
* **Travamento de Tabelas / Bloqueio de Linhas (*Row Locks*):** Impede novos checkouts enquanto a transação varre milhões de linhas.
* **Explosão do Log de Transações (Undo Log / Binary Log):** Pode esgotar o espaço em disco do servidor MySQL e paralisar as réplicas de leitura (*replication lag* extremo).
* **Fragmentação de Espaço:** O InnoDB **não devolve** espaço em disco para o sistema operacional após comandos `DELETE`; ele apenas mantém páginas vazias reutilizáveis internamente.

Abaixo estão os **dois padrões recomendados pela engenharia** para migração e descarte seguro:

### Padrão 1: Purga Fracionada em Lotes (Chunked Deletion)
Se a tabela não estiver particionada, a purga deve ser executada em pequenos lotes com pausas (*sleep*), preferencialmente por chave primária indexada:

```sql
-- Procedimento seguro de purga em lotes (executado em segundo plano)
DELIMITER $$

CREATE PROCEDURE PurgeOldOrders()
BEGIN
    DECLARE rows_affected INT DEFAULT 1;
    
    WHILE rows_affected > 0 DO
        -- Apaga apenas 5.000 registros por vez ordenados pela PK
        DELETE FROM orders 
        WHERE order_id IN (
            SELECT order_id FROM (
                SELECT order_id 
                FROM orders 
                WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
                  AND status IN ('DELIVERED', 'CANCELLED', 'REFUNDED')
                LIMIT 5000
            ) AS tmp
        );
        
        SET rows_affected = ROW_COUNT();
        
        -- Pausa de 500ms para permitir que a réplica sincronize o binlog
        -- e libere os locks de página do InnoDB
        DO SLEEP(0.5);
    END WHILE;
END$$

DELIMITER ;
```

### Padrão 2: Particionamento por Intervalo (Range Partitioning) — O Padrão Ouro
A forma mais limpa, rápida e elegante de implementar Data Tiering no MySQL/Postgres é o **Particionamento por Faixa de Data**:

```sql
-- Estrutura da tabela particionada mensalmente
CREATE TABLE orders (
    order_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id INT UNSIGNED NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (order_id, created_at)
) ENGINE=InnoDB
PARTITION BY RANGE (TO_DAYS(created_at)) (
    PARTITION p_2026_01 VALUES LESS THAN (TO_DAYS('2026-02-01')),
    PARTITION p_2026_02 VALUES LESS THAN (TO_DAYS('2026-03-01')),
    PARTITION p_2026_03 VALUES LESS THAN (TO_DAYS('2026-04-01')),
    PARTITION p_future   VALUES LESS THAN MAXVALUE
);
```

#### A Mágica do Descarte Instantâneo de Partição:
Quando um mês completa 90 dias e seus dados já foram replicados para o ScyllaDB, a remoção no MySQL é feita com um único comando DDL:

```sql
-- Remove 10 milhões de linhas em 0.01 segundos!
ALTER TABLE orders DROP PARTITION p_2026_01;
```
* **Tempo de Execução:** Instantâneo (< 50 milissegundos).
* **Impacto no Buffer Pool e Locks:** Zero. Não gera varredura de linhas, nem lock nas linhas ativas.
* **Espaço em Disco:** O arquivo `.ibd` da partição é desalocado imediatamente no sistema de arquivos, liberando espaço físico no SSD.

---

## 7. Modelagem Prática de Dados nas Três Camadas

A modelagem de um mesmo pedido evolui conforme ele percorre as camadas de armazenamento:

### 7.1 Camada Quente (MySQL): Normalizada e Transacional
Focada em integridade referencial, validação de chaves estrangeiras e consistência transacional:

```sql
-- Schema Relacional Hot
CREATE TABLE orders (
    order_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status ENUM('PENDING', 'PAID', 'SHIPPED', 'DELIVERED', 'REFUNDED') NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_customer (customer_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB;

CREATE TABLE order_items (
    item_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
) ENGINE=InnoDB;
```

### 7.2 Camada Morna (ScyllaDB / Cassandra): Desnormalizada e Imutável
Focada em consultas diretas por `customer_id` ou `order_id` em milissegundos, com desnormalização total dos itens e compactação TWCS:

```sql
-- Schema NoSQL Warm (Keyspace: ecommerce_archive)
CREATE KEYSPACE ecommerce_archive 
WITH replication = {'class': 'NetworkTopologyStrategy', 'datacenter1': 3};

CREATE TABLE ecommerce_archive.orders_by_customer (
    customer_id bigint,
    order_id bigint,
    created_at timestamp,
    total_amount decimal,
    status text,
    items_json text,         -- Snapshot serializado de todos os itens do pedido
    nfe_key text,            -- Chave de 44 dígitos da NF-e emitida
    shipping_address_json text,
    PRIMARY KEY ((customer_id), order_id)
) WITH CLUSTERING ORDER BY (order_id DESC)
  AND compaction = {
      'class': 'TimeWindowCompactionStrategy',
      'compaction_window_unit': 'DAYS',
      'compaction_window_size': '30'
  }
  AND compression = {
      'sstable_compression': 'ZstdCompressor',  -- Compressão extrema para economia de disco
      'chunk_length_in_kb': '64'
  };
```

### 7.3 Camada Fria (Parquet / Object Storage): Colunar e Comprimida
Os dados de auditoria e os XMLs das notas fiscais são gravados em arquivos Parquet particionados por ano e mês no bucket S3:

```
s3://meu-ecommerce-data-lake-cold/
  ├── nfe-xmls/
  │   └── ano=2026/
  │       └── mes=01/
  │           ├── 35260100000000000000550010000000011000000010.xml.gz
  │           └── 35260100000000000000550010000000021000000028.xml.gz
  └── pedidos-parquet/
      └── ano=2026/
          └── mes=01/
              ├── pedidos_part_001.snappy.parquet
              └── pedidos_part_002.snappy.parquet
```

> [!TIP]
> O formato **Apache Parquet** compacta dados tabulares em até 85% em relação ao JSON puro ou SQL texto, permitindo consultas ad-hoc rápidas diretamente na nuvem usando engines federadas como **DuckDB**, **Trino** ou **AWS Athena**, sem precisar carregar os dados em nenhum banco permanente.

---

## 8. Camada de Acesso Unificado: Roteamento Transparente na Aplicação

Para os desenvolvedores frontend ou serviços consumidores de APIs, a separação de camadas deve ser **completamente transparente**. O cliente da API não deve saber se o pedido veio do MySQL, do ScyllaDB ou do S3.

Isso é resolvido pelo padrão de arquitetura **Repository / Data Gateway Unificado**:

```python
from datetime import datetime, timedelta
from typing import Optional, Dict, Any

class UnifiedOrderRepository:
    def __init__(self, mysql_client, scylladb_client, s3_parquet_client):
        self.mysql = mysql_client
        self.scylla = scylladb_client
        self.s3 = s3_parquet_client
        self.hot_threshold_days = 90
        self.warm_threshold_days = 365

    def get_order(self, customer_id: int, order_id: int, order_date: datetime) -> Optional[Dict[str, Any]]:
        """
        Roteia a consulta dinamicamente para a camada correta 
        baseando-se na idade do pedido.
        """
        idade_dias = (datetime.now() - order_date).days

        # 1. Camada Hot (MySQL) - Pedidos recentes com ciclo operacional ativo
        if idade_dias <= self.hot_threshold_days:
            order = self.mysql.query(
                "SELECT * FROM orders WHERE customer_id = %s AND order_id = %s",
                (customer_id, order_id)
            )
            if order:
                return self._enrich_from_mysql(order)

        # 2. Camada Warm (ScyllaDB) - Pedidos históricos finalizados
        if idade_dias <= self.warm_threshold_days:
            query = "SELECT * FROM ecommerce_archive.orders_by_customer WHERE customer_id = %s AND order_id = %s"
            row = self.scylla.execute(query, (customer_id, order_id)).one()
            if row:
                return self._enrich_from_scylla(row)

        # 3. Camada Cold (S3 / Parquet via DuckDB ou Athena) - Auditoria profunda
        return self.s3.query_parquet_archive(customer_id, order_id, order_date.year)
```

---

## 9. Políticas de Ciclo de Vida no Object Storage (S3 Lifecycle)

Para a Camada Fria (Cold Tier) e os arquivos XML de notas fiscais que devem ser guardados por 5 anos pelo CTN, configure **Políticas de Ciclo de Vida (Lifecycle Rules)** no bucket de armazenamento em nuvem (S3 ou compatível):

```json
{
  "Rules": [
    {
      "ID": "DataTieringFiscalNFeLifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "nfe-xmls/"
      },
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 730,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 1825
      }
    }
  ]
}
```

### Explicação da Regra Econômica:
1. **0 a 90 dias (S3 Standard):** Acesso frequente para emissão de relatórios trimestrais e fechamentos fiscais.
2. **90 a 365 dias (S3 Standard-IA - Infrequent Access):** Custo de armazenamento 50% menor, com disponibilidade imediata para fiscalizações periódicas.
3. **1 a 2 anos (S3 Glacier Instant Retrieval):** Custo reduzido em até 68%, com tempo de recuperação em milissegundos.
4. **2 a 5 anos (S3 Glacier Deep Archive):** Custo de até **R$ 0,005 por Gigabyte/mês** (uma redução de ~95% nos custos de armazenamento).
5. **1825 dias (5 Anos completos):** **Expiração automática**. O próprio provedor de nuvem remove os arquivos em conformidade com o prazo de decadência do CTN, sem necessidade de intervenção manual.

---

## 10. Armadilhas Comuns e Checklist Operacional

Antes de colocar uma arquitetura de Data Tiering em produção, revise os seguintes riscos operacionais:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                CHECKLIST DE DATA TIERING                               │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ [ ] Verificação Dupla (Two-Phase Verification): O job de purga no Hot só deve          │
│     deletar registros após checar o checksum / contagem exata gravada no Warm.         │
│                                                                                        │
│ [ ] Evitar Tombstones no Cassandra: Não use DELETE na camada Warm; utilize apenas      │
│     TTL nativo (USING TTL) ou descarte de partição temporal TWCS.                      │
│                                                                                        │
│ [ ] Particionamento por Range: Sempre que possível, utilize PARTITION BY RANGE no      │
│     MySQL para descartar partições velhas instantaneamente com DROP PARTITION.         │
│                                                                                        │
│ [ ] Imutabilidade dos Snapshots: A camada Warm deve conter todos os dados do pedido    │
│     desnormalizados (preço praticado, frete, endereço na época), sem depender de      │
│     JOINs com tabelas mutáveis de catálogo ou usuários.                                │
│                                                                                        │
│ [ ] Automação de Retenção Legal: Os buckets da Camada Fria devem possuir Object Lock   │
│     (WORM - Write Once, Read Many) para impedir exclusão acidental durante o prazo     │
│     obrigatório de 5 anos do Código Tributário Nacional.                               │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Referências e Leituras Oficiais

### Legislação Tributária e Guarda de Documentos Fiscais
* [CRC-CE: Nota Fiscal Eletrônica - Guarda e Armazenamento após Ajuste SINIEF nº 2/2025](https://www.crc-ce.org.br/2025/04/nota-fiscal-eletronica-guarda-e-armazenamento-apos-ajuste-sinief-no-2-2025/)
* [Portal Contábeis: Ajuste SINIEF exige guarda de documentos fiscais por 11 anos para órgãos públicos](https://www.contabeis.com.br/artigos/70492/novo-ajuste-sinief-exige-guarda-de-documentos-fiscais-por-11-anos/)
* [Oobj Gestão: Por quanto tempo sua empresa deve guardar Notas Fiscais Eletrônicas?](https://oobj.com.br/gestao/por-quanto-tempo-guardar-notas-fiscais/)
* [Access Corp: Prazos Legais e Tributários para Guarda de Documentos Corporativos](https://accesscorp.com.br/blog/prazos-para-guarda-de-documentos-tributarios/)
* [SEFAZ-SP: Resposta à Consulta Tributária nº 24860/2021 sobre Guarda de XMLs](https://legislacao.fazenda.sp.gov.br/Paginas/RC24860_2021.aspx)
* [Mercado Pago: Prazos e regras de guarda para documentos fiscais e financeiros](https://www.mercadopago.com.br/blog/documentos-fiscais-financeiros-prazo-guarda)
* [Boletim CRC-BA: Temporalidade de Guarda de Livros Contábeis e Documentos Fiscais](http://www.crcba.org.br/boletim/edicoes/2054.htm)

### Tecnologias NoSQL e ScyllaDB
* [ScyllaDB Official Docs: Installation Guide on Linux / Debian](https://docs.scylladb.com/manual/stable/getting-started/install-scylla/install-on-linux.html)
* [ScyllaDB Architecture: ScyllaDB vs. Apache Cassandra Alternatives](https://www.scylladb.com/learn/apache-cassandra/alternatives/)
* [RisingWave Engineering: ScyllaDB vs. Cassandra - Exploring Architecture and Key Differences](https://risingwave.com/blog/scylladb-vs-cassandra-exploring-the-key-differences-and-similarities/)
* [ScyllaDB Engineering: Beyond Apache Cassandra - Performance and Shard-per-Core Design](https://www.scylladb.com/2025/09/02/beyond-apache-cassandra/)
* [JusDB Benchmarks: Technical Comparison Cassandra vs ScyllaDB](https://www.jusdb.com/compare/cassandra-vs-scylladb)
* [ScyllaDB OS Matrix: Official Linux Distribution Support](https://docs.scylladb.com/manual/branch-2025.1/getting-started/os-support.html)
