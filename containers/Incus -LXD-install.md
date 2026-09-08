# Proposta e Guia de Implantação: Containers de Sistema Incus para Laboratórios Acadêmicos

---

## 1. Contexto e Justificativa da Proposta

Nos laboratórios de faculdades e universidades, o uso tradicional do Windows impõe um dilema crônico entre **segurança da infraestrutura** e **autonomia pedagógica**:

* **O Problema das Contas Limitadas:** Para evitar alterações indevidas ou infecções por malware, a equipe de TI restringe o acesso dos alunos no Windows. No entanto, cursos de Tecnologia, Engenharia e Ciência da Computação exigem privilégios elevados para instalar compiladores (`gcc`, `rustc`, `clang`), gerenciadores de pacotes globais (`npm -g`, `pip`), ferramentas de redes, bancos de dados e ambientes de virtualização.
* **Prevenção de Exclusão Acidental de Arquivos:** Em sistemas compartilhados com diretórios públicos locais, a falta de isolamento individual resulta na exclusão ou sobrescrita acidental de projetos de outros colegas. Isso causa retrabalho, perda de notas e transtornos frequentes.
* **O Efeito Colateral do "Deep Freeze":** Softwares de congelamento de disco apagam qualquer progresso a cada reinicialização, inviabilizando projetos contínuos que duram várias semanas ao longo do semestre.
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
   * Como os containers compartilham o kernel do host com isolamento por *cgroups* e *namespaces*, uma máquina com 16 GB de RAM pode hospedar dezenas de containers simultâneos com baixíssima latência.
3. **Snapshots e Restauração Instantânea (Copy-on-Write):**
   * Usando sistemas de arquivos modernos como **Btrfs** ou **ZFS**, criar um snapshot ou restaurar um container quebrado leva menos de 2 segundos. Se um aluno desconfigurar o sistema antes de uma prova prática, o ambiente é recuperado imediatamente.
4. **Controle Estrito de Recursos:**
   * A TI pode definir limites máximos de CPU, memória RAM e armazenamento por aluno, impedindo que um loop infinito congele a estação de trabalho física.

---

## 3. Preparação do Hardware e Particionamento do Disco (Host Debian)

Para garantir o melhor desempenho e estabilidade em computadores com **16 GB de RAM** e **500 GB+ de HDD ou SSD**, a preparação do sistema operacional hospedeiro (Debian) deve seguir um esquema de particionamento planejado.

### 3.1. Configurações Prévias na BIOS / UEFI
1. Acesse o setup da placa-mãe ao ligar o computador (teclas `F2`, `F12` ou `Del`).
2. **Habilitar Virtualização:** Ative a tecnologia de virtualização do processador (**Intel VT-x** ou **AMD-V / SVM**). Embora containers compartilhem o kernel, essa opção permite ao Incus executar também Máquinas Virtuais completas se necessário em aulas futuras.
3. **Modo de Inicialização:** Certifique-se de que o modo de boot esteja configurado como **UEFI** com tabela de partição **GPT**.

---

### 3.2. Estratégia de Particionamento (Disco de 500 GB)

> **Regra de Ouro da Arquitetura:** Nunca utilize uma partição única ext4 para o sistema e os containers via "loop file" em ambientes de produção/laboratório. Criar uma **partição dedicada exclusiva para o Storage Pool do Incus** traz duas vantagens decisivas:
> 1. **Blindagem contra estouro de disco:** Se os containers dos alunos encherem o disco com downloads ou compilações, a partição raiz (`/`) do Debian não será afetada. O computador continua ligando e acessível para a equipe de TI.
> 2. **Performance Nativa Copy-on-Write:** O Incus formata a partição diretamente em **Btrfs** ou **ZFS**, garantindo velocidade máxima de I/O em snapshots e clonagens.

#### Tabela Recomendada de Particionamento (500 GB):

