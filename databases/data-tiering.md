# Data Tiering

Data Tiering (ou Armazenamento em Camadas) é uma estratégia de arquitetura de dados que consiste em organizar e mover informações entre diferentes tipos de armazenamento com base na frequência de acesso, desempenho necessário e custo.
Em vez de manter todos os dados do sistema no mesmo banco de dados de alta performance (o que custaria muito caro e deixaria o sistema lento), os dados são divididos em camadas (tiers) conforme o seu ciclo de vida.
O conceito é baseado no comportamento natural da informação: dados novos são acessados a todo momento, enquanto dados antigos raramente são consultados, mas ainda precisam ser guardados por questões legais ou fiscais.

## As Três Camadas Tradicionais do Data Tiering

Para entender como funciona, imagine o armazenamento dividido em três níveis principais:

### Camada Quente (Hot Tier):

	• O que armazena: Dados operacionais, em tempo real, que sofrem modificações constantes. No e-commerce, são os carrinhos de compras, usuários logados e pedidos dos últimos meses.
	• Tecnologia: Bancos de dados relacionais rápidos (como MySQL, PostgreSQL) ou caches em memória (Redis), rodando em SSDs de altíssima velocidade.
	• Foco: Baixíssima latência e alta taxa de transferência.

### Camada Morna (Warm Tier):

	• O que armazena: Dados que ainda são consultados com certa frequência, mas que não sofrem mais alterações (são imutáveis). Por exemplo, o histórico de pedidos finalizados no último ano.
	• Tecnologia: Bancos NoSQL distribuídos (como Apache Cassandra) ou instâncias de bancos relacionais otimizadas para leitura, rodando em storages de custo intermediário.
	• Foco: Alta capacidade de armazenamento com desempenho decente para consultas.

### Camada Fria (Cold Tier / Archive):
		• O que armazena: Dados antigos que raramente ou nunca são acessados, mantidos apenas por auditoria ou obrigação legal (como os 5 anos de dados fiscais).
		• Tecnologia: Object Storage na nuvem (como AWS S3, Google Cloud Storage) ou soluções de fita/Glacier.
		• Foco: O menor custo possível por Gigabyte. O acesso pode demorar de minutos a horas para ser liberado.

## Por que o Data Tiering é vital para grandes aplicações?

1. **Redução de Custos:** Manter Terabytes de dados antigos em SSDs rápidos de produção é um desperdício financeiro. O tiering move o grosso do volume para mídias muito mais baratas.
2. **Ganho de Performance:** O banco de dados principal (Hot) permanece leve e enxuto. Tabelas menores significam índices menores na memória RAM, resultando em queries de escrita e leitura muito mais rápidas para os usuários ativos.
3. **Escalabilidade e Sustentabilidade:** Evita o efeito "bomba-relógio", onde a aplicação inevitavelmente quebra ou fica inutilizável conforme o volume de dados acumulados cresce ao longo dos anos.
