# Proposta e Guia de Implantação: Containers de Sistema Incus para Laboratórios Acadêmicos

---

## 1. Contexto e Justificativa da Proposta

Nos laboratórios de faculdades e universidades, o uso tradicional do Windows impõe um dilema crônico entre **segurança da infraestrutura** e **autonomia pedagógica**:

* **O Problema das Contas Limitadas:** Para evitar alterações indevidas ou infecções por malware, a equipe de TI restringe o acesso dos alunos no Windows. No entanto, cursos de Tecnologia, Engenharia e Ciência da Computação exigem privilégios elevados para instalar compiladores (`gcc`, `rustc`, `clang`), gerenciadores de pacotes globais (`npm -g`, `pip`), ferramentas de redes, bancos de dados e ambientes de virtualização.
* **Prevenção de Exclusão Acidental de Arquivos:** Em sistemas compartilhados com diretórios públicos locais, a falta de isolamento individual resulta na exclusão ou sobrescrita acidental de projetos de outros colegas. Isso causa retrabalho, perda de notas e transtornos frequentes.
* **O Efeito Colateral do "Deep Freeze":** Softwares de congelamento de disco apagam qualquer progresso a cada reinicialização, inviabilizando projetos contínuos que duram várias semanas ao longo do semestre.
* **Sobrecarga de Máquinas Virtuais Convencionais (VirtualBox / VMware):** Cada VM reserva de 2 a 4 GB de memória RAM e inicializa um kernel completo e pesado. Em computadores de laboratório com 8 GB ou 16 GB de RAM, poucas VMs conseguem rodar simultaneamente sem degradar a máquina física.
* **Outro motivo para se utilizar Linux é que a grande maioria dos sistemas utilizam Linux:** Servidores, backbones, dispositivos móveis (Android), supercomputadores e etc. Portanto os alunos estarão em contato com o ambiente que irão encontrar no mercado de trabalho. Por exemplo, no mercado de trabalho, o profissional de TI for hospedar seu projeto em um provedor de nuvem (AWS, Azure, Google, etc.) ele receberá uma instância com o Linux instalado pronto para receber seus sistema. Um aluno que está se formando e irá trabalhar na área de TI, deve estar familiarizado com o ambiente Linux.

### Uma das várias soluções é o Incus.

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
------------------------------

Com a partição dedicada pronta (`/dev/sda4`), inicializamos o Incus apontando diretamente para ela:

```bash
sudo incus admin init
```

Você pode apenas apertar Enter para a maioria dessas perguntas, pois os valores padrão (que ficam dentro dos colchetes [default=...]) são ideais para uma VPS de servidor único.
Como você acabou de instalar o btrfs-progs, recomendo criar um pool de armazenamento do tipo btrfs ou usar o dir (diretório comum) caso o seu sistema de arquivos atual não seja BTRFS.
Siga este roteiro respondendo às perguntas do assistente:

   1. Deseja usar agrupamento? / Would you like to use clustering? (yes/no) [default=no]:
   * Pressione Enter (escolhe no, já que é uma VPS única).
   2. Deseja configurar uma nova pool de armazenamento? / Do you want to configure a new storage pool? (yes/no) [default=yes]:
   * Pressione Enter (escolhe yes, fundamental para resolver o seu erro anterior).
   3. Nome da nova pool de armazenamento / Name of the new storage pool [default=default]:
   * Pressione Enter (define o nome do pool como default).
   4. Nome do backend de armazenamento a usar / Name of the storage backend to use (btrfs, dir, mock) [default=btrfs]:
   * Se o seu disco principal da VPS já for formatado em BTRFS, pressione Enter.
      * Se não tiver certeza ou se for um disco ext4 comum, digite dir e pressione Enter (o tipo dir funciona em absolutamente qualquer VPS sem precisar de partições limpas).
   5. Criar uma nova pool BTRFS? (yes/no) [default=yes]: 
      * Pressione Enter (escolhe yes, fundamental para resolver o seu erro anterior).
   6. Deseja usar um dispositivo de bloco vazio existente (ex. um disco ou partição)?
   7. Tamanho em GiB do novo dispositivo de loop (1GiB minimum) [default=12GiB]: 850 /420   
   8. Would you like to connect to a MAAS server? (yes/no) [default=no]:
   * Pressione Enter.
   9. Deseja criar uma nova ponte de rede local? / Would you like to configure a new local network bridge? (yes/no) [default=yes]:
   * Pressione Enter (para criar a rede interna que dará internet às suas VMs).
   10. Como deve ser chamada a nova ponte? / Name of the new bridge [default=incusbr0]:
   * Pressione Enter.
   11. Que endereço IPv4 deve ser usado? / IPv4 address or fallback to auto [default=auto]:
   * Pressione Enter.
   12. Que endereço IPv6 deve ser usado? / IPv6 address or fallback to auto [default=auto]:
   * Pressione Enter.
   13. O servidor Incus deve estar disponível na rede? / Would you like the Incus server to be available over the network? (yes/no) [default=no]:
   * Pressione Enter (mantém o gerenciamento restrito apenas por dentro da VPS por segurança).
   14. Deseja que o servidor esteja disponível através da rede? / Would you like stale cached images to be updated automatically? (yes/no) [default=yes]
   * Pressione Enter.
   15. Endereço a onde unir (não incluindo o porto) [default=all]:
   16. Porto a onde unir [default=8443]: 
   17. Deseja que as imagens em cache obsoletas seja atualizadas automaticamente? (yes/no) [default=yes]: 
   18. Deseja que a pré-semente "init" do YAML seja escrita? / Would you like a YAML profile show to be printed? (yes/no) [default=no]:
   * Pressione Enter.
   


> **Atenção:** Em máquinas com SSD NVMe, substitua `/dev/sda4` pelo identificador correto (ex: `/dev/nvme0n1p4`). Use o comando `lsblk` para confirmar o nome da partição antes de executar o assistente.

Para permitir que o seu usuário de administração execute comandos do Incus sem precisar de `sudo`:

```bash
sudo usermod -aG incus-admin $USER
newgrp incus-admin
```