| Partição | Ponto de Montagem | Sistema de Arquivos | Tamanho Sugerido | Finalidade |
| :--- | :--- | :--- | :--- | :--- |
| **`/dev/sda1`** | `/boot/efi` | FAT32 | **1 GB** (1024 MB) | Inicialização UEFI padrão (ESP). |
| **`/dev/sda2`** | `/` (raiz) | ext4 | **70 GB** | Sistema operacional Debian, drivers, ambiente gráfico leve e logs do host. |
| **`/dev/sda3`** | `[swap]` | swap | **16 GB** | Área de troca para prevenção de OOM (*Out of Memory*) em picos de compilação. |
| **`/dev/sda4`** | *Nenhum (Não montar)* | **Btrfs** *(gerenciado pelo Incus)* | **Restante (~413 GB)** | **Storage Pool dedicado exclusivo do Incus** (onde ficam os containers dos alunos). |

*(Nota: Em discos NVMe, os nomes das partições serão `/dev/nvme0n1p1`, `/dev/nvme0n1p2`, etc.)*

---

### 3.3. Passo a Passo no Instalador do Debian (Netinstall)

1. No menu do instalador do Debian, escolha a opção **Instalação Gráfica (Graphical Install)** ou **Install**.
2. Prossiga com idioma, teclado e rede até chegar na tela **Particionamento de Discos**.
3. Selecione o método: **Manual**.
4. Selecione o disco principal (ex: `/dev/sda` ou `/dev/nvme0n1`) e crie a nova tabela de partição (GPT).
5. Crie as partições conforme a tabela:
   * **Partição 1 (1 GB):** Tipo primária, usar como: *Partição de Sistema EFI*.
   * **Partição 2 (70 GB):** Tipo primária, usar como: *Sistema de arquivos com "journaling" ext4*, ponto de montagem: `/`.
   * **Partição 3 (16 GB):** Tipo primária, usar como: *Área de troca (swap)*.
   * **Partição 4 (Restante ~413 GB):** Crie a partição com todo o espaço livre restante. Na opção **"Usar como"**, escolha **"Não usar esta partição"** (ou deixe sem formatação/sem ponto de montagem).
6. Conclua o particionamento e confirme a gravação das mudanças no disco.
7. Na tela de **Seleção de Softwares (tasksel)**:
   * Se a máquina for apenas um servidor acessado pela rede: marque apenas **Utilitários padrão do sistema** e **Servidor SSH**.
   * Se for uma estação de laboratório com monitor/teclado: marque **Ambiente de área de trabalho do Debian** e escolha **XFCE** ou **LXQt** (interfaces extremamente leves que consomem menos de 600 MB de RAM, deixando mais de 15 GB livres para os containers).

---

### 3.4. Otimizações do Host para Máquinas com 16 GB de RAM

Após o primeiro boot no Debian recém-instalado, abra o terminal como root (`sudo`) e aplique duas otimizações essenciais:

#### 1. Ajustar o uso do Swap (Swappiness)
Como a máquina possui 16 GB de RAM física, o Linux deve priorizar a memória rápida e usar o disco apenas em emergências:
```bash
# Reduz a tendência de uso do swap de 60 para 10
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-lab-performance.conf
sudo sysctl --system
```

#### 2. Ativar compressão de memória ZRAM (Multiplicador de RAM)
O pacote `zram-tools` cria um bloco de swap compactado dinamicamente na própria memória RAM usando o algoritmo `zstd`. Na prática, faz os 16 GB de RAM renderem como **24 a 32 GB** de capacidade útil para containers:
```bash
sudo apt update && sudo apt install -y zram-tools
# O zram é ativado automaticamente pelo systemd
sudo zramctl
```

---

## 4. Instalação do Incus no Debian

O Incus pode ser instalado nativamente no **Debian 13 (Trixie)** ou via repositório oficial Zabbly no **Debian 12 (Bookworm)**.

### Opção A: No Debian 13 (Trixie / Testing) - Repositório Oficial

O Debian 13 já inclui os pacotes do Incus em seus repositórios oficiais:

