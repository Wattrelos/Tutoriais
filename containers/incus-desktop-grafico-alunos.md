# Ambientes Gráficos Isolados por Aluno com Incus (Desktop + Terminal)

---

## 1. Visão Geral e Propósito

Em laboratórios acadêmicos compartilhados por múltiplos turnos e turmas, o modelo tradicional de contas multiusuário no mesmo sistema operacional cria conflitos insolúveis de portas e dependências. Se um aluno instala e ativa o **Apache2** na porta 80, o colega do turno seguinte não consegue subir o **Nginx** na mesma porta (`Address already in use`), além do risco constante de arquivos e bibliotecas em `/etc/` ou `/usr/` serem sobrescritos.

Com o **Incus**, cada aluno recebe um **Container de Sistema completo**, com sua própria interface gráfica leve (**XFCE4**), terminal nativo, permissão de `root`, endereço IP privativo e pilha de rede exclusiva.

### O Resultado para o Laboratório:
* **Isolamento de Serviços:** O Aluno A roda Apache2 na porta 80, o Aluno B roda Nginx na porta 80 e o Aluno C roda uma API Node.js na porta 3000, sem nenhuma colisão.
* **Experiência de Computador Físico:** Ao sentar na máquina, o aluno tem acesso a uma área de trabalho completa com navegador, terminal gráfico e editor de código.
* **Uso Inteligente de Recursos:** Como há apenas um aluno por máquina física em cada horário, apenas o container daquele aluno permanece ativo, aproveitando até 100% da RAM e CPU disponíveis da máquina real.
* **Economia de Armazenamento:** Graças ao **Btrfs com Copy-on-Write (CoW)**, 30 containers clonados de um modelo base compartilham os mesmos blocos de disco, consumindo espaço apenas para os arquivos que cada aluno criar ou modificar.

---

## 2. Comparativo de Arquitetura

| Critério | Contas no Host Físico (Linux Tradicional) | Máquinas Virtuais Clássicas (VirtualBox) | Containers de Sistema Incus com GUI |
| :--- | :--- | :--- | :--- |
| **Pilha de Rede** | Compartilhada (1 IP, portas disputadas) | Isolada (IP próprio via NAT/Bridge) | 🛡️ **Isolada nativamente (Network Namespace)** |
| **Conflito Apache2 x Nginx** | ❌ **Impossível coexistir na porta 80** |  Não conflitam | ⚡ **Não conflitam (cada um no seu IP/localhost)** |
| **Consumo de RAM em Idle** | ~500 MB (Desktop do Host) | ~2 GB a 4 GB por VM | ⚡ **~250 MB a 350 MB (XFCE + XRDP)** |
| **Tempo de Abertura da GUI** | Imediato no login | ~40 a 60 segundos (boot da VM) | ⚡ **< 2 segundos** |
| **Gerenciamento de Disco** | Partição compartilhada sem cotas simples | Arquivos `.vdi` pesados e estáticos | ⚡ **Btrfs CoW dinâmico e deduplicado** |
| **Segurança para a TI** | Alunos podem quebrar o sistema host | Boa (isolamento KVM) | 🛡️ **Total (Containers Não-Privilegiados)** |

---

## 3. Preparação da Imagem Modelo ("Golden Image")

Vamos criar uma única imagem modelo baseada no Debian 13 chamada `modelo-desktop`. Todas as contas dos alunos serão clones instantâneos desse modelo.

### 3.1. Inicializar o Container Base
Execute no terminal do host Debian físico:

```bash
# Cria e inicia o container com Debian 13
incus launch images:debian/13 modelo-desktop

# Abre o terminal root dentro do container
incus exec modelo-desktop -- bash
```

---

### 3.2. Instalar o Ambiente Gráfico XFCE4 e Utilitários
Dentro do prompt do container `modelo-desktop`, execute os comandos abaixo para instalar a interface gráfica XFCE4 enxuta, o servidor de exibição XRDP e ferramentas essenciais:

```bash
# 1. Atualizar repositórios
apt update && apt upgrade -y

# 2. Instalar XFCE4 básico, Xorg e XRDP (sem pacotes pesados desnecessários)
apt install --no-install-recommends -y \
    xfce4 \
    xfce4-terminal \
    xrdp \
    xorg \
    dbus-x11 \
    x11-xserver-utils

# 3. Instalar aplicativos básicos de desenvolvimento e navegação
apt install -y \
    firefox-esr \
    mousepad \
    sudo \
    curl \
    wget \
    git \
    build-essential \
    net-tools \
    iputils-ping \
    nano \
    vim
```

---

### 3.3. Configurar o XRDP e Sessão Padrão do XFCE
Ainda dentro do container, configure o XRDP para iniciar o XFCE4 por padrão:

