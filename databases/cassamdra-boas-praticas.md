# Cassandra: Boas Práticas

> A essência arquitetural do Cassandra é baseada em registros imutáveis que funciona apenas como Append-Only (como, por exemplo, um livro de ata). É uma das melhores formas de entender o comportamento ideal desse banco de dados.
O Cassandra foi projetado nativamente para cenários onde você insere dados continuamente (Escrita) e os lê rapidamente (Consulta), evitando ao máximo atualizações (UPDATE) e exclusões (DELETE).
Existem dois motivos técnicos fundamentais, sob a perspectiva acadêmica, que explicam por que tratar o Cassandra como uma ATA é o cenário perfeito:
## 1. Escrita ultrarrápida (Log-Structured Merge-tree)
O Cassandra não procura um espaço livre no disco para atualizar uma linha quando você faz um comando.

* Ele simplesmente joga o dado novo no final de um arquivo de log na memória (Memtable) e o descarrega sequencialmente no disco (SSTable).
* Se você atualizar um dado (UPDATE), o Cassandra não modifica o registro antigo; ele apenas grava uma nova versão do dado com um timestamp mais recente. Quem lê sempre recebe a versão mais nova. Portanto, atualizações frequentes geram acúmulo de versões antigas no disco até que ocorra um processo interno de limpeza (compactação).

## 2. O terrível problema do "Tombstone" (Apagar dados)
Quando você deleta um dado no Cassandra (DELETE), ele não apaga o registro imediatamente. Como o disco é composto por arquivos imutáveis (SSTables), apagar algo exigiria reescrever arquivos gigantestos no disco a todo momento.

* Em vez de apagar, o Cassandra grava um marcador especial chamado Tombstone (Lápide) avisando: "Este dado foi deletado no horário X".
* Se o seu sistema acadêmico fizer muitos DELETEs, o banco ficará cheio dessas "lápides". Quando você for fazer uma consulta simples, o Cassandra terá que ler milhares de Tombstones na memória antes de te entregar o dado real, o que degrada drasticamente a performance (podendo causar erros de ReadTimeout ou OutOfMemory).

## Exemplos Perfeitos de Uso (Padrão ATA)
Por isso que os grandes casos de uso de mercado utilizam o Cassandra exatamente como você descreveu:

* Sistemas de Mensageria (ex: Chats): As mensagens são gravadas sequencialmente. Você lê o histórico e envia novas mensagens. Quase nunca edita ou apaga.
* Logs de Auditoria e Telemetria (IoT): Sensores enviando temperatura a cada segundo. Os dados entram como uma linha do tempo contínua.
* Histórico Financeiro: Extratos bancários onde cada transação é apenas inserida.

# Exemplo de uso onde o Cassamdra brilha:


## Cenário 1: Histórico de Transações (Programa de Benefícios)
A Pergunta de Negócio: "Quais foram as compras do Cliente X, ordenadas das mais recentes para as mais antigas, para calcular o seu nível de pontos?"
No Cassandra, usamos a chave de partição (customer_id) para agrupar as compras do mesmo cliente no mesmo nó físico do servidor, e uma chave de agrupamento (transaction_id ou created_at) para ordenar os dados automaticamente no disco.

