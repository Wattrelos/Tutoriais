# Incus-troubleshooting-004

## Resumo da correção:

O que aconteceu é normal e esperado nessa etapa. O comando `incus console modelo-desktop --type=vga` funcionou e conectou ao display virtual (SPICE/virt-viewer), mas exibiu a tela em modo texto (TTY) por **três motivos**:

1. **O serviço gráfico não foi ativado no systemd:** A imagem base do Debian no Incus vem configurada para inicializar em modo texto (`multi-user.target`). A simples instalação dos pacotes via `apt` não altera o alvo padrão do sistema nem inicia o gerenciador gráfico imediatamente.
2. **O KDE Plasma não abre como `root`:** O KDE/Wayland bloqueia a inicialização de sessão gráfica sob o usuário `root` por segurança. Por isso, os passos 4.3 (autologin) e 4.4 (criação do usuário `aluno`) são obrigatórios para ver o desktop rodando.
3. **Pacotes complementares essenciais:** Como o passo 4.2 utilizou `--no-install-recommends`, alguns pacotes vitais para o display e para a integração com o `virt-viewer` ficaram de fora.

---

### Sim, você deve instalar alguns pacotes adicionais na imagem

Para que a VM funcione perfeitamente com o console VGA do Incus, instale estes componentes dentro da VM:

```bash
# Entre na VM pelo terminal do host:
incus exec modelo-desktop -- bash
```

Dentro da VM, execute:

```bash
apt install -y \
    spice-vdagent \
    xwayland \
    xserver-xorg \
    libxcb-cursor0 \
    sddm-theme-breeze
```

#### Por que esses pacotes são fundamentais?
* **`spice-vdagent` (Crítico):** É o agente do SPICE/QEMU para o Incus. Sem ele, a resolução da tela não se ajusta automaticamente ao tamanho da janela do `virt-viewer`, o ponteiro do mouse fica desalinhado ou "preso", e o Copiar/Colar (clipboard) entre o Host e a VM não funciona.
* **`xwayland` e `xserver-xorg`:** O SDDM e diversos aplicativos do KDE ainda requerem componentes X11/Xwayland para renderizar o cursor e a tela de login.
* **`libxcb-cursor0` e `sddm-theme-breeze`:** Sem eles, o SDDM instalado com `--no-install-recommends` falha ao carregar o tema visual do login e o cursor do mouse, abortando de volta para a tela preta ou terminal.

---

### Passo a passo para abrir a interface gráfica agora

Ainda dentro do terminal da VM (`incus exec modelo-desktop -- bash`):

#### 1. Configure o SDDM (Passo 4.3 do tutorial)
```bash
mkdir -p /etc/sddm.conf.d
cat << 'EOF' > /etc/sddm.conf.d/autologin.conf
[General]
DisplayServer=wayland

[Autologin]
User=aluno
Session=plasma
Relogin=false
EOF
```

#### 2. Crie o usuário `aluno` (Passo 4.4 do tutorial)
O KDE precisa de um usuário comum para carregar a área de trabalho:
```bash
useradd -m -s /bin/bash aluno
echo "aluno:aluno123" | chpasswd
usermod -aG sudo,video,audio aluno

cat << EOF > /etc/sudoers.d/90-aluno-init
aluno ALL=(ALL) NOPASSWD:ALL
EOF
chmod 0440 /etc/sudoers.d/90-aluno-init
chown -R aluno:aluno /home/aluno
```

#### 3. Defina o Debian para iniciar sempre em modo gráfico e inicie o SDDM
```bash
# Define o target do systemd para interface gráfica (em vez de modo texto)
systemctl set-default graphical.target

# Inicia o SDDM imediatamente
systemctl start sddm
```

---

### Testando no Host

Agora, no Konsole do seu Debian 13 físico, execute:

```bash
incus console modelo-desktop --type=vga
```

A janela do **`virt-viewer`** será aberta, o SDDM fará o autologin na conta `aluno` e a área de trabalho do **KDE Plasma em Wayland** carregará na tela.