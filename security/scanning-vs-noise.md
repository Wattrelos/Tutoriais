# Como automatizar o filtro de logs do UFW para identificar padrões de varredura e diferenciar de ruído comum da internet.

> É totalmente possível usar a saída do journalctl do UFW para identificar IPs recorrentes e diferenciar varreduras direcionadas de ruído comum da internet.

## Comandos para agrupar e analisar o journalctl

Você pode combinar o journalctl com ferramentas nativas do Linux (grep, awk, sort, uniq) para extrair estatísticas rápidas dos últimos eventos:
* Top 15 IPs mais bloqueados (geral):
```bash
sudo journalctl -k --since "24 hours ago" | grep "UFW BLOCK" | grep -oP 'SRC=\K\S+' | sort | uniq -c | sort -nr | head -n 15
```
* Cruzar IP de origem com portas de destino (SRC + DPT):
```bash
sudo journalctl -k --since "24 hours ago" | grep "UFW BLOCK" | awk '{for(i=1;i<=NF;i++){if($i~/^SRC=/){src=$i} if($i~/^DPT=/){dpt=$i}}} print src, dpt' | sort | uniq -c | sort -nr | head -n 20
```
* Contar quantas portas diferentes cada IP tentou acessar (ideal para achar port scans):
```bash
sudo journalctl -k --since "24 hours ago" | grep "UFW BLOCK" | awk '{for(i=1;i<=NF;i++){if($i~/^SRC=/){src=$i} if($i~/^DPT=/){dpt=$i}}} print src, dpt' | sort -u | awk '{print $1}' | uniq -c | sort -nr | head -n 15
```

## Como distinguir ruído de varreduras ou ataques direcionados

* Ruído de fundo (Internet Background Noise): Caracteriza-se por poucas tentativas esparsas vindas de milhares de IPs diferentes. Geralmente focam em portas padrão de alto volume (como 22, 23, 80, 443, 3389 ou 8080) com intervalo irregular.
* Varredura horizontal / vertical (Port Scan): Um mesmo IP (SRC) aparece dezenas ou centenas de vezes em poucos minutos testando várias portas distintas (DPT) em sequência ou aleatoriamente.
* Força bruta ou reconhecimento persistente: O mesmo IP insiste repetidamente em uma única porta específica ao longo de várias horas ou dias, gerando alta contagem para o mesmo par SRC e DPT.
* Pacotes anômalos: Presença frequente de flags TCP incomuns (como FIN, URG, PSH sem ACK, ou RES diferente de 0x00), típicos de reconhecimento ativo com ferramentas como Nmap ou Masscan.

## Próximo passo prático para automação

* Bloqueio automático: Se o objetivo for mitigar esses IPs recorrentes sem análise manual constante, ferramentas como o Fail2Ban (com a jail ufw ou recidive) ou o CrowdSec leem exatamente esses padrões do journalctl e aplicam bloqueios temporários ou permanentes.
Se você quiser automatizar essa triagem, posso ajudar a montar um script em bash para gerar um relatório diário ou configurar uma regra do Fail2Ban focada nos bloqueios do UFW. Qual caminho faz mais sentido para o seu servidor?