```bash
# Define o XFCE como gerenciador de janelas padrão para novos usuários
echo "startxfce4" > /etc/skel/.xsession

# Ajusta permissões do serviço XRDP
adduser xrdp ssl-cert 2>/dev/null || true

# Habilita o serviço XRDP na inicialização do container
systemctl enable xrdp
```

---

### 3.4. Criar o Usuário Padrão do Aluno
Crie a conta de usuário padrão que o aluno utilizará na interface gráfica:

```bash
# Cria o usuário 'aluno' com pasta home e shell bash
useradd -m -s /bin/bash aluno

# Define uma senha padrão didática (ex: aluno123)
echo "aluno:aluno123" | chpasswd

# Adiciona o aluno ao grupo sudo (para poder usar apt install livremente)
usermod -aG sudo aluno

# Permite que o aluno use sudo sem exigir senha dentro do seu container
echo "aluno ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-aluno-init
chmod 0440 /etc/sudoers.d/90-aluno-init

# Garante o arquivo .xsession na pasta do usuário
cp /etc/skel/.xsession /home/aluno/.xsession
chown aluno:aluno /home/aluno/.xsession
```

---

### 3.5. Finalizar e Congelar a Imagem Base
Saia do container para o host e crie a imagem de referência:

```bash
# Sai do container
exit

# Para o container modelo
incus stop modelo-desktop

# Cria um snapshot imutável de referência
incus snapshot create modelo-desktop base

# (Opcional) Publica como imagem local reutilizável
incus publish modelo-desktop --alias template-desktop-lab description="Modelo Debian 13 Desktop XFCE4 para Alunos"
```

---

## 4. Provisionamento dos Ambientes dos Alunos

Com a imagem modelo pronta, provisionar novos ambientes leva menos de 2 segundos graças ao Btrfs Copy-on-Write.

### 4.1. Criar Containers para Dois Alunos de Exemplo

```bash
# Cria o container para o aluno João (Turno Manhã)
incus copy modelo-desktop aluno-joao
incus config set aluno-joao limits.cpu=2 limits.memory=3GiB
incus start aluno-joao

# Cria o container para o aluno Maria (Turno Noite)
incus copy modelo-desktop aluno-maria
incus config set aluno-maria limits.cpu=2 limits.memory=3GiB
incus start aluno-maria
```

### 4.2. Verificar IPs Privativos Gerados
```bash
incus list -c n,s,4
```

*Saída esperada:*
```text
+-------------+---------+--------------------+
|    NAME     | STATUS  |        IPV4        |
+-------------+---------+--------------------+
| aluno-joao  | RUNNING | 10.0.100.25 (eth0) |
| aluno-maria | RUNNING | 10.0.100.26 (eth0) |
+-------------+---------+--------------------+
```

Observe que cada container possui seu **próprio endereço IP privativo** na ponte de rede local (`incusbr0`).

---

## 5. Como o Aluno Acessa a Interface Gráfica na Máquina Física

Na máquina física do laboratório, precisamos apenas de um visualizador RDP ultraleve (**FreeRDP** ou **Remmina**) instalado no host Debian.

### 5.1. Instalar o Cliente RDP no Host Físico
No Debian da máquina física:

```bash
sudo apt install -y freerdp3-x11 remmina
```

### 5.2. Conexão Imediata via Linha de Comando
Para abrir a interface gráfica do container em tela cheia na máquina física:

```bash
# Obtém dinamicamente o IP do container do aluno
IP_ALUNO=$(incus list aluno-joao -c 4 --format csv | awk '{print $1}')

# Conecta em tela cheia com áudio e área de transferência integrados
xfreerdp /v:$IP_ALUNO /u:aluno /p:aluno123 /f /dynamic-resolution +clipboard /sound
```

> **Resultado Visual:** A tela do monitor físico é ocupada 100% pela área de trabalho do XFCE4 rodando de dentro do container do João. Ele tem menu Iniciar, barra de tarefas, área de trabalho, terminal gráfico e navegador.

---

## 6. Automação do Ciclo de Vida: Login e Logout no Host

Para que o aluno não precise digitar comandos do Incus, configuramos um script de sessão transparente.

### 6.1. Script de Inicialização da Sessão (`/usr/local/bin/iniciar-ambiente-aluno.sh`)
Crie este script no host físico:

```bash
sudo nano /usr/local/bin/iniciar-ambiente-aluno.sh
```

Conteúdo do script:

