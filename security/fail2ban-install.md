# Tutorial: Instalação e Configuração do Fail2ban no Debian 13 (Trixie) e Ubuntu

> O **Fail2ban** é um serviço de prevenção contra intrusões que analisa logs de serviços de rede (como SSH, Apache e Nginx) em tempo real. Ao identificar comportamentos maliciosos, como sucessivas tentativas erradas de autenticação por ataques de força bruta (*brute-force*) e *botnets*,, o Fail2ban adiciona automaticamente regras dinâmicas no firewall para bloquear temporariamente os IPs dos atacantes.

---

> [!IMPORTANT]
> Nunca altere diretamente o arquivo `/etc/fail2ban/jail.conf`. Em distribuições Debian e Ubuntu, todas as personalizações devem ser mantidas no arquivo `/etc/fail2ban/jail.local`, garantindo que suas regras persistam mesmo após atualizações de pacotes do sistema.

---

## 1. Instalação do Fail2ban

Atualize os repositórios locais e instale o pacote oficial via `apt`:

```bash
sudo apt update
sudo apt install -y fail2ban
```

---

## 2. Configuração da Prisão SSH (`jail.local`)

No **Debian 12+** e **Ubuntu 22.04+**, o serviço `sshd` registra eventos de autenticação diretamente no `systemd-journald` em vez do tradicional arquivo `/var/log/auth.log`. Portanto, definir `backend = systemd` é crucial para que o monitoramento funcione sem falhas.

Crie ou atualize o arquivo `/etc/fail2ban/jail.local`:

```bash
sudo tee /etc/fail2ban/jail.local > /dev/null << 'EOF'
[DEFAULT]
# Tempo padrão de banimento do IP (1 hora)
bantime = 1h

# Janela de monitoramento para contabilizar tentativas falhas (10 minutos)
findtime = 10m

# Número máximo de falhas dentro da janela antes de aplicar o bloqueio
maxretry = 3

# Endereços que NUNCA devem ser banidos (whitelist)
# Adicione o IP público ou VPN do administrador separado por espaço
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = ssh
backend = systemd
EOF
```

> [!TIP]
> **Proteção contra auto-bloqueio:** Se você possui um IP estático na sua rede ou utiliza uma VPN corporativa, adicione-o ao parâmetro `ignoreip` (ex: `ignoreip = 127.0.0.1/8 ::1 200.150.10.25`).

---

## 3. Iniciar e Habilitar o Serviço

Inicie o Fail2ban e configure-o para inicializar automaticamente com o boot do sistema operacional:

```bash
sudo systemctl enable --now fail2ban
```

Para validar se o serviço está ativo e sem erros:

```bash
sudo systemctl status fail2ban
```

---

## 4. Monitoramento e Verificação de Status

O utilitário `fail2ban-client` gerencia as prisões (*jails*) em execução.

### 4.1. Listar todas as prisões ativas
```bash
sudo fail2ban-client status
```
*A saída informará a lista de prisões habilitadas (ex: `Jail list: sshd`).*

### 4.2. Inspecionar a prisão do SSH
```bash
sudo fail2ban-client status sshd
```

Saída típica esperada:
```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     6
|  `- File list:        systemd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   198.51.100.42
```

---

## 5. Gerenciamento de IPs: Desbanir e Whitelist

### 5.1. Desbanir um IP bloqueado manualmente
Se você ou algum usuário de confiança digitar a credencial errada e for bloqueado por engano, você pode revogar a punição imediatamente:

```bash
# Desbanir apenas da prisão sshd
sudo fail2ban-client set sshd unbanip SEU_IP_AQUI

