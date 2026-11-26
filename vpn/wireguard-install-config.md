# 🛡️ Instalação e Configuração do WireGuard (Debian 13 Local ↔ Debian 13 VPS Skanstull)

> [!NOTE]
> **Por que WireGuard no Skanstull?** Na arquitetura com **Debian 13** nas duas pontas e IPv4 público fixo na VPS (Hostinger), o WireGuard roda nativamente no kernel Linux com consumo quase nulo de memória e CPU. Como usa pacotes UDP silenciosos (não responde a *scans* sem a chave pública correta), ele torna o **SSH invisível para bots** assim que a porta pública do SSH for fechada no UFW.

---

## 1. Instalar o WireGuard nas Duas Pontas

Execute o comando abaixo tanto na **VPS** quanto no seu **Debian 13 local**:

```bash
sudo apt update && sudo apt install -y wireguard-tools
```

*(O módulo de kernel do WireGuard já vem embutido nativamente no kernel do Debian 13; o pacote `wireguard-tools` fornece os utilitários `wg` e `wg-quick`).*

---

## 2. Gerar o Par de Chaves (VPS e Local)

Em **ambas as máquinas**, gere as chaves privada e pública com permissão restrita (`0600`):

```bash
sudo mkdir -p /etc/wireguard
wg genkey | sudo tee /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
sudo chmod 600 /etc/wireguard/private.key
```

Consulte e anote o conteúdo de `/etc/wireguard/public.key` de cada máquina (você cruzará essas chaves nos arquivos `wg0.conf` abaixo):

```bash
sudo cat /etc/wireguard/public.key
```

> [!CAUTION]
> **Nunca compartilhe ou versione `private.key` em repositórios Git.** Apenas a chave pública (`public.key`) deve ser copiada para a outra ponta.

---

## 3. Configurar o Servidor (VPS Debian 13)

1. Crie o arquivo `/etc/wireguard/wg0.conf` na VPS:
   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

2. Adicione a configuração abaixo (substituindo pelas chaves reais):
   ```ini
   [Interface]
   Address = 10.10.0.1/24
   ListenPort = 51820
   PrivateKey = <CHAVE_PRIVADA_DA_VPS>

   [Peer]
   # Estação de trabalho Debian 13 Local
   PublicKey = <CHAVE_PUBLICA_DO_SEU_DEBIAN_LOCAL>
   AllowedIPs = 10.10.0.2/32
   ```

3. Proteja o arquivo de configuração (ele contém a `PrivateKey`), libere a porta UDP no UFW da VPS e ative o túnel:
   ```bash
   sudo chmod 600 /etc/wireguard/wg0.conf
   sudo ufw allow 51820/udp comment 'WireGuard VPN'
   sudo systemctl enable --now wg-quick@wg0
   ```

4. Verifique se a interface `wg0` subiu na VPS:
   ```bash
   sudo wg show
   ```

---

## 4. Configurar o Cliente (Debian 13 Local)

1. Crie `/etc/wireguard/wg0.conf` no seu computador local:
   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

2. Adicione a configuração abaixo:
   ```ini
   [Interface]
   Address = 10.10.0.2/24
   PrivateKey = <CHAVE_PRIVADA_DO_SEU_DEBIAN_LOCAL>

   [Peer]
   PublicKey = <CHAVE_PUBLICA_DA_VPS>
   Endpoint = <IP_PUBLICO_DA_VPS>:51820
   AllowedIPs = 10.10.0.1/32
   PersistentKeepalive = 25
   ```

   > **Por que `AllowedIPs = 10.10.0.1/32`?** Garante modo *split-tunnel*: apenas o tráfego destinado à VPS (`10.10.0.1`) passa pela VPN; sua navegação comum na internet e chamadas externas continuam usando sua conexão direta sem perda de velocidade.

3. Proteja o arquivo, ative o túnel localmente e valide o *handshake*:
   ```bash
   sudo chmod 600 /etc/wireguard/wg0.conf
   sudo systemctl enable --now wg-quick@wg0
   sudo wg show
   ```

4. Teste o acesso SSH com o usuário `deploy` através do IP privado do túnel (usando a porta configurada no seu `sshd`, seja `22` ou porta customizada):
   ```bash
   ssh -p <SUA_PORTA_SSH> deploy@10.10.0.1
   ```

---

## 5. Blindar o SSH no UFW e Atualizar Atalhos Locais

