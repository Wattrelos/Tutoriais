# Ambientes Gráficos Isolados por Aluno com Incus (KDE Plasma Wayland + Autologin)

---

## 1. Visão Geral e Propósito

Em laboratórios acadêmicos compartilhados por múltiplos turnos e turmas, o modelo tradicional de contas multiusuário compartilhando o mesmo sistema operacional cria conflitos insolúveis de portas e dependências. Se um aluno instala e ativa o **Apache2** na porta 80, o colega do turno seguinte não consegue subir o **Nginx** na mesma porta (`Address already in use`), além do risco constante de arquivos de projetos, bibliotecas em `/usr/` ou configurações em `/etc/` serem sobrescritos.

### O Novo Cenário do Laboratório:
* **Uma Instância Isolada por Aluno:** Cada estudante possui seu próprio ambiente isolado rodando localmente na máquina física do laboratório.
* **Interface Gráfica Moderna (KDE Plasma em Wayland):** O aluno tem acesso a um desktop gráfico completo, moderno e fluido com **KDE Plasma 6 / 5.27** rodando nativamente sobre **Wayland**.
* **Aproveitamento Total do Hardware:** Como há apenas um estudante por computador físico em cada horário, a instância daquele aluno pode usufruir de até 100% da CPU, memória RAM e aceleração gráfica da máquina real.
* **Autologin Transparente:** O aluno digita seu usuário e senha no gerenciador de login do computador físico (**SDDM**). Ao carregar o ambiente do aluno, o login na interface gráfica KDE Wayland é automático (**autologin**), sem que ele precise redigitar senhas.
* **Desligamento e Liberação Automática de Recursos:** Ao terminar a aula, o aluno simplesmente clica em "Encerrar Sessão" ou "Desligar" pelo próprio menu do KDE. O ambiente é desligado no Incus, liberando imediatamente toda a memória RAM da máquina física e retornando o computador para a tela de login do SDDM, pronto para o próximo turno.

---

## 2. Decisão de Arquitetura: Por que Incus VM (`--vm`) para Wayland?

O **Incus** oferece suporte tanto a **Containers de Sistema (LXC)** quanto a **Máquinas Virtuais KVM (`--vm`)**. Para este cenário específico com **KDE Plasma em Wayland nativo**, a escolha técnica ideal é a **Incus VM**:

| Critério | Container Incus (LXC) com XRDP | Máquina Virtual Convencional (VirtualBox) | Incus VM com KDE Wayland (Esta Solução) |
| :--- | :--- | :--- | :--- |
| **Servidor Gráfico** | X11 clássico (XRDP não suporta Wayland nativo) | Emulado / Driver proprietário | ⚡ **Wayland nativo (KWin Wayland + virtio-gpu)** |
| **Console Gráfico** | Depende de RDP em rede local | Janela pesada do VirtualBox | ⚡ **Nativo via SPICE (`incus console --type=vga`)** |
| **Autologin Gráfico** | Complexo / Emulado via `.xsession` | Configuração manual em cada VM | ⚡ **Nativo via SDDM interno (`autologin.conf`)** |
| **Tempo de Boot** | < 1 segundo | 40 a 60 segundos | ⚡ **~3 a 5 segundos** |
| **Economia de Disco** | Btrfs Copy-on-Write (CoW) | Arquivos `.vdi` pesados e estáticos | ⚡ **Btrfs CoW (clones instantâneos em < 2s)** |
| **Consumo no Host** | Host precisa de cliente FreeRDP | Host roda app pesada VirtualBox | 🛡️ **Host mínimo (apenas SDDM + virt-viewer)** |

> [!NOTE]
> O comando `incus console <instancia> --type=vga` (que renderiza a tela diretamente pelo protocolo SPICE em tela cheia) é fornecido pelo QEMU e pelo pacote `virt-viewer`, sendo suportado exclusivamente em instâncias criadas como **Máquina Virtual (`--vm`)**.

---

## 3. Preparação do Host Físico (Debian 13 Mínimo)

O computador físico do laboratório não precisa de um desktop completo (como GNOME ou KDE) instalado no sistema operacional hospedeiro. Ele precisa apenas do servidor gráfico básico, do gerenciador de login SDDM e do visualizador SPICE (`virt-viewer`).

### 3.1. Instalar os Pacotes Necessários no Host
No Debian físico (como `root` ou via `sudo`):

```bash
sudo apt update
sudo apt install -y xorg sddm virt-viewer incus zenity
```

### 3.2. Garantir que os Usuários Possam Controlar o Incus
Para que o script de login do SDDM possa iniciar e abrir a VM do aluno sem exigir senha de root, adicione os usuários locais ou o usuário genérico ao grupo `incus-admin`:

