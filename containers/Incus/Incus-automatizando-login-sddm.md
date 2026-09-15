# Automação de Login via SDDM no Incus com FreeRDP

---

## 1. Visão Geral e Arquitetura

O **SDDM** (*Simple Desktop Display Manager*) não é obrigado a abrir uma área de trabalho pesada instalada no sistema operacional real (como KDE Plasma ou GNOME). Ele pode ser configurado para executar uma **Sessão Customizada (*XSession*)** que entrega um ambiente 100% isolado por container.

### O Fluxo da Solução:
```text
[ Tela de Login do SDDM no Monitor Físico ]
         │
         │  (Aluno digita: usuario "joao" e senha "aluno123")
         ▼
[ SDDM valida a conta e executa: /usr/share/xsessions/incus-aluno.desktop ]
         │
         │  (Dispara o script /usr/local/bin/iniciar-sessao-sddm.sh)
         ▼
[ 1. incus start aluno-$USER ]
[ 2. Abre FreeRDP em tela cheia direto no IP do container com /cert:ignore ]
         │
         │  (Aluno estuda, instala Apache2/Nginx, compila, programa)
         │  (Aluno clica em "Sair / Encerrar Sessão" no XFCE)
         ▼
[ 3. Conexão RDP encerra ]
[ 4. incus stop aluno-$USER (desliga o container e libera a RAM) ]
         ▼
[ Retorna automaticamente para a Tela de Login do SDDM ]
```

---

## 2. Script de Instalação e Automação em 1 Clique

Para configurar todo o servidor host (pacotes, permissões, contas didáticas, scripts e SDDM) de uma só vez, crie e execute o script abaixo como `root` no host Debian:

```bash
sudo nano /usr/local/bin/setup-sddm-incus-lab.sh
```

Cole o conteúdo:

```bash
#!/bin/bash
set -e

echo "=== 1. Instalando dependências essenciais no Host ==="
apt update
apt install -y freerdp3-x11 zenity netcat-openbsd

# No Debian 13, o binário do FreeRDP 3 é 'xfreerdp3'. Criamos o symlink universal:
ln -sf /usr/bin/xfreerdp3 /usr/bin/xfreerdp

echo "=== 2. Criando contas didáticas locais para os alunos no Host ==="
# Cria os usuários joao e maria com senha padronizada 'aluno123'
for u in joao maria; do
    if ! id "$u" &>/dev/null; then
        useradd -m -s /bin/bash "$u"
        echo "Usuário $u criado com sucesso."
    fi
    echo "$u:aluno123" | chpasswd
    usermod -aG incus-admin "$u"
    chown -R "$u:$u" "/home/$u"
    echo "Permissões do Incus concedidas para $u."
done

# Garante que o usuário genérico 'aluno' também tenha permissão de gerenciar o Incus
usermod -aG incus-admin aluno 2>/dev/null || true

echo "=== 3. Criando o Script Orquestrador de Sessão ==="
cat << "EOF" > /usr/local/bin/iniciar-sessao-sddm.sh
#!/bin/bash
LOG_FILE="/tmp/incus-sddm-session.log"
echo "=== Sessão iniciada para $USER em $(date) ===" > "$LOG_FILE"

# Se logar como usuário genérico 'aluno', exibe o menu seletor (Zenity)
if [ "$USER" = "aluno" ]; then
    LISTA=$(incus list -c n --format csv | grep "^aluno-" | sed "s/^aluno-//")
    if [ -z "$LISTA" ]; then
        zenity --error --text="Nenhum ambiente de aluno encontrado no Incus!"
        exit 1
    fi
    ESCOLHA=$(echo "$LISTA" | zenity --list \
        --title="Laboratório Virtual de Informática" \
        --text="Selecione o seu ambiente de aula:" \
        --column="Aluno" \
        --width=350 --height=400)
    [ -z "$ESCOLHA" ] && exit 0
    CONTAINER="aluno-${ESCOLHA}"
else
    # Se logar como joao ou maria, vai direto para seu container dedicado
    CONTAINER="aluno-${USER}"
fi

echo "Container selecionado: $CONTAINER" >> "$LOG_FILE"

# Verifica se o container existe
if ! incus info "$CONTAINER" >/dev/null 2>&1; then
    zenity --error --text="Erro: O ambiente '$CONTAINER' não foi provisionado no Incus."
    exit 1
fi

# Inicia o container caso esteja parado
STATUS=$(incus list "^${CONTAINER}\$" -c s --format csv)
if [ "$STATUS" != "RUNNING" ]; then
    zenity --info --title="Laboratório Virtual" --text="Iniciando seu ambiente virtual...\nAguarde alguns instantes." --timeout=3 &
    incus start "$CONTAINER"
fi

# Aguarda obter endereço IPv4 na bridge incusbr0
IP=""
for i in {1..20}; do
    IP=$(incus list "^${CONTAINER}\$" -c 4 --format csv | awk "{print \$1}")
    [ -n "$IP" ] && break
    sleep 0.5
done

if [ -z "$IP" ]; then
    zenity --error --text="Falha ao obter endereço IP do container."
    exit 1
fi

echo "IP do container: $IP" >> "$LOG_FILE"

# Aguarda o serviço XRDP (porta 3389) estar pronto
for i in {1..20}; do
    nc -z -w 1 "$IP" 3389 2>/dev/null && break
    sleep 0.5
done

# Conecta via FreeRDP 3 em tela cheia com certificado ignorado
RDP_BIN=$(which xfreerdp3 || which xfreerdp)
echo "Iniciando $RDP_BIN para $IP..." >> "$LOG_FILE"

"$RDP_BIN" /v:"$IP" /u:aluno /p:aluno123 /f /dynamic-resolution +clipboard /sound /cert:ignore >> "$LOG_FILE" 2>&1

# Ao encerrar a sessão (Logout no XFCE), desliga o container para liberar 100% da RAM
echo "Desligando container $CONTAINER..." >> "$LOG_FILE"
incus stop "$CONTAINER"
echo "=== Sessão finalizada com sucesso ===" >> "$LOG_FILE"
exit 0
EOF

chmod +x /usr/local/bin/iniciar-sessao-sddm.sh

echo "=== 4. Registrando a Sessão X11 no SDDM ==="
cat << "EOF" > /usr/share/xsessions/incus-aluno.desktop
[Desktop Entry]
Name=Laboratório Virtual (Incus)
Comment=Inicia a área de trabalho virtual individual do aluno
Exec=/usr/local/bin/iniciar-sessao-sddm.sh
Type=Application
Keywords=incus;container;virtual;lab;
EOF

chmod 644 /usr/share/xsessions/incus-aluno.desktop

echo "=== 5. Configurando o SDDM ==="
mkdir -p /etc/sddm.conf.d
cat << "EOF" > /etc/sddm.conf.d/10-lab-session.conf
[General]
DisplayServer=x11

[Theme]
Current=debian-theme

[Users]
RememberLastSession=false

[Autologin]
Session=incus-aluno.desktop
EOF

echo "=== 6. Definindo a sessão padrão nos perfis dos usuários ==="
for u in joao maria aluno; do
    HOMEDIR=$(getent passwd "$u" | cut -d: -f6)
    if [ -d "$HOMEDIR" ]; then
        cat << "EOF" > "$HOMEDIR/.dmrc"
[Desktop]
Session=incus-aluno
EOF
        chown "$u:$u" "$HOMEDIR/.dmrc" 2>/dev/null || true
    fi
done

# Reinicia o SDDM para carregar as alterações
systemctl restart sddm.service || true

echo "=== Configuração do SDDM concluída com sucesso! ==="
```

