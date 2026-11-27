# Comandos básicos do UFW


## Revogar permissões
- Para revogar a permissão da porta 8089 no UFW, você pode usar o comando delete diretamente ou remover pela numeração da regra.

1. Método 1: Pelo comando direto (Mais rápido)

Execute o comando correspondente ao protocolo que você liberou anteriormente (geralmente TCP ou ambos):
* Se você liberou apenas TCP (8089/tcp):
```bash
sudo ufw delete allow 8089/tcp
```
* Se você liberou sem especificar o protocolo (8089):
```bash
sudo ufw delete allow 8089
```

2. Método 2: Pelo número da regra (Mais seguro)

Se houver regras específicas por IP ou se o comando direto não encontrar a regra exata, liste todas as regras numeradas:
1. Liste as regras com números:
```bash
sudo ufw status numbered
```
2. Identifique o número entre colchetes [ ] correspondente à porta 8089 (lembre-se de verificar se há duas linhas: uma para IPv4 e outra para IPv6).
3. Apague pelo número (substitua X pelo número da regra):
```bash
sudo ufw delete X
```
(Atenção: ao apagar uma regra pelo número, a numeração das restantes muda. Sempre rode sudo ufw status numbered novamente antes de apagar uma segunda regra).

Verificação final

Após remover, confira se a porta sumiu da lista de permissões ativas:

```bash
sudo ufw status
```

# Logs:

Para ver o log do UFW em tempo real, use:
```bash
sudo tail -f /var/log/ufw.log
# caso o arquivo dedicado não exista:
sudo journalctl -k -f | grep -i ufw 
```

## Comandos principais

* Ver o arquivo de log: `sudo less /var/log/ufw.log`
* Acompanhar em tempo real: `sudo tail -f /var/log/ufw.log`
* Buscar nos logs do sistema / systemd: `sudo journalctl -k | grep -i ufw` ou `sudo grep -i ufw /var/log/syslog`
* Filtrar bloqueios ou um IP específico: `sudo grep "UFW BLOCK" /var/log/ufw.log` ou `sudo grep "SRC=203.0.113.5" /var/log/ufw.log`

## Se o log estiver vazio ou ausente

* Checar status e nível de log: sudo ufw status verbose
* Ativar o registro: sudo ufw logging on (ou defina low, medium, high ou full)
* Verificar serviço de log: sudo service rsyslog status (se o arquivo /var/log/ufw.log não for criado)

Campos mais comuns no log

* UFW BLOCK / UFW ALLOW: indica se o pacote foi bloqueado ou permitido.
* IN / OUT: interface de rede de entrada ou saída.
* SRC / DST: endereços IP de origem e destino.
* PROTO: protocolo (TCP, UDP, etc.).
* SPT / DPT: porta de origem e porta de destino.