CREATE TABLE ecommerce.customer_transactions (
    customer_id uuid,
    transaction_id uuid,
    amount decimal,
    points_earned int,
    created_at timestamp,
    PRIMARY KEY (customer_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);


* Por que funciona como ATA: Cada compra gera um novo registro. Você nunca edita o valor de uma compra passada; se houver um estorno, você insere uma nova transação com valor negativo.

------------------------------
## Cenário 2: Avaliações de Produtos (Gestão de Estoque)
A Pergunta de Negócio: "Quais são as avaliações mais recentes do Produto Y para verificar se ele está recebendo reclamações recorrentes?"
Aqui, queremos agrupar os dados por produto. Assim, o analista de estoque consegue puxar rapidamente o histórico de notas e comentários de um item específico para decidir se cancela a renovação do lote.

CREATE TABLE ecommerce.product_reviews (
    product_id uuid,
    review_id uuid,
    customer_id uuid,
    rating int,          -- Ex: Notas de 1 a 5
    comment text,
    created_at timestamp,
    PRIMARY KEY (product_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);


* Por que funciona como ATA: Os clientes dão notas que entram na linha do tempo do produto. O sistema lê as últimas avaliações de forma sequencial e ultrarrápida para gerar alertas de qualidade.

------------------------------
## Dica de Ouro para a sua Apresentação Acadêmica
Em um banco relacional como o PostgreSQL, para saber o nome do cliente que fez a avaliação, você usaria um JOIN com a tabela de usuários. No Cassandra, como não há JOIN, se você precisar exibir o nome do cliente na tela de avaliações, você grava o nome do cliente diretamente dentro da tabela de avaliações junto com o comentário. Esse conceito de "aceitar a redundância para ganhar velocidade" é o pilar central do ecossistema NoSQL.

# Outros exemplos:

Estes três são exemplos perfeitos e clássicos para o Apache Cassandra. Eles seguem rigorosamente o padrão de "ATA" (dados puramente sequenciais, temporais e imutáveis) e são amplamente utilizados pelo mercado e pela comunidade acadêmica.

## 1. Registros de HoneyPots (Segurança da Informação)

* Por que o Cassandra? Um HoneyPot (servidor armadilha) exposto na internet pode sofrer ataques automatizados massivos com milhares de requisições por segundo (ataques de força bruta, varreduras de portas, injeção de exploits). Um banco de dados tradicional travaria com o volume de escrita. O Cassandra aguenta essa carga de gravação de forma linear.
* A Estrutura (ATA): Cada tentativa de ataque gera um log contendo: IP de origem, timestamp, porta visada, payload enviado e tipo de protocolo. O analista de segurança consulta esses registros por período para identificar novas assinaturas de malware ou padrões de ataque (técnicas de Threat Intelligence).

## 2. Log de Eventos de Aplicação (Centralização de Logs)

* Por que o Cassandra? Grandes arquiteturas de microsserviços geram gigabytes de logs por minuto (ex: UserLoggedIn, CartAbandoned, PaymentFailed).
* A Estrutura (ATA): Os dados são particionados pelo identificador do serviço (service_name) e ordenados pelo tempo (timestamp). Isso permite que ferramentas de monitoramento busquem rapidamente os erros ocorridos em um microserviço específico nos últimos 15 minutos sem precisar varrer arquivos de texto espalhados por dezenas de servidores.

## 3. Trilhas de Auditoria (Compliance e Finanças)

* Por que o Cassandra? Para auditorias regulatórias (como LGPD, GDPR ou normas bancárias), é obrigatório registrar quem alterou o quê no sistema, e esse registro nunca pode ser apagado ou adulterado (característica de imutabilidade).
* A Estrutura (ATA): Toda vez que um usuário administrativo altera um dado crítico, o sistema gera uma linha na tabela de auditoria contendo: user_id, action (ex: "Mudança de nível de acesso"), old_value, new_value e timestamp. Se alguém tentar fraudar o sistema apagando um registro, o Cassandra não facilitará o processo (devido ao problema do Tombstone visto anteriormente), tornando a arquitetura naturalmente resiliente a deleções.

------------------------------
## Outros Exemplos de Mercado para Expandir seu Trabalho
Se você quiser diversificar para além do e-commerce e da infraestrutura de TI, pode citar estes três pilares onde o Cassandra domina o mercado real:

* Séries Temporais de Telemetria e IoT (Cidades Inteligentes / Indústria 4.0): Sensores de clima, medidores de energia elétrica ou rastreadores de frotas enviando coordenadas e métricas a cada segundo. O Cassandra armazena esses dados usando o ID do dispositivo como partição para plotagem rápida de gráficos de desempenho.
* Histórico de Geolocalização em Tempo Real (Aplicativos de Corrida/Delivery): Coleta da latitude e longitude de motoristas e entregadores a cada 5 segundos para que o usuário veja o ícone do veículo se movendo no mapa. O histórico vira uma "ATA" do trajeto feito.
* Sistemas de Mensagens e Chats (ex: Discord): O Discord utiliza o Cassandra para gerenciar bilhões de mensagens. Quando você entra em um canal, o banco lê de forma ultraeficiente a partição daquele canal (channel_id) trazendo as últimas mensagens postadas de maneira ordenada.



