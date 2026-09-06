# Instalação do Android Studio no Debian 13

Guia passo a passo para a instalação e configuração do **Android Studio** no Debian 13 (Trixie), incluindo a preparação do ambiente, bibliotecas de compatibilidade de 32 bits para o emulador e criação de atalho de inicialização.

---

## Passo 1: Habilitar o Multiarch (Suporte a 32 bits)

O emulador do Android e algumas ferramentas do SDK exigem bibliotecas de arquitetura 32 bits (`i386`) em sistemas x86_64. Habilite a arquitetura e atualize os repositórios:

```bash
sudo dpkg --add-architecture i386
sudo apt update
```

---

## Passo 2: Instalar as Dependências e o JDK

Instale o Java Development Kit (JDK) compatível e as bibliotecas essenciais de 32 bits atualizadas para o Debian 13:

```bash
sudo apt install -y default-jdk libc6:i386 libstdc++6:i386 zlib1g:i386 libncurses6:i386
```

---

## Passo 3: Download do Android Studio

1. Acesse a página oficial de download em [Android Developers](https://developer.android.com/studio).
2. Baixe o pacote oficial para Linux no formato `.tar.gz` (exemplo: `android-studio-*.tar.gz`).

---

## Passo 4: Extração e Instalação

Extraia o pacote baixado para o diretório `/opt` (recomendado para instalação global no sistema):

```bash
cd ~/Downloads
sudo tar -xzf android-studio-*.tar.gz -C /opt/
```

> **Dica de Permissão:** Se quiser que seu usuário tenha acesso direto sem necessidade de `sudo` para atualizações de plugins, você pode ajustar o proprietário da pasta:
> ```bash
> sudo chown -R $USER:$USER /opt/android-studio
> ```

---

## Passo 5: Inicialização e Assistente de Configuração

Inicie o assistente de configuração executando o script `studio.sh`:

```bash
/opt/android-studio/bin/studio.sh
```

Durante a primeira execução:
- Selecione o tipo de instalação (**Standard** ou **Custom**).
- O assistente fará o download automático do **Android SDK**, das **Platform-Tools** e do **Android Emulator**.

---

## Passo 6: Criar Atalho no Menu de Aplicativos (Desktop Entry)

Para abrir o Android Studio diretamente pelo menu de aplicativos ou inicializador do sistema:

1. Com o Android Studio aberto, vá no menu superior em:
   **Tools** > **Create Desktop Entry...**
2. Marque a opção para disponibilizar para todos os usuários (se desejar) e confirme.
