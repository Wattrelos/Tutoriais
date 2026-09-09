# Automação de login via SDDM no Incus com FreeRDP

*Essa é a forma mais profissional e elegante de implantar essa solução em um laboratório.* 

O **SDDM** (*Simple Desktop Display Manager*) não é obrigado a abrir um desktop tradicional do host (como KDE ou GNOME). Ele pode ser configurado para rodar uma **Sessão Customizada (*XSession*)** que:

1. Autentica o aluno na tela de login normal do SDDM.
2. Inicia o container Incus individual do aluno em segundo plano.
3. Abre a interface gráfica do container em **tela cheia imediata**.
4. **Protege o Host:** O aluno **nunca vê nem tem acesso à área de trabalho do sistema operacional real**.
5. Ao clicar em "Sair" (*Logout*) dentro do container, o script desliga o container e o SDDM volta instantaneamente para a tela de login, liberando 100% da memória RAM para o aluno do turno seguinte.

---

### Como funciona a Arquitetura com o SDDM

```text
[ Tela de Login do SDDM no Monitor Físico ]
         │
         │  (Aluno digita: usuario "joao" e senha)
         ▼
[ SDDM valida a conta e executa: /usr/share/xsessions/incus-aluno.desktop ]
         │
         │  (Dispara o script /usr/local/bin/iniciar-sessao-sddm.sh)
         ▼
[ 1. incus start aluno-$USER ]
[ 2. Abre FreeRDP em tela cheia direto no IP do container ]
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

### Passo a Passo de Configuração no Host Debian

#### 1. Criar a Sessão Customizada para o SDDM
Crie um arquivo `.desktop` no diretório de sessões do X11 no host:

```bash
sudo nano /usr/share/xsessions/incus-aluno.desktop
```

Insira o conteúdo:
```ini
[Desktop Entry]
Name=Laboratório Virtual (Incus)
Comment=Inicia a área de trabalho virtual individual do aluno
Exec=/usr/local/bin/iniciar-sessao-sddm.sh
Type=Application
Keywords=incus;container;virtual;lab;
```

---

#### 2. Criar o Script Orquestrador (`/usr/local/bin/iniciar-sessao-sddm.sh`)
Este script roda no momento em que o SDDM autentica o aluno, herdando a variável `$USER`:

```bash
sudo nano /usr/local/bin/iniciar-sessao-sddm.sh
```

Conteúdo do script:
```bash
#!/bin/bash
# Identifica o nome do aluno autenticado no SDDM
ALUNO="$USER"
CONTAINER="aluno-${ALUNO}"

# 1. Verifica se o container do aluno existe no Incus
if ! incus info "$CONTAINER" >/dev/null 2>&1; then
    # Se não existir, avisa na tela e encerra voltando ao SDDM
    xmessage -center -buttons Ok:0 "Erro: O ambiente para o usuário '$ALUNO' não foi provisionado. Procure o professor ou suporte técnico."
    exit 1
fi

# 2. Inicia o container do aluno (caso esteja parado)
incus start "$CONTAINER" 2>/dev/null || true

# 3. Aguarda o container obter o endereço IP na rede interna (máximo 5 segundos)
for i in {1..10}; do
    IP=$(incus list "$CONTAINER" -c 4 --format csv | awk '{print $1}')
    [ -n "$IP" ] && break
    sleep 0.5
done

if [ -z "$IP" ]; then
    xmessage -center -buttons Ok:0 "Erro: Não foi possível obter o IP do ambiente virtual."
    incus stop "$CONTAINER"
    exit 1
fi

# 4. Lança o FreeRDP em tela cheia conectando na interface gráfica do container
# Parâmetros:
# /f -> Tela cheia (oculta qualquer resquício do host)
# /dynamic-resolution -> Ajusta na resolução nativa do monitor
# +clipboard -> Permite copiar e colar
# /sound -> Redireciona áudio
xfreerdp /v:"$IP" /u:aluno /p:aluno123 /f /dynamic-resolution +clipboard /sound