# (Alternativa) Desbanir de todas as prisões ativas de uma só vez
sudo fail2ban-client unban SEU_IP_AQUI
```

### 5.2. Confirmar a liberação
Execute o comando de status para garantir que o endereço não consta mais em `Banned IP list`:

```bash
sudo fail2ban-client status sshd
```

### 5.3. Tornar o IP imune (Adicionar à lista branca)
Para impedir que esse endereço volte a ser bloqueado no futuro, adicione-o à linha `ignoreip` dentro de `/etc/fail2ban/jail.local` e recarregue o serviço:

```bash
sudo fail2ban-client reload
```

---

## 6. Compreendendo as Métricas do Fail2ban

Ao analisar o status da prisão, é comum ver contadores subindo com o tempo:

- **`Currently failed`**: Número de tentativas com falha registradas dentro da janela atual de tempo (`findtime = 10m`). Se o atacante não atingir o limite `maxretry`, esse valor zera sozinho.
- **`Total failed`**: Total acumulado de tentativas incorretas desde o momento em que o Fail2ban foi iniciado. Bots automatizados testam constantemente portas públicas em busca de falhas; é comum esse número chegar a centenas ou milhares sem afetar a estabilidade do servidor.
- **`Currently banned`**: Quantidade de IPs atualmente impedidos de acessar a porta monitorada pelo firewall.
- **`Total banned`**: Contagem histórica de banimentos aplicados pelo serviço.

> [!NOTE]
> O Fail2ban atua como uma barreira de contenção de tráfego. Combinado com autenticação SSH baseada exclusivamente em chaves públicas (`PubkeyAuthentication yes`) e desativação de login por senha (`PasswordAuthentication no`), seu servidor torna-se praticamente impenetrável a ataques por força bruta.

---

## 7. Dicas de Endurecimento (*Hardening*) Adicional

### 7.1. Alteração da Porta Padrão do SSH
A maioria absoluta dos ataques automatizados busca apenas a porta padrão `22/tcp`. Mudar o SSH para uma porta alta (ex: `2222` ou `54321`) reduz o ruído de logs em mais de 95%:

1. Altere a porta no SSH: edite `/etc/ssh/sshd_config` (ou `/etc/ssh/sshd_config.d/`) e defina `Port 2222`.
2. Libere a nova porta no firewall antes de fechar a sessão atual:
   ```bash
   sudo ufw allow 2222/tcp comment 'SSH Custom Port'
   ```
3. No arquivo `/etc/fail2ban/jail.local`, atualize o parâmetro da prisão:
   ```ini
   [sshd]
   enabled = true
   port = 2222
   backend = systemd
   ```
4. Recarregue os serviços:
   ```bash
   sudo systemctl restart ssh
   sudo fail2ban-client reload
   ```

### 7.2. Acompanhamento dos logs em tempo real
Para inspecionar ações de bloqueio e liberação de IPs à medida que acontecem:

```bash
sudo tail -f /var/log/fail2ban.log
```

Para visualizar e listar os logs do Fail2ban no Linux, você pode consultar o arquivo de log padrão ou usar o utilitário de linha de comando do próprio serviço.

1. Ler o arquivo de log diretamente

O arquivo principal fica geralmente em /var/log/fail2ban.log. Use os comandos abaixo no terminal conforme sua necessidade:
* Ver as últimas 50 linhas:
```bash
sudo tail -n 50 /var/log/fail2ban.log
```
* Acompanhar o log em tempo real:
```bash
sudo tail -f /var/log/fail2ban.log
```
* Filtrar apenas IPs banidos ou desbanidos:
```bash
sudo grep -E "Ban|Unban" /var/log/fail2ban.log
```
* Navegar pelo arquivo completo:
```bash
sudo less /var/log/fail2ban.log
```
(Pressione q para sair).

2. Usar o comando journalctl (Sistemas com Systemd)

Se o seu sistema não estiver gravando no arquivo tradicional ou preferir consultar o log do serviço via systemd, execute:
* Ver os registros recentes do serviço:
```bash
sudo journalctl -u fail2ban
```
* Acompanhar em tempo real:
```bash
sudo journalctl -u fail2ban -f
```


