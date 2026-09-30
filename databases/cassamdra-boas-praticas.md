# Cassandra: Boas Práticas

> A essência arquitetural do Cassandra é baseada em registros imutáveis que funciona apenas como Append-Only (como, por exemplo, uma Livro de Ata). É uma das melhores formas de entender o comportamento ideal desse banco de dados.
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

Para o seu projeto acadêmico, você pretende modelar um cenário desse tipo (como logs de eventos, sistemas de chat ou séries temporais)? Se quiser, posso te mostrar como definir as chaves de partição e ordenação (PRIMARY KEY e CLUSTERING KEY) para estruturar essa linha do tempo de forma super eficiente no Cassandra.

