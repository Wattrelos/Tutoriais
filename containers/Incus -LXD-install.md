# Proposta e Guia de Implantação: Containers de Sistema Incus para Laboratórios Acadêmicos

---

## 1. Contexto e Justificativa da Proposta

Nos laboratórios de faculdades e universidades, o uso tradicional do Windows impõe um dilema crônico entre **segurança da infraestrutura** e **autonomia pedagógica**:

* **O Problema das Contas Limitadas:** Para evitar alterações indevidas ou infecções por malware, a equipe de TI restringe o acesso dos alunos no Windows. No entanto, cursos de Tecnologia, Engenharia e Ciência da Computação exigem privilégios elevados para instalar compiladores (`gcc`, `rustc`, `clang`), gerenciadores de pacotes globais (`npm -g`, `pip`), ferramentas de redes, bancos de dados e ambientes de virtualização.
* **O Efeito Colateral do "Deep Freeze":** Softwares de congelamento de disco apagam qualquer progresso a cada reinicialização, impossibilitando projetos contínuos ao longo das semanas de aula.
* **Sobrecarga de Máquinas Virtuais Convencionais (VirtualBox / VMware):** Cada VM reserva de 2 a 4 GB de memória RAM e inicializa um kernel completo e pesado. Em computadores de laboratório com 8 GB ou 16 GB de RAM, poucas VMs conseguem rodar simultaneamente sem degradar a máquina física.

### Por que o Incus é a Solução Ideal?

O **Incus** é o gerenciador comunitário de containers de sistema mantido pelo projeto *Linux Containers* (os criadores originais do LXC/LXD). Ao contrário de containers de aplicação (como o Docker), o Incus entrega uma **máquina Linux completa** com seu próprio init (`systemd`), serviços em background, rede própria e gerenciamento de processos.

| Característica | Windows com Conta Restrita | Máquina Virtual (VirtualBox) | Docker | Containers de Sistema (Incus) |
| :--- | :--- | :--- | :--- | :--- |
| **Acesso Root para o Aluno** | ❌ Não (bloqueado pela TI) |  Sim (dentro da VM) | ⚠️ Sim, mas sem `systemd` |  **Sim (root isolado)** |
| **Consumo de RAM em Idle** | Alto (> 2.5 GB) | Alto (~1 a 2 GB por VM) | Baixo (~20 MB) | ⚡ **Muito Baixo (~30 a 60 MB)** |
| **Tempo de Inicialização** | ~40 a 60 segundos | ~30 a 50 segundos | < 1 segundo | ⚡ **< 1 segundo** |
| **Segurança para o Host** | Frágil se for administrador | Alta (isolamento por hardware) | Média (acesso ao daemon) | 🛡️ **Máxima (*User Namespaces*)** |
| **Rollback / Restauração** | Demorado (reimagem) | Médio (snapshots pesados) | Não aplicável | ⚡ **Instantâneo (< 2 segundos)** |
| **Custo de Licenciamento** | Alto (licenças Microsoft) | Gratuito / Comercial | Gratuito |  **100% Livre e Open Source** |

---

## 2. Os Quatro Pilares para Convencer a Coordenação e a TI

1. **Root Seguro via *Unprivileged Containers* (Garantia da TI):**
   * O Incus opera por padrão com containers não-privilegiados.
   * O usuário `root` (UID 0) do container é mapeado para um UID sem privilégios no host (ex: UID 1000000).
   * **Consequência prática:** Mesmo que o aluno execute `rm -rf --no-preserve-root /` ou tente alterar o hardware, **ele não consegue afetar o Debian físico nem outros containers**. O host permanece 100% blindado.
2. **Alta Densidade e Desempenho:**
   * Como os containers compartilham o kernel do host com isolamento por *cgroups* e *namespaces*, uma única máquina com 16 GB de RAM pode hospedar dezenas de containers simultâneos sem perda de fluidez.
3. **Snapshots e Restauração Instantânea (Copy-on-Write):**
   * Usando sistemas de arquivos modernos como **ZFS** ou **Btrfs**, criar um snapshot ou restaurar um container quebrado leva menos de 2 segundos. Se um aluno desconfigurar o sistema antes de uma prova prática, o ambiente é recuperado imediatamente.
4. **Controle Estrito de Recursos:**
   * A TI pode definir limites máximos de CPU, memória RAM e armazenamento por aluno, impedindo que um loop infinito (ou vazamento de memória) congele a estação de trabalho.

---

## 3. Instalação do Incus no Debian

O Incus pode ser instalado nativamente no **Debian 13 (Trixie)** ou via repositório oficial Zabbly no **Debian 12 (Bookworm)**.

### Opção A: No Debian 13 (Trixie / Testing) - Repositório Oficial

O Debian 13 já inclui os pacotes do Incus em seus repositórios oficiais:

