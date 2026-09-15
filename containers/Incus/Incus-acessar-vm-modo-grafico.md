# Guia para acessar uma VM no Incus em modo gráfico 


## 1. Instalar o visualizador no Debian 13 (Host)
No terminal do seu sistema principal (Debian 13), instale o cliente gráfico necessário:
```bash
sudo apt update
sudo apt install virt-viewer
```

> [!IMPORTANT]
> **Pré-requisito obrigatório:** O console VGA nativo (`--type=vga`) **funciona exclusivamente em Máquinas Virtuais** criadas com a flag `--vm` (ex: `incus launch images:debian/13 modelo-desktop --vm`). Se a instância for um contêiner convencional (LXC), o comando retornará: `Erro: VGA console is only supported by virtual machines`.
> Além disso, certifique-se de ter o pacote `spice-vdagent` instalado dentro da VM para integração suave do mouse e resolução dinâmica.

## 2. Acessar a VM em modo gráfico
Certifique-se de que a VM está rodando. Em seguida, execute o comando abaixo no terminal do seu Debian 13 para abrir a tela da VM:
```bash
incus console modelo-desktop --type=vga
```
Uma janela do virt-viewer será aberta automaticamente exibindo a interface gráfica da sua VM.


## Alternativa: Acesso via RDP ou VNC (Rede)
Se preferir não usar o console nativo, você pode instalar um servidor de desktop remoto dentro da VM (usando o comando que você citou) e conectar via rede:

   1. Acesse o terminal da VM:
```bash   
   incus exec modelo-desktop -- bash
```   
   2. Instale um servidor RDP (ex: XRDP):
```bash   
   apt update && apt install xrdp -y
   systemctl enable --now xrdp
```   
   3. Descubra o IP da VM:
```bash   
   incus list
```   
   4. Conecte do Debian 13: Use um cliente como o Remmina para se conectar ao IP da VM via RDP.

