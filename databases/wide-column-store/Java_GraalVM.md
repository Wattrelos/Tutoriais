# Java + GraalVM + Spring Boot

> O Java pode (e deve) ser compilado normalmente, mesmo se você utilizar o Apache Cassandra ou o ScyllaDB no seu ecossistema.
A confusão sobre a compilação do Java e o uso do Cassandra geralmente acontece por causa de dois fatores distintos: a linguagem em que o banco foi escrito e a forma como a sua aplicação se comunica com ele.
Abaixo, detalho como essa relação funciona na prática e as opções de compilação modernas que você e seus colegas de faculdade podem explorar:

### 1. Sua aplicação Java é independente do Cassandra

O Cassandra é apenas um serviço externo rodando no Linux. A sua aplicação se comunica com ele através de uma biblioteca chamada Driver Java do Cassandra (fornecida pela DataStax).
* Você escreve seu código Java (usando Spring Boot, por exemplo).
* Você compila seu projeto gerando um arquivo .jar tradicional ou um executável nativo.
* O binário compilado roda na sua máquina ou VPS e envia comandos de rede (CQL) para o servidor do Cassandra.

### 2. O Cassandra roda em Java, mas você não o compila

O Apache Cassandra foi escrito em Java. Quando você o instala via apt no Debian 13, você está baixando o código que a comunidade Apache já compilou. Ele roda em cima da máquina virtual do Java (JVM / OpenJDK) do seu servidor.
(Nota: Se você optar pelo ScyllaDB, ele foi escrito e compilado em C++, eliminando a necessidade de uma JVM para o banco de dados, mas o seu código Java continua conversando com ele da mesmíssima forma).

### Opções Modernas de Compilação para o seu Stack (Java 21+)

Como você e seus colegas estão utilizando Java 21 ou superior com Spring Boot em servidores VPS (onde o consumo de memória RAM de 4GB pode ser um fator crítico), existem duas formas principais de compilar o Java hoje:

#### Abordagem A: Compilação Tradicional JIT (Bytecode)

É o padrão do ecossistema Java. O comando do Maven ou Gradle compila seu código em arquivos .class empacotados em um .jar.
* Comando: ./mvnw clean package
* Como funciona: O código é interpretado pela JVM e otimizado em tempo de execução (Just-In-Time).
* Impacto no VPS: Consome mais memória RAM no início (comum passar de 1GB a 2GB facilmente apenas para a aplicação Spring Boot subir).

#### Abordagem B: Compilação Nativa AOT (GraalVM) — Altamente Recomendada para VPS

O ecossistema Java moderno permite compilar o código diretamente para um binário nativo do Linux (arquitetura ELF), exatamente como o Rust ou o Go fazem, usando a GraalVM e o conceito de Ahead-Of-Time (AOT).
* Comando (no Spring Boot): ./mvnw spring-boot:build-image ou ./mvnw -Pnative native:compile
* Como funciona: O compilador analisa todo o seu código, remove o que não está sendo usado e gera um arquivo executável que não precisa de uma JVM instalada no servidor para rodar.
* Impacto no VPS: O tempo de inicialização cai de segundos para milissegundos e o consumo de memória RAM em repouso cai drasticamente (frequentemente de 1.5GB para menos de 150MB). Isso ajuda muito a liberar recursos no seu VPS de 4GB para que o Cassandra e o MariaDB rodem folgados.

#### Exemplo de conexão no Java (Spring Data Cassandra)

No seu código estruturado, as consultas de "Livro-Ata" compiladas ficariam protegidas em classes de repositório, onde você apenas insere os registros:

```java
@Repository
public interface LogAuditoriaRepository extends CassandraRepository<LogAuditoria, String> {
    // Código compilado nativamente para buscar dados por cliente no Cassandra
    List<LogAuditoria> findByClienteIdAndDataAfter(UUID clienteId, Instant data);
}
```j