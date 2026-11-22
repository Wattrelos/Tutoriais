# 🔍 Comparação de Acesso Privado à VPS: Tailscale vs WireGuard vs Cloudflare Tunnel

> **Contexto Skanstull:** Infraestrutura com **Debian 13 (Trixie)** na estação local e na VPS (Hostinger com IPv4 público fixo), **Cloudflare + Nginx** já atuando na borda HTTP/HTTPS (Storefront, `grafana.agsonhos.com.br`, `prometheus.agsonhos.com.br`), **Forgejo** (Git) e serviços internos isolados em `127.0.0.1` (`3000`, `3001`, `3002`, `3306`, `5672`, `6379`, `9090`, `27017`).

Todas as três soluções abaixo permitem **fechar 100% da porta pública do SSH (`22`) no UFW da VPS**, eliminando ataques de força bruta e *scanners* automatizados. A diferença está em soberania, latência, dependência externa e facilidade de operação.

---

## 1. Comparação Técnica (Debian 13 Local ↔ Debian 13 VPS)

| Critério | Tailscale | WireGuard (Puro) | Cloudflare Tunnel (`cloudflared`) |
| :--- | :--- | :--- | :--- |
| **Arquitetura** | Malha P2P (baseada em WireGuard) com plano de controle na nuvem. | VPN ponto-a-ponto direta e nativa no kernel Linux (`wireguard-tools`). | Túnel de saída (*outbound*) via rede de borda da Cloudflare. |
| **Portas abertas no UFW** | **Nenhuma** (atravessa NAT via UDP automaticamente). | **1 porta UDP silenciosa** na VPS (ex.: `51820/udp`). | **Nenhuma** (apenas conexão de saída HTTPS/QUIC). |
| **Dependência de Terceiros** | **Alta** (login OAuth Google/GitHub/Microsoft e coordenação Tailscale). | **Zero** (100% soberano, trafega apenas entre suas máquinas). | **Alta** (depende da conta e infraestrutura Cloudflare). |
| **Latência & Consumo** | Muito baixa (P2P direto após *handshake*) / daemon Go leve. | **Mínima possível** (direto no kernel, consumo próximo de zero). | Média (tráfego passa pelos POPs da Cloudflare). |
| **Setup no Debian 13** | Muito fácil (repositório oficial + `tailscale up`). | Médio (gerar chaves + criar `/etc/wireguard/wg0.conf`). | Médio (instalar `cloudflared` na VPS + `ProxyCommand` no `~/.ssh/config`). |
| **Segurança Extra** | ACLs por usuário/dispositivo + *Tailscale SSH*. | Chaves criptográficas estáticas + isolamento total de sub-rede (`10.10.0.0/24`). | *Cloudflare Access* (exige OTP/2FA no navegador antes do SSH). |
| **Túneis DB / Git SSH** | Excelente (IP estável na rede Tailscale). | Excelente (IP fixo `10.10.0.1` + túneis `3306`/`27017` sem latência extra). | Requer configurar `cloudflared access tcp` por porta ou `ProxyCommand`. |

---

## 2. Qual Escolher no Cenário Skanstull?

* **Escolha WireGuard (Puro) — *Recomendado para o Skanstull* — se:**
  * A VPS possui **IPv4 público fixo** (como na Hostinger) e você valoriza **soberania total** sem depender de provedores de identidade externos (Google/GitHub).
  * Você quer a **menor latência e consumo de RAM/CPU possíveis** no Debian 13, mantendo túneis SSH rápidos para MySQL (`3306`), MongoDB (`27017`) e `git push` no Forgejo.
  * 👉 Veja o passo a passo completo em [`wireguard-install-config.md`](./wireguard-install-config.md).

* **Escolha Tailscale se:**
  * Você precisa adicionar vários dispositivos/membros da equipe rapidamente sem gerenciar chaves manuais ou precisa atravessar redes com **CGNAT / roteadores restritos** onde o IP do servidor muda ou não é acessível diretamente.

* **Escolha Cloudflare Tunnel (`cloudflared`) se:**
  * Você precisa acessar a VPS a partir de **redes corporativas muito restritas** que bloqueiam tráfego UDP de VPNs (liberando apenas `443/TCP`) ou deseja exigir autenticação web (*Cloudflare Zero Trust / Access*) antes mesmo de o pacote SSH alcançar o daemon OpenSSH.

---

## 3. Referências Relacionadas

* [Instalação e Configuração do WireGuard (Debian 13 ↔ VPS)](./wireguard-install-config.md)
* [Ofuscação e Hardening de SSH na VPS](../../Guia_desenvolvimento_software/seguranca/ofuscando-ssh.md)
