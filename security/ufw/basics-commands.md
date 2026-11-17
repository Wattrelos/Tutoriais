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
```
```bash
sudo ufw status
```
Assim, a porta 8089 ficará acessível apenas internamente via 127.0.0.1 para o Nginx, bloqueando conexões diretas de fora do servidor.
Você já definiu qual nova porta externa vai liberar no UFW (como a 2053, 2083 ou 8444) para o Nginx? Se quiser, já te passo o comando completo de liberação e o bloco final ajustado.