```bash
# 1. Atualize a lista de pacotes
sudo apt update && sudo apt upgrade -y

# 2. Instale o Incus e os utilitários de sistema de arquivos (Btrfs ou ZFS)
# Recomendado: btrfs-progs (nativo no kernel) ou zfsutils-linux
sudo apt install -y incus btrfs-progs
```

### Opção B: No Debian 12 (Bookworm) - Repositório Oficial Zabbly

Caso o servidor ou estações do laboratório estejam no Debian 12 Stable:

```bash
# 1. Instale ferramentas básicas
sudo apt update && sudo apt install -y curl ca-certificates

# 2. Adicione a chave GPG oficial mantida pelos desenvolvedores do Incus (Zabbly)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.zabbly.com/key.asc -o /etc/apt/keyrings/zabbly.asc

# 3. Adicione o repositório estável do Incus
sudo sh -c 'cat <<EOF > /etc/apt/sources.list.d/zabbly-incus-stable.sources
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc
EOF'

# 4. Atualize o cache e instale o Incus
sudo apt update
sudo apt install -y incus btrfs-progs
```

---

## 4. Configuração Inicial do Servidor (`incus admin init`)

A inicialização e o provisionamento dos pools de armazenamento e rede são realizados através do assistente administrativo:

```bash
sudo incus admin init
```

### Respostas recomendadas durante o assistente:

```text
Would you like to use Incus clustering? (default: no): no
Do you want to configure a new storage pool? (default: yes): yes
Name of the new storage pool [default: default]: default
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default: btrfs]: btrfs
Create a new BTRFS pool? (default: yes): yes
Would you like to use an existing empty block device (e.g. a partition)? (default: no): no
Size of the loop device in GiB (1GiB minimum) [default: 30GiB]: 50GiB
Would you like to connect to a MAAS server? (default: no): no
Would you like to create a new local network bridge? (default: yes): yes
What should the new bridge be named? [default: incusbr0]: incusbr0
What IPv4 address should be used? (CIDR subnet, "auto" or "none") [default: auto]: auto
What IPv6 address should be used? (CIDR subnet, "auto" or "none") [default: auto]: none
Would you like the Incus server to be available over the network? (default: no): no
Would you like stale cached images to be updated automatically? (default: yes): yes
Would you like a YAML "init" preseed to be printed? (default: no): no
```

> **Dica de Storage:** Se sua máquina já tiver uma partição formatada em Btrfs ou um pool ZFS existente, você pode apontar o storage pool diretamente para ela para máxima performance.

Para permitir que o seu usuário de administração execute comandos do Incus sem precisar de `sudo`:

```bash
sudo usermod -aG incus-admin $USER
newgrp incus-admin
```

---

## 5. Criação do Container Modelo (Golden Image)

Em vez de baixar e configurar um container do zero para cada aluno, criamos uma imagem modelo com as ferramentas de aula pré-instaladas.

```bash
# 1. Cria e inicializa um container base Debian
incus launch images:debian/12 modelo-lab

# 2. Acessa o shell do container modelo
incus exec modelo-lab -- bash
```

Dentro do container `modelo-lab`, instale as ferramentas necessárias para as aulas:

```bash
# Atualiza pacotes internos
apt update && apt upgrade -y

# Instala ferramentas essenciais de desenvolvimento
apt install -y build-essential gdb git curl wget nano vim sudo python3 python3-pip python3-venv htop net-tools

# Cria um usuário padrão para o aluno (opcional) ou mantém o root acessível
echo "root:aluno123" | chpasswd

# Sai do container
exit
```

Com o container configurado, pare-o e crie um **snapshot de referência**:

```bash
# Para o container
incus stop modelo-lab

# Cria um snapshot base que servirá de ponto de restauração
incus snapshot create modelo-lab base
```

---

## 6. Métodos de Acesso para os Alunos

Aqui resolvemos o desafio de **conectar o aluno ao seu container com root, sem dar acesso root ao host físico**.

### Método 1: Login Direto no Terminal Local (Ideal para Estações de Laboratório)

Neste modelo, o aluno faz login na máquina física com seu usuário normal, mas ao abrir o terminal, ele entra imediatamente no seu container com privilégios de `root`:

#### 1. Criar o container individual a partir do modelo
```bash
# Clona o modelo instantaneamente (em menos de 1 segundo via Copy-on-Write)
incus copy modelo-lab aluno01
incus start aluno01
```

#### 2. Criar um script de entrada restrita no host
Crie o arquivo `/usr/local/bin/entrar-container.sh`:
```bash
sudo tee /usr/local/bin/entrar-container.sh << 'EOF'
#!/bin/bash
CONTAINER="$USER"

# Garante que o container esteja iniciado
incus start "$CONTAINER" 2>/dev/null

# Transfere a sessão diretamente para o root do container
exec incus exec "$CONTAINER" -- /bin/bash --login
EOF

sudo chmod 755 /usr/local/bin/entrar-container.sh
```

