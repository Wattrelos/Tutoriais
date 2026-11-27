# Boas práticas com Cloudflare

> A solução ideal é restringir o acesso às portas 80 e 443 no UFW para aceitar conexões apenas das faixas de IP oficiais do Cloudflare, bloqueando todo o tráfego direto ao IP do servidor.
Quando o domínio passa pelo proxy do Cloudflare (nuvem laranja), todo tráfego legítimo chega ao seu servidor vindo dos servidores de borda do Cloudflare. Qualquer requisição nas portas 80 ou 443 vinda de outro IP é um atacante tentando contornar o WAF/DDoS do Cloudflare acessando o IP de origem diretamente.

## Passo 1: Liberar apenas os IPs do Cloudflare no UFW

Antes de fechar o acesso geral, adicione as faixas de IPv4 e IPv6 do Cloudflare. Você pode automatizar isso com um script simples (que também pode ser agendado no cron mensalmente, pois a lista raramente muda, mas pode ser atualizada):
```bash
#!/bin/bash
# Baixa as listas oficiais do Cloudflare
CF_IPV4=$(curl -s https://cloudflare.com)
CF_IPV6=$(curl -s https://cloudflare.com)

# Adiciona regras de permissão apenas para portas 80 e 443
for ip in $CF_IPV4 $CF_IPV6; do
    sudo ufw allow proto tcp from $ip to any port 80,443 comment 'Cloudflare IP'
done
```

## Passo 2: Remover as regras antigas abertas ao público

Verifique suas regras atuais numeradas e apague qualquer regra genérica que permita 80/tcp ou 443/tcp vindo de Anywhere.
* Listar regras numeradas:
```bash
sudo ufw status numbered
```
* Remover regras genéricas (substitua NUM pelo número da regra, sempre de baixo para cima para não alterar a numeração):
```bash
sudo ufw delete NUM
```
* Garantir política padrão de bloqueio:
```bash
sudo ufw default deny incoming
sudo ufw reload
```

## Passo 3: Monitorar tentativas de acesso direto ao IP

Com essa configuração, o Cloudflare continua recebendo todo tráfego legítimo sem gerar ruído no firewall. Enquanto isso, qualquer bot ou scanner que tentar acessar http://SEU_IP ou https://SEU_IP diretamente será barrado e registrado pelo UFW como UFW BLOCK nas portas 80 e 443.
* Filtrar tentativas diretas de acesso web (burladores de Cloudflare):
```bash
sudo journalctl -k --since "24 hours ago" | grep "UFW BLOCK" | grep -E "DPT=(80|443)\b"
```

## Camada extra recomendada (Authenticated Origin Pulls)

* Validação por certificado: Além do firewall, você pode ativar o recurso Authenticated Origin Pulls no painel do Cloudflare (em SSL/TLS > Origin Server) e configurar o Nginx ou Apache para exigir o certificado cliente do Cloudflare. Isso garante criptograficamente que mesmo um IP autorizado só consiga conversar com o servidor web se a requisição vier de fato da sua conta/rede Cloudflare.

