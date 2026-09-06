# Instalação do Antigravity & Antigravity IDE no Debian 13

Guia passo a passo para a instalação e configuração do **Antigravity & Antigravity IDE** no Debian 13 (Trixie). O Antigravity é um ambiente de desenvolvimento avançado potencializado por inteligência artificial, oferecendo completação inteligente de código, depuração assistida e integração nativa com fluxos de desenvolvimento.

---

# Primeiro, instação do Antigravity
## 1. Download do Antigravity & Antigravity IDE

Acesse a página oficial de download:
- **Site oficial:** [https://antigravity.google/product/antigravity-ide/](https://antigravity.google/product/antigravity-ide/)
- Clique no botão **Download** e selecione a versão para Linux (x64).

Você pode escolher entre duas opções de pacote:
1. **Pacote `.deb` (Recomendado para Debian/Ubuntu):** Instalação automática com gerenciamento de pacotes, atalhos de menu e binários configurados automaticamente.
2. **Pacote compactado `.tar.gz`:** Instalação manual portátil (ideal para quem prefere isolar a instalação no diretório `/opt`).

---

## Opção 1: Instalação via pacote `.deb` (Recomendado)

Esta é a maneira mais simples e limpa de instalar no Debian, pois configura automaticamente o comando no terminal e o ícone no menu de aplicativos.

1. Acesse o diretório de downloads:
   ```bash
   cd ~/Downloads
   ```

2. Instale o pacote utilizando o `apt` (ele cuidará da instalação e de eventuais dependências):
   ```bash
   sudo apt install ./antigravity*.deb
   ```

> **Pronto!** O Antigravity já estará disponível no menu de aplicativos e você também poderá executá-lo pelo terminal digitando:
> ```bash
> antigravity
> ```

---

## Opção 2: Instalação manual via arquivo compactado (`.tar.gz`)

Caso você tenha baixado o arquivo `.tar.gz`, siga os passos abaixo para instalar no diretório `/opt`.

### Passo 1: Extrair os arquivos para `/opt/`

Abra o terminal e execute a extração com privilégios administrativos (`sudo`):

```bash
cd ~/Downloads
sudo tar -xzf Antigravity*.tar.gz -C /opt/
```

### Passo 2: Renomear o diretório

Para padronizar o caminho de instalação, renomeie a pasta extraída:

```bash
sudo mv /opt/Antigravity* /opt/antigravity
```

### Passo 3: Ajustar permissões

Defina o seu usuário como proprietário da pasta (para facilitar atualizações de plugins) ou conceda permissão de execução:

```bash
sudo chown -R $USER:$USER /opt/antigravity
sudo chmod +x /opt/antigravity/bin/antigravity
```

### Passo 4: Criar link simbólico para o terminal

Para conseguir abrir o editor em qualquer diretório pelo terminal usando `antigravity .`:

```bash
sudo ln -sf /opt/antigravity/bin/antigravity /usr/local/bin/antigravity
```

### Passo 5: Criar o ícone do sistema

Copie o ícone da aplicação para a pasta de ícones do sistema:

```bash
sudo cp /opt/antigravity/resources/app/resources/linux/code.png /usr/share/pixmaps/antigravity.png
```

### Passo 6: Criar o atalho no menu de aplicativos (`.desktop`)

Crie o arquivo de inicialização para que o Antigravity apareça no menu de programas:

```bash
sudo nano /usr/share/applications/antigravity.desktop
```

Cole o seguinte conteúdo no arquivo:

```ini
[Desktop Entry]
Name=Antigravity
Comment=Experience liftoff
GenericName=Text Editor
Exec=/opt/antigravity/bin/antigravity %F
Icon=antigravity
Type=Application
StartupNotify=false
StartupWMClass=Antigravity
Categories=TextEditor;Development;IDE;
MimeType=application/x-antigravity-workspace;
Actions=new-empty-window;
Keywords=vscode;antigravity;ide;

[Desktop Action new-empty-window]
Name=New Empty Window
Name[pt_BR]=Nova Janela Vazia
Name[es]=Nueva ventana vacía
Exec=/opt/antigravity/bin/antigravity --new-window %F
Icon=antigravity
```

Para salvar no `nano`: pressione `Ctrl + O` e `Enter`, depois saia com `Ctrl + X`.

Atualize o banco de dados dos aplicativos:

```bash
sudo update-desktop-database
```

---

## Verificação da Instalação

Abra o terminal e teste a inicialização abrindo a pasta atual:

```bash
antigravity .
```


# Como atualizar o Antigravity & Antigravity IDE