```bash
#!/bin/bash
# Script de login transparente para laboratório com Incus
# Recebe o nome do aluno como argumento ou usa o usuário logado no host

USUARIO_HOST="${1:-$USER}"
CONTAINER="aluno-${USUARIO_HOST}"

# 1. Verifica se o container do aluno existe
if ! incus info "$CONTAINER" >/dev/null 2>&1; then
    zenity --error --text="Ambiente para $USUARIO_HOST não encontrado. Procure o professor ou TI."
    exit 1
fi

# 2. Exibe notificação de inicialização
zenity --info --title="Laboratório Virtual" --text="Iniciando seu ambiente dedicado...\nAguarde 2 segundos." --timeout=2 &

# 3. Garante que o container esteja iniciado
incus start "$CONTAINER" 2>/dev/null || true

# 4. Aguarda a atribuição do endereço IP
for i in {1..10}; do
    IP=$(incus list "$CONTAINER" -c 4 --format csv | awk '{print $1}')
    [ -n "$IP" ] && break
    sleep 0.5
done

if [ -z "$IP" ]; then
    zenity --error --text="Falha ao obter endereço de rede do ambiente."
    exit 1
fi

# 5. Lança o ambiente gráfico em tela cheia
# Ao fechar a janela ou clicar em Sair do XFCE, a conexão encerra
xfreerdp /v:"$IP" /u:aluno /p:aluno123 /f /dynamic-resolution +clipboard /sound

# 6. Ao encerrar a sessão, desliga o container para liberar toda a memória RAM
incus stop "$CONTAINER"
```

Torne o script executável:
```bash
sudo chmod +x /usr/local/bin/iniciar-ambiente-aluno.sh
```

---

## 7. Demonstração Prática: O "Teste de Fogo" (Apache2 vs Nginx)

Para validar que o isolamento é absoluto e elimina qualquer conflito de serviços de rede:

### Cenário 1: Aluno João instala o Apache2 (Turno da Manhã)
1. João inicia seu container `aluno-joao`.
2. No terminal gráfico do XFCE, ele digita:
   ```bash
   sudo apt update
   sudo apt install -y apache2
   echo "<h1>Ambiente do Joao: Servidor Apache2 Ativo na Porta 80</h1>" | sudo tee /var/www/html/index.html
   ```
3. Ele abre o Firefox dentro do seu desktop e acessa:
   `http://localhost`
   * **Resultado:** A página do Apache2 carrega perfeitamente na porta padrão 80.
4. João desloga do sistema. O script encerra o container dele com `incus stop aluno-joao`.

---

### Cenário 2: Aluna Maria instala o Nginx (Turno da Noite)
1. Maria senta na mesma máquina física e inicia seu container `aluno-maria`.
2. No terminal gráfico do XFCE, ela digita:
   ```bash
   sudo apt update
   sudo apt install -y nginx
   echo "<h1>Ambiente da Maria: Servidor NGINX Ativo na Porta 80</h1>" | sudo tee /var/www/html/index.nginx-debian.html
   ```
3. Ela abre o Firefox dentro do seu desktop e acessa:
   `http://localhost`
   * **Resultado:** O Nginx carrega com sucesso imediato na porta padrão 80.
   * **Sem erro de porta ocupada:** O fato de o Apache2 do João estar instalado e configurado na porta 80 não afeta em nada a Maria, pois pertencem a Namespaces de rede e sistemas de arquivos totalmente separados.

---

## 8. Manutenção e Restauração Instantânea (Professor e TI)

### 8.1. Restaurar um Ambiente Corrompido
Se um aluno danificar bibliotecas do sistema ou quebrar o ambiente gráfico:

```bash
# 1. Para o container do aluno
incus stop aluno-joao

# 2. Restaura o estado inicial limpo em menos de 2 segundos
incus snapshot restore aluno-joao base

# 3. Reinicia o ambiente pronto para uso
incus start aluno-joao
```

### 8.2. Script em Lote para Criar Turmas Inteiras (`criar-turma-gui.sh`)
Para provisionar 35 alunos de uma vez antes do início das aulas:

```bash
#!/bin/bash
TURMA="turma-a-2026"
TOTAL=35

for i in $(seq -w 1 $TOTAL); do
    NOME="aluno-${TURMA}-${i}"
    echo "Provisionando $NOME..."
    
    # Clona a imagem modelo instantaneamente
    incus copy modelo-desktop "$NOME"
    
    # Aplica cotas de recursos (2 Cores, 3 GB RAM)
    incus config set "$NOME" limits.cpu=2 limits.memory=3GiB
    
    # Snapshot inicial de segurança
    incus snapshot create "$NOME" base
done

echo "Turma $TURMA provisionada com sucesso!"
```

---

## 9. Conclusão

A utilização do Incus com interface gráfica entrega o melhor dos dois mundos:
1. **Autonomia Pedagógica Total:** Cada estudante tem privilégios de `root`, ambiente gráfico completo, liberdade para instalar qualquer stack (Apache, Nginx, Docker interno, bancos de dados) sem interferir nos colegas.
2. **Eficiência de Infraestrutura:** Elimina o peso insustentável de dezenas de Máquinas Virtuais pesadas (VirtualBox/VMware), aproveita o poder do Btrfs para economizar centenas de gigabytes em disco e garante o desligamento automático de containers inativos para liberar a memória RAM da máquina física.