```bash
sudo usermod -aG incus-admin aluno 2>/dev/null || true
```

---

## 4. Construção da Imagem Modelo ("Golden Image")

Criaremos uma única imagem modelo baseada no Debian 13 chamada `modelo-desktop`. Todos os ambientes dos alunos serão clones instantâneos desse modelo.

### 4.1. Inicializar a VM Base no Incus
Execute no terminal do host Debian físico:

```bash
# Cria e inicializa a Máquina Virtual com Debian 13
incus launch images:debian/13 modelo-desktop --vm -c limits.cpu=4 -c limits.memory=4GiB

# Aguarda 10 segundos para a VM concluir a inicialização do agente interno
sleep 10

# Abre o terminal root dentro da VM
incus exec modelo-desktop -- bash
```

---

### 4.2. Instalar o KDE Plasma (Wayland) e Utilitários no Guest
Dentro do prompt root da VM `modelo-desktop`, execute os comandos abaixo para instalar o KDE Plasma enxuto com Wayland, o SDDM e ferramentas de desenvolvimento:

```bash
# 1. Atualizar repositórios
apt update && apt upgrade -y

# 2. Instalar KDE Plasma com suporte nativo a Wayland e o SDDM
apt install --no-install-recommends -y \
    kde-plasma-desktop \
    plasma-workspace-wayland \
    sddm \
    konsole \
    dolphin \
    kwrite

# 3. Instalar utilitários essenciais de desenvolvimento e rede
apt install -y \
    firefox-esr \
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

### 4.3. Configurar o Autologin no SDDM da VM
Como o aluno já terá digitado a senha na tela de login física da máquina (Host), a VM deve entrar imediatamente no ambiente gráfico sem solicitar a senha uma segunda vez.

Ainda dentro da VM, configure o autologin do SDDM para a sessão `plasmawayland`:

```bash
# Cria o diretório de configurações do SDDM
mkdir -p /etc/sddm.conf.d

# Configura o autologin para o usuário 'aluno' na sessão KDE Wayland
cat << 'EOF' > /etc/sddm.conf.d/autologin.conf
[General]
DisplayServer=wayland

[Autologin]
User=aluno
Session=plasmawayland
Relogin=false
EOF
```

---

### 4.4. Criar a Conta Local do Aluno na VM
Crie a conta de usuário padrão que o aluno utilizará na interface gráfica com privilégios administrativos via `sudo`:

```bash
#!/bin/bash
USUARIO="aluno"

# Cria o usuário com pasta pessoal e shell bash
useradd -m -s /bin/bash "$USUARIO"

# Define uma senha padrão didática (ex: aluno123)
echo "${USUARIO}:${USUARIO}123" | chpasswd

# Adiciona o usuário aos grupos de sudo, vídeo e áudio
usermod -aG sudo,video,audio "$USUARIO"

# Permite que o usuário utilize o sudo sem exigir senha dentro da VM
cat << EOF > "/etc/sudoers.d/90-${USUARIO}-init"
${USUARIO} ALL=(ALL) NOPASSWD:ALL
EOF
chmod 0440 "/etc/sudoers.d/90-${USUARIO}-init"

# Garante a permissão correta na home do usuário
chown -R "${USUARIO}:${USUARIO}" "/home/${USUARIO}"
```

---

### 4.5. Finalizar e Congelar a Imagem Base
Saia da VM para o host físico e crie o snapshot de referência:

```bash
# Sai da VM
exit

# Para a VM modelo de forma limpa
incus stop modelo-desktop

# Cria um snapshot imutável de referência
incus snapshot create modelo-desktop base

# (Opcional) Define a imagem com descrição no catálogo local
incus publish modelo-desktop --alias template-desktop-kde description="Debian 13 KDE Plasma Wayland com Autologin"
```

---

## 5. Provisionamento dos Ambientes dos Alunos

Com o modelo pronto em Btrfs CoW, provisionar novos ambientes para os alunos leva menos de 2 segundos.

### 5.1. Criar as VMs para os Alunos
Execute no host físico:

```bash
# Cria o ambiente para o aluno João (Turno Manhã)
incus copy modelo-desktop aluno-joao
incus start aluno-joao