#### 3. Configurar permissão segura no Sudoers
Adicione uma regra em `/etc/sudoers.d/alunos-incus` para permitir que os alunos executem apenas este script com privilégios administrativos:
```bash
sudo tee /etc/sudoers.d/alunos-incus << 'EOF'
# Permite que qualquer usuário execute o script de entrada sem senha
ALL ALL=(ALL) NOPASSWD: /usr/local/bin/entrar-container.sh
EOF
```

#### 4. Criar a conta do aluno no host com inicialização automática
```bash
# Cria o usuário aluno01 no Debian host
sudo adduser --gecos "" --disabled-login aluno01
echo "aluno01:senhaAluno123" | sudo chpasswd

# Configura o shell ou o .bashrc para chamar o script automaticamente
echo "sudo /usr/local/bin/entrar-container.sh" | sudo tee -a /home/aluno01/.bashrc
echo "exit" | sudo tee -a /home/aluno01/.bashrc
```

> **Resultado:** Quando o `aluno01` faz login no Debian do laboratório, o terminal cai diretamente dentro do seu container com `root`. Ao digitar `exit`, ele é desconectado da máquina física. Ele não tem acesso ao sistema de arquivos do host!

---

### Método 2: Acesso Remoto via SSH ou Rede Interna

Se os alunos acessam um servidor central de laboratório ou utilizam seus próprios notebooks:

#### 1. Instalar o OpenSSH dentro do container
```bash
incus exec aluno01 -- apt install -y openssh-server
incus exec aluno01 -- systemctl enable --now ssh
incus exec aluno01 -- sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
incus exec aluno01 -- systemctl restart ssh
```

#### 2. Mapear uma porta específica para cada aluno (Proxy Device)
O Incus possui um recurso nativo de proxy reverso de rede:
```bash
# Redireciona a porta 2201 da máquina host para a porta 22 do container aluno01
incus config device add aluno01 ssh-proxy proxy listen=tcp:0.0.0.0:2201 connect=tcp:127.0.0.1:22
```

O aluno se conecta de qualquer lugar da rede da faculdade (ou via VS Code Remote SSH):
```bash
ssh root@<IP_DA_MAQUINA_HOST> -p 2201
```

---

## 7. Controle de Recursos e Quotas (cgroups)

Para evitar que scripts com loops infinitos travem a máquina física compartilhada, configure limites rígidos por container:

```bash
# Limita o uso de CPU a 2 núcleos
incus config set aluno01 limits.cpu=2

# Limita o uso de memória RAM a 2 GiB
incus config set aluno01 limits.memory=2GiB

# Força a interrupção de processos que estourarem a memória (evita swap excessivo)
incus config set aluno01 limits.memory.enforce=hard

# Limita o espaço em disco do container a 15 GiB (em pools ZFS ou Btrfs)
incus config device set aluno01 root size=15GiB
```

---

## 8. Guia Prático de Manutenção para Professores e Técnicos

### Restaurar o Container Quebrado em 2 Segundos
Se o aluno excluir arquivos críticos do sistema ou quebrar dependências:
```bash
incus stop aluno01
incus snapshot restore aluno01 base
incus start aluno01
```

### Script de Automação para Criar Turmas Inteiras
Crie um script `/usr/local/bin/criar-turma.sh`:
```bash
#!/bin/bash
TURMA="so2026"
TOTAL_ALUNOS=30

for i in $(seq -w 1 $TOTAL_ALUNOS); do
    NOME="aluno-${TURMA}-${i}"
    echo "Provisionando container para $NOME..."
    
    # 1. Clona a partir do modelo base
    incus copy modelo-lab "$NOME"
    
    # 2. Aplica limites de recursos
    incus config set "$NOME" limits.cpu=2 limits.memory=2GiB
    
    # 3. Inicia o container
    incus start "$NOME"
    
    # 4. Tira o snapshot inicial para rollback
    incus snapshot create "$NOME" base
done

echo "Todos os $TOTAL_ALUNOS containers foram provisionados com sucesso!"
```

### Limpeza ao Final do Semestre
Para desalocar os recursos de todos os containers de uma turma:
```bash
incus delete --force aluno-so2026-01 aluno-so2026-02 aluno-so2026-03
```
Ou listar e remover em lote:
```bash
incus list -c n --format csv | grep 'aluno-so2026' | xargs -r incus delete --force
```

---

## 9. Conclusão da Proposta

A adoção do **Incus com Debian** transforma o laboratório acadêmico:
1. **Para os Alunos:** Liberdade total com acesso `root`, aprendizado de Linux real em padrão de mercado e sem bloqueios arbitrários.
2. **Para a TI:** Garantia inegociável de segurança com containers não-privilegiados, isolamento de recursos por cgroups e sem risco de contaminação da rede física.
3. **Para os Professores:** Agilidade nas aulas, padronização de imagens didáticas e restauração instantânea de qualquer ambiente danificado em segundos.