Torne o script executável e execute-o:
```bash
sudo chmod +x /usr/local/bin/setup-sddm-incus-lab.sh
sudo /usr/local/bin/setup-sddm-incus-lab.sh
```

---

## 3. Os Três Pontos Críticos que Faziam a Integração Falhar Anteriormente

1. **Binário do FreeRDP no Debian 13 (`xfreerdp3`):**
   * O pacote `freerdp3-x11` instala o executável `/usr/bin/xfreerdp3`. Se o script invocar apenas `xfreerdp`, o comando não é encontrado e a sessão encerra em 0.1 segundo. O symlink `/usr/bin/xfreerdp -> /usr/bin/xfreerdp3` resolve isso definitivamente.
2. **Confirmação de Certificado SSL do XRDP (`/cert:ignore`):**
   * Por padrão, conexões RDP contra certificados autoassinados aguardam confirmação interativa do usuário. No modo gráfico direto, essa espera travava a inicialização. A flag `/cert:ignore` permite a conexão imediata e silenciosa.
3. **Permissão de Grupo (`incus-admin`):**
   * Os usuários precisam pertencer ao grupo `incus-admin` no host para poderem executar `incus start` e `incus stop` através do socket `/var/lib/incus/unix.socket` sem exigir senha de root.

---

## 4. Como Testar no Laboratório

Agora você pode testar na máquina real de duas formas diferentes:

### Modo 1: Login Nominal Direto
1. Na tela do SDDM, digite o usuário `joao` e senha `aluno123`.
2. Em 2 segundos, a tela cheia se torna a área de trabalho do container `aluno-joao`.
3. Ao clicar em **"Sair / Encerrar Sessão"** no XFCE, o container desliga e o computador volta para o SDDM.
4. Digite a usuária `maria` e senha `aluno123`. O container `aluno-maria` inicia isoladamente com seu próprio IP e desktop.

### Modo 2: Conta Única com Seletor (Zenity)
1. Na tela do SDDM, digite o usuário `aluno` e senha `aluno123`.
2. Uma janela visual do **Zenity** surge perguntando: *"Selecione o seu ambiente de aula"*.
3. O aluno clica em `joao` ou `maria` e o FreeRDP carrega o desktop correspondente.

---

## 5. Logs e Diagnóstico de Falhas

Se alguma sessão fechar inesperadamente, basta inspecionar o arquivo de log gerado no host:

```bash
cat /tmp/incus-sddm-session.log
```