# Cria o ambiente para a aluna Maria (Turno Noite)
incus copy modelo-desktop aluno-maria
incus start aluno-maria
```

### 5.2. Verificar as Instâncias Ativas
```bash
incus list -c n,s,t,4
```

*Saída esperada:*
```text
+-------------+---------+-----------------+--------------------+
|    NAME     | STATUS  |      TYPE       |        IPV4        |
+-------------+---------+-----------------+--------------------+
| aluno-joao  | RUNNING | VIRTUAL-MACHINE | 10.0.100.25 (enp5s0)|
| aluno-maria | RUNNING | VIRTUAL-MACHINE | 10.0.100.26 (enp5s0)|
+-------------+---------+-----------------+--------------------+
```

---

## 6. Automação do Host: SDDM + Lançamento da VM em Tela Cheia

Para proporcionar a experiência transparente de computador físico, configuramos o host físico para executar a VM do aluno em tela cheia logo após o login no SDDM.

### 6.1. Script de Inicialização da Sessão (`/usr/local/bin/iniciar-ambiente-aluno.sh`)
Crie este script no host físico:

```bash
sudo nano /usr/local/bin/iniciar-ambiente-aluno.sh
```

Cole o conteúdo:

```bash
#!/bin/bash
# Script de login transparente para laboratório com Incus VM (KDE Wayland)
LOG_FILE="/tmp/incus-session-${USER}.log"
echo "=== Sessão iniciada para $USER em $(date) ===" > "$LOG_FILE"

# 1. Se logar como usuário genérico 'aluno', permite selecionar a VM
if [ "$USER" = "aluno" ]; then
    LISTA=$(incus list -c n --format csv | grep "^aluno-" | sed "s/^aluno-//")
    if [ -z "$LISTA" ]; then
        zenity --error --text="Nenhum ambiente de aluno encontrado no Incus!"
        exit 1
    fi
    ESCOLHA=$(echo "$LISTA" | zenity --list \
        --title="Laboratório de Informática" \
        --text="Selecione o seu ambiente de estudos:" \
        --column="Aluno" \
        --width=350 --height=400)
    [ -z "$ESCOLHA" ] && exit 0
    VM="aluno-${ESCOLHA}"
else
    # Se logar com conta nominal (ex: joao, maria), vai direto para a sua VM
    VM="aluno-${USER}"
fi

echo "Instância selecionada: $VM" >> "$LOG_FILE"

# 2. Verifica se a VM existe
if ! incus info "$VM" >/dev/null 2>&1; then
    zenity --error --text="O ambiente '$VM' não existe. Procure o professor ou administrador do laboratório."
    exit 1
fi

# 3. Inicia a VM caso esteja desligada
STATUS=$(incus list "^${VM}\$" -c s --format csv)
if [ "$STATUS" != "RUNNING" ]; then
    zenity --info --title="Laboratório Virtual" --text="Iniciando seu ambiente de trabalho...\nAguarde 3 segundos." --timeout=3 &
    incus start "$VM"
fi

# 4. Abre o console gráfico SPICE nativo em tela cheia
# O comando 'incus console --type=vga' invoca o remote-viewer (virt-viewer) automaticamente
echo "Lançando console VGA SPICE em tela cheia..." >> "$LOG_FILE"
incus console "$VM" --type=vga --fullscreen >> "$LOG_FILE" 2>&1

# 5. Ao encerrar a sessão (Logout ou Desligar dentro do KDE), garante o desligamento da VM para liberar a RAM
echo "Encerrando instância $VM..." >> "$LOG_FILE"
incus stop "$VM" 2>/dev/null || true
echo "=== Sessão finalizada com sucesso ===" >> "$LOG_FILE"
exit 0
```

Torne o script executável:
```bash
sudo chmod +x /usr/local/bin/iniciar-ambiente-aluno.sh
```

---

### 6.2. Registrar a Sessão no SDDM do Host Físico
Crie o descritor de sessão para o SDDM:

```bash
sudo nano /usr/share/xsessions/incus-aluno.desktop
```

Cole o conteúdo:

```ini
[Desktop Entry]
Name=Ambiente Virtual do Aluno (Incus)
Comment=Carrega a área de trabalho KDE Plasma isolada do estudante
Exec=/usr/local/bin/iniciar-ambiente-aluno.sh
Type=Application
Keywords=incus;vm;kde;wayland;lab;
```

Defina a permissão correta:
```bash
sudo chmod 644 /usr/share/xsessions/incus-aluno.desktop
```

---

### 6.3. Configurar o SDDM no Host Físico
Defina essa sessão como a padrão no host em `/etc/sddm.conf.d/10-lab-session.conf`:

```bash
sudo mkdir -p /etc/sddm.conf.d
sudo bash -c 'cat << "EOF" > /etc/sddm.conf.d/10-lab-session.conf
[General]
DisplayServer=x11

[Theme]
Current=debian-theme

[Users]
RememberLastSession=true