# 5. QUANDO O ALUNO CLICAR EM 'LOGOUT' NO XFCE:
# O comando xfreerdp acima encerra. O script continua e desliga o container:
incus stop "$CONTAINER"

# Finaliza a sessão do X11, forçando o SDDM a reabrir a tela de login
exit 0
```

Torne o script executável:
```bash
sudo chmod +x /usr/local/bin/iniciar-sessao-sddm.sh
```

---

#### 3. Configurar o SDDM para usar essa Sessão como Padrão
Para que o aluno não precise escolher nada em menus suspensos no SDDM:

Crie o arquivo de configuração de sessão do SDDM:
```bash
sudo mkdir -p /etc/sddm.conf.d
sudo nano /etc/sddm.conf.d/10-lab-session.conf
```

Insira:
```ini
[Theme]
Current=debian-theme

[Users]
# Força a carregar a sessão do Incus por padrão
DefaultSession=incus-aluno.desktop

[General]
DisplayServer=x11
```

---

#### 4. Permissões de Execução do Incus para os Usuários do Host
Para que o script do aluno consiga executar `incus start` e `incus stop` sem pedir senha de administrador no host:

Basta adicionar os usuários de login ao grupo `incus-admin` (ou criar uma regra pontual no `sudoers` se preferir restringir apenas aos comandos de start/stop):

```bash
# Substitua 'seu_usuario' pelo nome real do usuário que faz login no host (ex: wattrelos ou aluno)
sudo usermod -aG incus-admin seu_usuario

# Exemplo no seu servidor (adicionando o usuário wattrelos):
sudo usermod -aG incus-admin wattrelos
```

> **Atenção:** `joao` e `maria` citados anteriormente são apenas **nomes fictícios de exemplo**. No sistema físico, você só precisa dar essa permissão para o usuário real que loga no Debian host (como o usuário `wattrelos` ou uma conta genérica `aluno`).

---

### Experiência Final do Aluno no Laboratório:

1. **Chegada:** O aluno senta na cadeira da máquina física. O monitor exibe a tela limpa do SDDM.
2. **Login:** Ele digita sua conta (ex: `joao`) e sua senha.
3. **Entrada:** Em menos de 2 segundos, a tela cheia do monitor físico se torna a sua área de trabalho XFCE (com terminal, VS Code, Firefox e acesso `root`).
4. **Uso:** Ele instala Apache2 na porta 80, cria bancos de dados, programa.
5. **Saída:** No final da aula, clica no botão **"Sair / Encerrar Sessão"** do XFCE.
6. **Resultado:** O container desliga em segundo plano, liberando os 16 GB de RAM e a máquina volta instantaneamente para a tela do SDDM para o aluno da próxima turma!

---

# Inserindo contas de usuário no Debian Físico

Para a administração de um laboratório, **é melhor e mais fácil NÃO criar contas individuais no Debian físico**. 

O **Modelo 1** é o mais recomendado e prático:

---

### Modelo 1: Conta Única no Host + Seletor de Aluno (O Mais Recomendado 🌟)

Neste modelo, a máquina física possui apenas **uma única conta genérica** (ex: `aluno` ou `laboratorio`). Toda a separação dos alunos existe **exclusivamente dentro do Incus**.

#### Como funciona na prática:
1. A máquina real liga e entra automaticamente na conta genérica (ou o aluno apenas clica em "Entrar").
2. Uma janela visual e amigável (**Zenity**) abre na tela com a lista de alunos daquela máquina (obtida dinamicamente dos containers do Incus).
3. O aluno clica no seu nome (ou digita sua matrícula/PIN).
4. O script sobe o container dele (`incus start aluno-joao`) e abre o FreeRDP em tela cheia.
5. Quando ele clica em "Sair", o container desliga e a tela volta para a lista de alunos.

#### Vantagens gigantescas:
* **Zero poluição no host:** Você não precisa encher o Debian real com dezenas de pastas `/home/joao`, `/home/maria`.
* **Manutenção centralizada:** Toda a vida, arquivos, pacotes do Apache/Nginx e permissões do aluno ficam 100% dentro do container.
* **Agilidade:** Para adicionar um aluno novo na máquina, você só precisa rodar `incus copy modelo-desktop aluno-pedro`. Não precisa mexer em `/etc/passwd` do host nem criar senhas de sistema.

---

### Exemplo Prático do Menu Seletor (Modelo 1)

Você pode substituir o script do host por um seletor visual automático como este:

```bash
#!/bin/bash
# /usr/local/bin/seletor-ambiente-aluno.sh

