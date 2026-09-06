# Instalar o Steam no Debian 13 KDE

- Para in   stalar o Steam no Debian 13 (Trixie), o método recomendado envolve habilitar os repositórios contrib e non-free, adicionar a arquitetura multi-arquitetura i386, e então usar o apt para instalar o pacote steam-installer, permitindo que ele baixe dependências e o cliente Steam. Você também pode optar por instalar via Flatpak para uma solução mais isolada. 

Método 1: Via Repositórios Debian (Recomendado)

- Editar sources.list: Abra o terminal e edite o arquivo de fontes do apt para incluir: contrib, non-free e non-free-firmware para o Debian 13 (Trixie):

```bash
nano /etc/apt/sources.list
```


- Adicione contrib non-free non-free-firmware ao final das linhas que começam com deb e deb-src.
- Adicionar 	arquitetura i386: Habilite o suporte para pacotes de 32 	bits.

```bash
dpkg --add-architecture i386
```


- Atualizar 	o sistema: Sincronize os novos repositórios e pacotes.

```bash
apt update
```


- Instalar o 	Steam: Instale o pacote steam-installer 	(ou steam).

```bash
bash
apt install steam-installer
```


- Executar: Abra o Steam pelo 	menu de aplicativos ou execute steam 	no terminal. Ele fará uma instalação adicional de runtime.