```bash
# 1. Atualize a lista de pacotes
sudo apt update && sudo apt upgrade -y

# 2. Instale o Incus e os utilitários do sistema de arquivos Btrfs
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
## 3. Decisão
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

## 5. Configuração Inicial do Servidor (`incus admin init`)

Com a partição dedicada pronta (`/dev/sda4`), inicializamos o Incus apontando diretamente para ela:

```bash
sudo incus admin init
```



> **Atenção:** Em máquinas com SSD NVMe, substitua `/dev/sda4` pelo identificador correto (ex: `/dev/nvme0n1p4`). Use o comando `lsblk` para confirmar o nome da partição antes de executar o assistente.

Para permitir que o seu usuário de administração execute comandos do Incus sem precisar de `sudo`:

```bash
sudo usermod -aG incus-admin $USER
newgrp incus-admin
```

---

## 6. Criação do Container Modelo (Golden Image)

Em vez de baixar e configurar um container do zero para cada aluno, criamos uma imagem modelo com as ferramentas de aula pré-instaladas.

```bash
# 1. Cria e inicializa um container base Debian
incus launch images:debian/13 modelo-lab

# 2. Acessa o shell do container modelo
incus exec modelo-lab -- bash
```

Dentro do container `modelo-lab`, instale as ferramentas necessárias para as aulas:

```bash
# Atualiza pacotes internos
apt update && apt upgrade -y

# Instala ferramentas essenciais de desenvolvimento
apt install -y build-essential gdb git curl wget nano vim sudo python3 python3-pip python3-venv htop net-tools

# Cria uma senha para o root do container (opcional)
echo "root:aluno123" | chpasswd

# Sai do container
exit
```

Com o container configurado, pare-o e crie um **snapshot de referência**:

```bash
# Para o container
incus stop modelo-lab

# Cria um snapshot base que servirá de ponto de restauração instantâneo
incus snapshot create modelo-lab base
```

---

## 7. Métodos de Acesso para os Alunos

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
# Captura o usuário real que invocou o sudo (ex: aluno01)
CONTAINER="${SUDO_USER:-$USER}"

# Garante que o container esteja iniciado
incus start "$CONTAINER" 2>/dev/null

# Transfere a sessão exportando o SHELL explicitamente para evitar falhas em emuladores como o Konsole
exec incus exec "$CONTAINER" --env SHELL=/bin/bash -- /bin/bash --login
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

> [!TIP]
> **Dica para Testes e Diagnóstico:** Se o terminal fechar instantaneamente durante os testes iniciais, comente temporariamente a linha `exit` em `/home/aluno01/.bashrc` (`sudo sed -i 's/^exit/#exit/' /home/aluno01/.bashrc`) para visualizar eventuais mensagens de erro de inicialização. Para mais detalhes e soluções de problemas conhecidos com emuladores de terminal (Konsole, xterm), consulte o guia [Incus-troubleshooting.md](Incus-troubleshooting.md).

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

## 8. Controle de Recursos e Quotas (cgroups)

Para evitar que scripts com loops infinitos travem a máquina física compartilhada, configure limites rígidos por container:

```bash
# Limita o uso de CPU a 2 núcleos
incus config set aluno01 limits.cpu=2

# Limita o uso de memória RAM a 2 GiB
incus config set aluno01 limits.memory=2GiB

# Força a interrupção de processos que estourarem a memória (evita swap excessivo)
incus config set aluno01 limits.memory.enforce=hard

# Limita o espaço em disco do container a 15 GiB (em pools Btrfs ou ZFS)
incus config device set aluno01 root size=15GiB
```

---

## 9. Guia Prático de Manutenção para Professores e Técnicos

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

## 10. Conclusão da Proposta

A adoção do **Incus com Debian** transforma o laboratório acadêmico:
1. **Para os Alunos:** Liberdade total com acesso `root`, projetos contínuos sem perdas por congelamento de disco, isolamento contra exclusão acidental de arquivos de terceiros e aprendizado de Linux real em padrão de mercado.
2. **Para a TI:** Garantia inegociável de segurança com containers não-privilegiados, partição dedicada que impede travamento do host por disco cheio, isolamento de recursos por cgroups e sem risco de contaminação da rede física.
3. **Para os Professores:** Agilidade nas aulas, padronização de imagens didáticas e restauração instantânea de qualquer ambiente danificado em segundos.