> [!WARNING]
> **Só execute este passo depois de confirmar que o `ssh ... deploy@10.10.0.1` funcionou.** Em caso de bloqueio acidental, utilize o **Browser Terminal** no hPanel da Hostinger para restaurar o acesso.

1. Restrinja a porta SSH exclusivamente à interface `wg0` na VPS e remova a regra pública:
   ```bash
   sudo ufw allow in on wg0 to any port <SUA_PORTA_SSH> proto tcp comment 'SSH exclusivo via WireGuard'
   sudo ufw delete allow <SUA_PORTA_SSH>/tcp
   sudo ufw status verbose
   ```

2. Atualize o `~/.ssh/config` na sua máquina local para apontar para o IP privado da VPN:
   ```sshconfig
   Host skanstull-vps
       HostName 10.10.0.1
       User deploy
       Port <SUA_PORTA_SSH>
       IdentityFile ~/.ssh/id_ed25519
       IdentitiesOnly yes
       ServerAliveInterval 60
       ServerAliveCountMax 3
   ```

---

## 6. Uso Diário no Ecossistema Skanstull

* **Administração e Túneis de Banco (`127.0.0.1`):**
  Os serviços internos da VPS (MySQL `3306`, MongoDB `27017`, Redis `6379`, RabbitMQ `5672`) continuam isolados em `127.0.0.1`. Com o WireGuard ativo, seus túneis SSH funcionam normalmente sobre a VPN:
  ```bash
  ssh -N -L 13306:127.0.0.1:3306 -L 27017:127.0.0.1:27017 skanstull-vps
  ```
* **Git via SSH no Forgejo:**
  Se a porta SSH pública foi fechada e restrita à `wg0`, operações `git push/pull` via SSH funcionam sempre que o túnel WireGuard estiver ativo (apontando o `Host` para `10.10.0.1`). Desenvolvedores ou pipelines sem VPN continuam utilizando Git via HTTPS (`443` / `8443` via Nginx/Cloudflare).
* **Pausar/Retomar a VPN no cliente local (se necessário):**
  ```bash
  sudo systemctl stop wg-quick@wg0     # Desconecta
  sudo systemctl start wg-quick@wg0    # Reconecta
  ```

---

## 7. Referências Relacionadas

* [Comparação: Tailscale vs WireGuard vs Cloudflare Tunnel](./vpn-comparision.md)
* [Ofuscação e Hardening de SSH na VPS](../../Guia_desenvolvimento_software/seguranca/ofuscando-ssh.md)


# Extra: Como adicionar mais um computador para gerenciamento remoto pelo WireGuard:

> Para redundãncia e segurança, para não ficar sem acesso à VPS (em caso de falha do computador principal, por exemplo), é interessante ter mais de um computador configurado no WireGuard.
Para adicionar um novo PC ao seu servidor WireGuard, você precisa criar um novo bloco [Peer] no final do arquivo de configuração da VPS e configurar o cliente no novo computador com um IP exclusivo.

## 1. Gere as chaves no novo PC

No novo computador que deseja conectar, abra o terminal e gere o par de chaves (pública e privada):
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```
* Guarde o conteúdo de `privatekey` para usar no arquivo `wg0.conf` do novo PC.
* Copie o conteúdo de `publickey` para colocar na VPS.

## 2. Atualize a configuração da VPS

Adicione um segundo bloco [Peer] ao arquivo da sua VPS (geralmente /etc/wireguard/wg0.conf), atribuindo o próximo IP disponível da sub-rede (10.10.0.3/32):
```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <CHAVE_PRIVADA_DA_VPS>

[Peer]
# Estação de trabalho Debian 13 Local
PublicKey = <CHAVE_PUBLICA_DO_SEU_DEBIAN_LOCAL>
AllowedIPs = 10.10.0.2/32

[Peer]
# Novo PC
PublicKey = <CHAVE_PUBLICA_DO_NOVO_PC>
AllowedIPs = 10.10.0.3/32
```
Reinicie o serviço na VPS para aplicar a mudança (ou use `wg syncconf` para não derrubar a conexão atual):
```bash
sudo systemctl restart wg-quick@wg0
```

## 3. Configure o cliente no Novo PC

No arquivo de configuração do WireGuard do novo computador, insira:
```ini
[Interface]
Address = 10.10.0.3/32
PrivateKey = <CHAVE_PRIVADA_DO_NOVO_PC>

[Peer]
PublicKey = <CHAVE_PUBLICA_DA_VPS>
Endpoint = <IP_PUBLICO_DA_VPS>:51820
AllowedIPs = 10.10.0.0/24
PersistentKeepalive = 25
```

