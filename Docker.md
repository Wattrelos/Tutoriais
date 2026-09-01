# Instalar o Docker no Debian 13 (Trixie)

_Ementa: o método mais recomendado e seguro é utilizar o repositório oficial do Docker. Isso garante que você receba sempre as atualizações de segurança mais recentes diretamente da fonte._

---

## 1. Atualize o sistema e instale as dependências
Abra o terminal e prepare o gerenciador de pacotes instalando as ferramentas necessárias para download e validação segura:

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg
```

## 2. Adicione a chave GPG oficial do Docker
Crie o diretório para chaves de segurança e baixe a chave oficial para que o Debian confie nos pacotes do Docker:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## 3. Adicione o repositório ao APT
Execute o comando abaixo para incluir a lista de fontes oficiais do Docker no seu sistema:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 4. Instale o Docker e o Docker Compose
Atualize o índice de pacotes novamente (agora incluindo o repositório do Docker) e instale o motor do Docker junto com os plug-ins essenciais, como o Compose:

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 5. Verifique a instalação
Para ter certeza de que o Docker está rodando corretamente, execute o contêiner de teste "hello-world":

```bash
sudo docker run hello-world
```

Se a instalação foi bem-sucedida, você verá uma mensagem de boas-vindas informando que o Docker foi baixado e executado corretamente.

---

## 💡 Dica opcional: Executar o Docker sem sudo
Por padrão, você precisa usar `sudo` para rodar comandos do Docker. Caso queira usá-lo com seu usuário comum, adicione seu perfil ao grupo do Docker:

```bash
sudo usermod -aG docker $USER
```

> **Nota:** É necessário deslogar e logar novamente no sistema para que essa alteração faça efeito.

---

# Interfaces Gráficas (GUI) para Docker

Para executar e gerenciar o Docker em modo gráfico no Debian/Linux, existem duas abordagens principais:
1. **Portainer** (Interface Web leve em contêiner - Recomendado para servidores e Linux nativo)
2. **Docker Desktop** (Aplicativo de desktop oficial com interface gráfica integrada)

---

## Opção 1: Portainer (Recomendado para Linux)
O Portainer é a ferramenta gráfica mais popular para gerenciar o Docker no Linux. Ele roda direto no navegador, não pesa no sistema e gerencia contêineres, imagens, volumes e redes de forma completa.

Execute o comando abaixo no terminal para criar o contêiner do Portainer:

```bash
sudo docker run -d -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

### Como acessar:
1. Abra o seu navegador de internet.
2. Acesse o endereço: `https://localhost:9443`
3. O navegador exibirá um aviso de segurança (normal para certificados locais autoassinados). Clique em **Avançado** e depois em **Continuar/Prosseguir**.
4. Crie sua senha de administrador e acesse o painel.

### Obter logs ou token do Portainer:
```bash
sudo docker logs portainer
```
Para filtrar tokens ou senhas nos logs:
```bash
sudo docker logs portainer 2>&1 | grep -i token
```

---

## Opção 2: Docker Desktop Oficial (Interface Gráfica de Desktop)
O **Docker Desktop** é o aplicativo oficial com interface gráfica tradicional para desktop.

### 1. Pré-requisito: Suporte a KVM (Virtualização)
O Docker Desktop no Linux roda uma VM interna com QEMU/KVM. Verifique se o KVM está habilitado:

```bash
# Verifica suporte a virtualização na CPU (deve retornar > 0)
egrep -c '(vmx|svm)' /proc/cpuinfo

# Carrega os módulos KVM
sudo modprobe kvm
sudo modprobe kvm_intel   # Se processador Intel
sudo modprobe kvm_amd     # Se processador AMD

# Adiciona seu usuário ao grupo kvm
sudo usermod -aG kvm $USER
```

### 2. Baixar o pacote oficial `.deb`
Baixe a versão mais recente do Docker Desktop para arquitetura x86_64 / amd64:

```bash
curl -O https://desktop.docker.com/linux/main/amd64/docker-desktop-amd64.deb
```

### 3. Instalar o Docker Desktop
Instale o pacote resolvendo automaticamente as dependências:

```bash
sudo apt update
sudo apt install ./docker-desktop-amd64.deb
```

### 4. Como iniciar o Docker Desktop:
- **Pelo menu do sistema:** Procure por **Docker Desktop** no menu de aplicativos da sua interface gráfica (GNOME, KDE, XFCE).
- **Pelo terminal:**
  ```bash
  systemctl --user start docker-desktop
  ```
- **Para iniciar automaticamente no login (opcional):**
  ```bash
  systemctl --user enable docker-desktop
  ```

---

## Qual escolher?

| Recurso | Portainer CE | Docker Desktop |
| :--- | :--- | :--- |
| **Tipo de Interface** | Web (Navegador) | Janela de Aplicativo Desktop |
| **Consumo de Recursos** | Mínimo (~50MB RAM) | Alto (Roda VM QEMU/KVM) |
| **Requisito KVM** | Não | Sim |
| **Ideal para** | Servidores, Linux nativo, máquinas com pouca RAM | Usuários acostumados com Windows/Mac e extensões Docker |