# 1. Lista todos os containers de alunos existentes no Incus
ALUNOS=$(incus list -c n --format csv | grep '^aluno-' | sed 's/^aluno-//')

if [ -z "$ALUNOS" ]; then
    zenity --error --text="Nenhum container de aluno cadastrado nesta máquina!"
    exit 1
fi

# 2. Abre uma janela gráfica para o aluno selecionar seu nome
ESCOLHA=$(echo "$ALUNOS" | zenity --list \
    --title="Laboratório Virtual de Informática" \
    --text="Selecione o seu ambiente de aula:" \
    --column="Aluno / Matrícula" \
    --width=350 --height=400)

# Se cancelou, sai
[ -z "$ESCOLHA" ] && exit 0

CONTAINER="aluno-${ESCOLHA}"

# 3. Notificação e Start
zenity --info --text="Iniciando o ambiente de $ESCOLHA...\nAguarde 2 segundos." --timeout=2 &
incus start "$CONTAINER" 2>/dev/null || true

# 4. Obtém o IP da interface interna
for i in {1..10}; do
    IP=$(incus list "$CONTAINER" -c 4 --format csv | awk '{print $1}')
    [ -n "$IP" ] && break
    sleep 0.5
done

# 5. Abre em tela cheia
xfreerdp /v:"$IP" /u:aluno /p:aluno123 /f /dynamic-resolution +clipboard /sound

# 6. Ao sair, encerra o container liberando a memória RAM
incus stop "$CONTAINER"
```

---

### Modelo 2: Contas Individuais (Apenas se houver Active Directory / LDAP)

Você só precisa de contas individuais se a sua faculdade já possuir um **servidor central de autenticação de rede** (Active Directory, OpenLDAP ou FreeIPA).

* **Como funcionaria:** Você conecta o Debian físico ao AD da faculdade via `SSSD`.
* O aluno senta, digita a matrícula da faculdade no SDDM.
* O SDDM valida a senha com o servidor da faculdade e passa a variável `$USER` (matrícula) para o script abrir o container correspondente `aluno-$USER`.
* Mesmo nesse cenário, **você não cria contas locais manualmente**; o Linux consulta o servidor central da faculdade na hora do login.

---

### Comparativo Rápido:

| Abordagem | Precisa criar contas no Debian real? | Como o aluno entra? | Recomendado para: |
| :--- | :--- | :--- | :--- |
| **Menu Seletor (Modelo 1)** | ❌ **NÃO** (Apenas 1 usuário genérico no host) | Clica no seu nome na lista na tela | **Laboratórios locais sem servidor de domínio** |
| **Integração AD/LDAP (Modelo 2)** | ❌ **NÃO** (Validado via rede da faculdade) | Digita matrícula e senha institucional no SDDM | **Universidades com infraestrutura de rede centralizada** |
| **Contas Locais Manuais** | ⚠️ **SIM** (`adduser` para cada aluno no host) | Digita usuário local no SDDM | **Não recomendado** (gera trabalho administrativo dobrado) |

**Resumo:** Não crie contas no Debian real. Mantenha o Debian físico apenas como o "hospedeiro invisível" e deixe todos os alunos existirem única e exclusivamente dentro do Incus!