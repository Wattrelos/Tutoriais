# Comandos

## Comandos úteis no Linux Debian 13    

### Copiar pastas

O comando correto para copiar uma pasta inteira no Debian 13 (incluindo subpastas e arquivos ocultos) sem sobrescrever os arquivos já existentes no destino é o cp com as opções -a (arquivamento/preservação) e -n (não sobrescrever).
Execute o seguinte comando no terminal:

```bash
cp -an /caminho/da/pasta_origem/. /caminho/da/pasta_destino/
```

### O que cada parte do comando faz:

* -a (archive): Copia de forma recursiva (tudo dentro da pasta), preservando links simbólicos, permissões, donos e atributos dos arquivos. Ele garante que os arquivos ocultos (que começam com .) sejam copiados.
* -n (no-clobber): Impede que arquivos no destino sejam sobrescritas. Se um arquivo já existir, ele será pulado silenciosamente.
* /. (ponto após a origem): É o segredo para copiar o conteúdo da pasta de origem (incluindo os itens ocultos) diretamente para dentro da pasta de destino, em vez de copiar a pasta mãe em si.

Se você precisar ver exatamente quais arquivos estão sendo copiados na tela, me avise para eu te mostrar como ativar o modo detalhado ou como criar um log dessa cópia.

### Testar Redis
O comando principal para testar se o Redis está ativo no Debian 13 é o redis-cli ping.
Aqui estão as opções mais comuns para verificar o status:
## 1. Testar a resposta do serviço (Mais direto)
Execute o comando abaixo. Se o Redis estiver funcionando, ele retornará PONG:
```bash
redis-cli ping
```
Se o seu Redis exigir senha, use: redis-cli -a sua_senha ping
## 2. Verificar o status do sistema (systemd)
Para ver se o processo do serviço está rodando em segundo plano no sistema:
```bash
sudo systemctl status redis-server
```
* Procure pela linha que diz Active: active (running).

### Verificar a porta de rede
O Redis utiliza por padrão a porta 6379. Você pode testar se ela está aberta e respondendo:
```bash
ss -tuln | grep 6379
```
### Ativar Redis no boot
Se o redis-server.service estiver carregado mas desabilitado, para ativar o Redis no boot, execute o seguinte comando:
```bash
sudo systemctl enable redis-server
``` 
### Desabilitar Redis no boot
Se o redis-server.service estiver carregado mas habilitado, para desabilitar o Redis no boot, execute o seguinte comando:
```bash
sudo systemctl disable redis-server
``` 
### Parar o Redis
Se o redis-server.service estiver carregado mas ativo, para parar o Redis, execute o seguinte comando:
```bash
sudo systemctl stop redis-server
``` 
### Iniciar o Redis
Se o redis-server.service estiver carregado mas inativo, para iniciar o Redis, execute o seguinte comando:
```bash
sudo systemctl start redis-server
``` 
### Reiniciar o Redis
Se o redis-server.service estiver carregado mas ativo, para reiniciar o Redis, execute o seguinte comando:
```bash
sudo systemctl restart redis-server
``` 
### Verificar se o Redis está rodando
Se o redis-server.service estiver carregado mas ativo, para verificar se o Redis está rodando, execute o seguinte comando:
```bash
sudo systemctl is-active redis-server
``` 
### Verificar se o Redis está habilitado
Se o redis-server.service estiver carregado mas desabilitado, para verificar se o Redis está habilitado, execute o seguinte comando:
```bash
sudo systemctl is-enabled redis-server
``` 
### Verificar se o Redis está desabilitado
Se o redis-server.service estiver carregado mas habilitado, para verificar se o Redis está desabilitado, execute o seguinte comando:
```bash
sudo systemctl is-disabled redis-server
``` 
### Verificar o status do Redis
Se o redis-server.service estiver carregado, para verificar o status do Redis, execute o seguinte comando:
```bash
sudo systemctl status redis-server
``` 
### Verificar se o Redis está carregado
Se o redis-server.service não estiver carregado, para verificar se o Redis está carregado, execute o seguinte comando:
```bash
sudo systemctl is-loaded redis-server
``` 
### Verificar se o Redis está descarregado
Se o redis-server.service estiver carregado, para verificar se o Redis está descarregado, execute o seguinte comando:
```bash
sudo systemctl is-failed redis-server
``` 
