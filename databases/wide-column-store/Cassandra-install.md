# 📦 Guia de Instalação e Configuração: Apache Cassandra 5.0 no Debian 13 (Trixie)

Este guia prático documenta o passo a passo completo para instalação, inicialização, endurecimento de segurança e resolução de incompatibilidades do **Apache Cassandra 5.0 (série 50x)** no **Debian 13 (Trixie)**.

---

## 📑 Sumário

1. [Pré-requisitos e Ambiente Java](#1-pré-requisitos-e-ambiente-java)
2. [Adição do Repositório Oficial Apache](#2-adição-do-repositório-oficial-apache)
3. [Instalação do Apache Cassandra](#3-instalação-do-apache-cassandra)
4. [Gerenciamento e Verificação do Serviço](#4-gerenciamento-e-verificação-do-serviço)
5. [Resolução do Problema do cqlsh no Debian 13 (Python 3.13)](#5-resolução-do-problema-do-cqlsh-no-debian-13-python-313)
6. [Configurações Essenciais de Produção (cassandra.yaml)](#6-configurações-essenciais-de-produção-cassandrayaml)
7. [Tuning do Sistema Operacional e Kernel (Debian 13)](#7-tuning-do-sistema-operacional-e-kernel-debian-13)
8. [Topologia de Portas de Rede](#8-topologia-de-portas-de-rede)
9. [Comandos de Diagnóstico e Telemetria](#9-comandos-de-diagnóstico-e-telemetria)
10. [Referências e Leituras Recomendadas](#10-referências-e-leituras-recomendadas)

---

## 1. Pré-requisitos e Ambiente Java

O **Apache Cassandra 5.x** exige o **Java 11 ou Java 17** em tempo de execução. No Debian 13, o padrão homologado e mais estável é o **OpenJDK 17**.

Atualize os repositórios locais e instale os pacotes de suporte e a JVM:

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk curl gnupg2 apt-transport-https
```

Confirme se o Java 17 está ativo como interpretador padrão:

```bash
java -version
```

*Saída esperada (exemplo):*
```text
openjdk version "17.0.x" ...
OpenJDK Runtime Environment (build 17.0.x+...)
OpenJDK 64-Bit Server VM (build 17.0.x+..., mixed mode, sharing)
```

---

## 2. Adição do Repositório Oficial Apache

Para instalar os binários oficiais mantidos pela Apache Software Foundation, configure a chave GPG de assinatura e o repositório da série **50x** (Cassandra 5.0).

> [!IMPORTANT]
> **Atenção à Chave GPG:** Certifique-se de importar o chaveiro público do endereço canônico de distribuição da Apache (`downloads.apache.org/cassandra/KEYS`), garantindo a validação criptográfica dos pacotes `.deb`.

```bash
# 1. Cria o diretório canônico de chaveiros caso ainda não exista
sudo mkdir -p /etc/apt/keyrings

# 2. Baixa e desarmadura a chave oficial de assinatura dos releases da Apache
curl -sL https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /etc/apt/keyrings/cassandra-archive-keyring.gpg

# 3. Registra a fonte de pacotes da série estável 50x (Cassandra 5.0)
echo "deb [signed-by=/etc/apt/keyrings/cassandra-archive-keyring.gpg] https://debian.cassandra.apache.org 50x main" | sudo tee /etc/apt/sources.list.d/cassandra.list
```

---

## 3. Instalação do Apache Cassandra

Atualize o cache de metadados do APT para reconhecer o novo repositório e instale o pacote `cassandra`:

```bash
sudo apt update
sudo apt install -y cassandra
```

Durante a instalação, o Debian cria automaticamente:
* O usuário e grupo de sistema `cassandra`.
* O diretório de configurações em `/etc/cassandra/`.
* Os diretórios de dados em `/var/lib/cassandra/` (data, commitlog, saved_caches).
* Os diretórios de logs em `/var/log/cassandra/`.
* A unidade systemd `/lib/systemd/system/cassandra.service`.

---

## 4. Gerenciamento e Verificação do Serviço

Habilite a inicialização automática do Cassandra junto com o boot do sistema e inicie o serviço:

```bash
# Habilita o serviço para o boot e inicia imediatamente
sudo systemctl enable --now cassandra
```

### 4.1 Checagem do Estado do Serviço

```bash
sudo systemctl status cassandra
```

> [!NOTE]
> O Cassandra inicializa diversas threads de JVM, buffers de MemTable e validação de SSTables em disco. Pode levar entre **15 e 40 segundos** para que a porta CQL nativa (`9042`) esteja totalmente vinculada e pronta para conexões.

### 4.2 Verificação do Cluster via `nodetool`

O utilitário oficial `nodetool` permite inspecionar o status operacional do nó:

```bash
nodetool status
```

*Interpretação do retorno do `nodetool`:*
```text
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load        Tokens  Owns (effective)  Host ID                               Rack
UN  127.0.0.1  250.15 KiB  16      100.0%            a1b2c3d4-e5f6-7890-abcd-1234567890ab  rack1
```

* **`UN` (Up / Normal):** O nó está ativo, comunicando-se perfeitamente e operando em estado de produção normal.
* **`DN` (Down / Normal):** O nó está inacessível ou o serviço foi interrompido.

---

## 5. Resolução do Problema do cqlsh no Debian 13 (Python 3.13)

### ⚠️ Por que o `cqlsh` falha no Debian 13?

O **Debian 13 (Trixie)** adota nativamente o **Python 3.13**. Nesta versão, a linguagem removeu módulos legados obsoletos (como `pipes`, `cgi`, `asyncore` e `asynchat` — [PEP 594](https://peps.python.org/pep-0594/)). Como o utilitário `/usr/bin/cqlsh` depende de bibliotecas que ainda invocam essas importações, a execução direta do comando gera erros de inicialização (`ModuleNotFoundError: No module named 'pipes'`).

Abaixo estão as três abordagens recomendadas para contornar essa situação com elegância:

---

### Opção A: Execução via Container Docker Efêmero (Recomendada / Zero-Impact)

A forma mais limpa, que não polui o ambiente Python do sistema e não quebra futuras atualizações de pacotes do Debian, é executar o `cqlsh` através de uma imagem oficial temporária compartilhando a rede local:

```bash
# Conexão direta ao nó local
docker run -it --rm --network host cassandra:5.0 cqlsh localhost 9042
```

#### Criação de um Alias Permanente no Terminal:

Adicione um alias ao seu arquivo `~/.bashrc` (ou `~/.zshrc`) para utilizá-lo exatamente como se fosse o binário nativo:

```bash
echo "alias cqlsh='docker run -it --rm --network host cassandra:5.0 cqlsh'" >> ~/.bashrc
source ~/.bashrc
```

Agora você pode simplesmente rodar:
```bash
cqlsh localhost 9042
```

---

### Opção B: Instalação Isolada via `pipx` com Python Compatível

Se preferir não usar Docker, instale o `pipx` para criar um ambiente isolado com uma versão Python que preserve os módulos necessários:

```bash
sudo apt install -y pipx
pipx ensurepath
```

Execute o `cqlsh` especificando o interpretador estável:

```bash
pipx run --python python3.11 cqlsh localhost 9042
```

---

### Opção C: Ambiente Virtual com `uv` ou `venv`

Caso utilize o gerenciador rápido `uv`:

```bash
# Cria um venv efêmero com Python 3.12 e roda o cqlsh
uv run --python 3.12 --with cqlsh cqlsh localhost 9042
```

---

## 6. Configurações Essenciais de Produção (cassandra.yaml)

O arquivo principal de configuração reside em:
👉 `/etc/cassandra/cassandra.yaml`

Antes de colocar o servidor em rede ou em produção, ajuste os seguintes parâmetros:

```yaml
# 1. Nome do Cluster (Todos os nós do mesmo cluster devem ter o mesmo nome)
cluster_name: 'Alpha_Production_Cluster'

# 2. Endereço de escuta para comunicação interna entre nós (inter-node)
listen_address: 192.168.1.100

# 3. Endereço onde o Cassandra escuta conexões de clientes (CQL / Aplicações)
rpc_address: 0.0.0.0

# 4. Endereço público/anunciado para clientes se houver NAT/Firewall
broadcast_rpc_address: 192.168.1.100

# 5. Lista de nós de inicialização (Seeds) para descoberta de topologia
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "192.168.1.100,192.168.1.101"

# 6. SEGURANÇA: Habilitar autenticação com usuário e senha
# ATENÇÃO: O padrão de fábrica 'AllowAllAuthenticator' aceita qualquer conexão sem senha!
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

> [!CAUTION]
> **Troca da Senha Padrão de Fábrica:**
> Ao habilitar o `PasswordAuthenticator`, o usuário padrão inicial é **`cassandra`** com a senha **`cassandra`**. Altere-a imediatamente no primeiro acesso via `cqlsh`:
> ```sql
> ALTER USER cassandra WITH PASSWORD 'NovaSenhaForte!2026';
> ```

Após editar o arquivo `/etc/cassandra/cassandra.yaml`, reinicie o daemon para aplicar as mudanças:

```bash
sudo systemctl restart cassandra
```

---

## 7. Tuning do Sistema Operacional e Kernel (Debian 13)

Para evitar degradação de I/O e pausas de Stop-The-World da JVM em servidores de banco de dados, aplique as seguintes diretrizes de infraestrutura:

### 7.1 Limites de Memória Mapeada e Swap (`/etc/sysctl.d/99-cassandra.conf`)

Crie o arquivo de parametrização de kernel:

```bash
sudo tee /etc/sysctl.d/99-cassandra.conf << EOF
# Aumenta o limite de tabelas de memória virtual mapeadas (mmap)
vm.max_map_count = 1048575

# Reduz drasticamente a troca em swap (evita travar nós da JVM)
vm.swappiness = 1

# Ajusta buffers de rede TCP para alta vazão
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
EOF

sudo sysctl --system
```

### 7.2 Limites de Descritores de Arquivos (`/etc/security/limits.d/cassandra.conf`)

O Cassandra abre simultaneamente milhares de descritores de SSTables e conexões TCP:

```bash
sudo tee /etc/security/limits.d/cassandra.conf << EOF
cassandra - memlock unlimited
cassandra - nofile 100000
cassandra - nproc 32768
cassandra - as unlimited
EOF
```

### 7.3 Alocação de Memória JVM (Heap Size)

No Cassandra 5.0 com Java 17, as diretivas de memória residem em:
👉 `/etc/cassandra/jvm17-server.options`

Para servidores dedicados, a recomendação geral é alocar **50% da memória RAM física disponível** para a JVM (até o limite típico de 16GB a 31GB para evitar perda de ponteiros comprimidos `UseCompressedOops`):

```properties
# Exemplo para servidor com 16 GB de RAM:
-Xms8G
-Xmx8G
```

---

## 8. Topologia de Portas de Rede

Certifique-se de configurar o firewall (UFW / iptables / AWS Security Groups) de acordo com a função de cada porta:

| Porta | Protocolo | Padrão | Finalidade | Exposição Recomendada |
| :---: | :---: | :---: | :--- | :--- |
| **`9042`** | TCP | CQL | Conexões de clientes e drivers de aplicação (Rust, Python, Java) | Rede Interna / VPC |
| **`7000`** | TCP | Gossip | Comunicação interna entre nós do cluster | Apenas entre nós Cassandra |
| **`7001`** | TCP | Gossip TLS | Comunicação inter-node criptografada com TLS | Apenas entre nós Cassandra |
| **`7199`** | TCP | JMX | Monitoramento local e comandos do `nodetool` | Somente `localhost` |
| **`9160`** | TCP | Thrift | Protocolo legado Thrift (Descontinuado nas versões modernas) | Manter desativado |

---

## 9. Comandos de Diagnóstico e Telemetria

Tabela rápida com os comandos essenciais para a rotina do Administrador de Banco de Dados (DBA):

| Comando | Finalidade |
| :--- | :--- |
| `nodetool status` | Exibe o estado operacional, IPs e carga de dados de todos os nós |
| `nodetool info` | Exibe uptime, uso de memória Heap e contadores de tokens |
| `nodetool ring` | Mostra a distribuição do anel e os intervalos de tokens por nó |
| `nodetool tablestats` | Estatísticas detalhadas de leitura/escrita, MemTables e SSTables |
| `nodetool netstats` | Diagnóstico de conexões de streaming e replicação ativa |
| `journalctl -u cassandra -f` | Acompanhamento em tempo real dos logs de inicialização via systemd |
| `tail -f /var/log/cassandra/system.log` | Log detalhado de operações internas, flushes e compacções |

---

## 10. Referências e Leituras Recomendadas

* [Documentação Oficial do Apache Cassandra](https://cassandra.apache.org/doc/latest/)
* [Repositório de Pacotes Debian da Apache](https://debian.cassandra.apache.org/)
* [PEP 594 – Removing Dead Batteries from Python Standard Library](https://peps.python.org/pep-0594/)
* [Claudio Kuenzler: Workaround to use cqlsh on Debian 13 Trixie](https://www.claudiokuenzler.com/blog/1538/workaround-use-cqlsh-command-cassandra-5.x-on-debian-13-trixie)
* [Boas Práticas de Modelagem e Uso do Cassandra no Alpha Engine](/docs/quality_and_testing/cassandra-analisys.md)