[Autologin]
Session=incus-aluno.desktop
EOF'
```

Reinicie o SDDM no host físico para aplicar:
```bash
sudo systemctl restart sddm.service || true
```

---

## 7. Como Funciona na Prática para o Aluno

1. **Chegada ao Laboratório:** O aluno senta no computador e vê a tela de login do **SDDM**.
2. **Autenticação:** O aluno insere seu usuário (`joao`) e sua senha (`aluno123`).
3. **Abertura Imediata:** O monitor pisca rapidamente e abre a tela cheia do Debian 13. O SDDM interno da VM executa o autologin no usuário `aluno`.
4. **Ambiente KDE Wayland:** O estudante está diante de uma área de trabalho **KDE Plasma nativa em Wayland**, com **Konsole**, **Dolphin**, **Firefox** e aceleração de vídeo, podendo programar e instalar pacotes com `sudo apt`.
5. **Encerramento da Aula:** Quando o aluno clica em **"Desligar"** ou **"Encerrar Sessão"** no menu Iniciar do KDE:
   - A VM encerra seus processos de forma limpa;
   - O `virt-viewer` fecha a janela;
   - O script do host executa `incus stop aluno-joao` para liberar 100% da RAM;
   - O monitor retorna imediatamente à tela de login do SDDM, aguardando o próximo aluno.

---

## 8. Demonstração Prática: O "Teste de Fogo" (Apache2 vs Nginx)

Para validar que o isolamento é absoluto e elimina qualquer conflito de serviços de rede entre turnos:

### Turno 1: Aluno João instala o Apache2 (Turno da Manhã)
1. João faz login na máquina física como `joao`.
2. A tela do KDE Plasma abre automaticamente. Ele abre o terminal **Konsole** e executa:
   ```bash
   sudo apt update
   sudo apt install -y apache2
   echo "<h1>Ambiente do Joao: Servidor Apache2 Ativo na Porta 80</h1>" | sudo tee /var/www/html/index.html
   ```
3. Ele abre o Firefox dentro do KDE e acessa:
   `http://localhost`
   * **Resultado:** A página do Apache2 carrega perfeitamente na porta 80.
4. João clica no menu do KDE em **Desligar**. O container/VM desliga e o PC volta ao SDDM.

---

### Turno 2: Aluna Maria instala o Nginx (Turno da Noite)
1. Maria senta na mesma máquina física e faz login como `maria`.
2. A tela do KDE Plasma abre automaticamente. No **Konsole**, ela executa:
   ```bash
   sudo apt update
   sudo apt install -y nginx
   echo "<h1>Ambiente da Maria: Servidor NGINX Ativo na Porta 80</h1>" | sudo tee /var/www/html/index.nginx-debian.html
   ```
3. Ela abre o Firefox dentro do seu desktop e acessa:
   `http://localhost`
   * **Resultado:** O Nginx carrega com sucesso imediato na porta 80.
   * **Sem erro de porta ocupada:** O fato de o Apache2 do João estar instalado na porta 80 não afeta em nada a Maria, pois pertencem a Namespaces de rede e sistemas de arquivos totalmente isolados.

---

## 9. Manutenção e Restauração Instantânea (Professor e TI)

### 9.1. Restaurar um Ambiente Corrompido
Se um estudante desconfigurar o sistema operacional ou corromper arquivos essenciais:

```bash
# 1. Para a instância do aluno
incus stop aluno-joao

# 2. Restaura o estado inicial limpo em menos de 2 segundos
incus snapshot restore aluno-joao base

# 3. Reinicia o ambiente pronto para uso
incus start aluno-joao
```

### 9.2. Script em Lote para Criar Turmas Inteiras (`criar-turma-gui.sh`)
Para provisionar uma turma inteira antes do início do semestre:

```bash
#!/bin/bash
TURMA="turma-a-2026"
TOTAL=35

for i in $(seq -w 1 $TOTAL); do
    NOME="aluno-${TURMA}-${i}"
    echo "Provisionando $NOME..."
    
    # Clona a imagem modelo instantaneamente
    incus copy modelo-desktop "$NOME"
    
    # Snapshot inicial de segurança
    incus snapshot create "$NOME" base
done

echo "Turma $TURMA provisionada com sucesso!"
```

---

## 10. Conclusão

Essa arquitetura entrega a melhor experiência para laboratórios acadêmicos:
1. **Autonomia Pedagógica Total:** Cada estudante conta com privilégios de `root` via `sudo`, ambiente gráfico moderno com **KDE Plasma sobre Wayland**, liberdade para compilar códigos e subir servidores sem afetar outros turnos.
2. **Eficiência e Estabilidade de Hardware:** Elimina o peso insustentável de máquinas virtuais pesadas convencionais, evita a execução de ambientes de desktop redundantes no host e garante o desligamento automático das instâncias para manter a máquina física sempre limpa e rápida.
