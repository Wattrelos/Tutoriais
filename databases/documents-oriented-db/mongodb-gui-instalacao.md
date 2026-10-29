# Tutorial: Instalação e Configuração do MongoDB Compass no Debian 13 (Trixie)

Este guia prático ensina como instalar a interface gráfica oficial do MongoDB (Compass) no Debian 13 e como resolver o erro comum de inicialização via usuário Root.

---

## Passo 1: Atualizar o Sistema e Instalar Dependências

Abra o terminal e garanta que o gerenciador de pacotes e as ferramentas de download estejam atualizados. O utilitário `gdebi` é usado porque resolve automaticamente dependências que possam faltar no Debian.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget gdebi-core -y
```

## Passo 2: Baixar o MongoDB Compass

Acesse a pasta de Downloads do seu usuário e baixe o pacote oficial `.deb` (arquitetura AMD64/64-bit):

```bash
cd ~/Downloads
wget https://downloads.mongodb.com/compass/mongodb-compass_1.49.14_amd64.deb
```
*(Nota: Se precisar da versão mais recente no futuro, consulte a [Página de Downloads Oficial da MongoDB](https://mongodb.com)).*

## Passo 3: Instalar o Pacote `.deb`

Execute a instalação utilizando o `gdebi`. Quando o terminal perguntar se deseja confirmar a instalação, digite **S** (ou **y**):

```bash
sudo gdebi mongodb-compass_1.49.14_amd64.deb
```

---

## Passo 4: Como Inicializar Corretamente (Evitando Erros)

Aplicativos baseados em Electron (como o Compass) **não permitem** a execução direta pelo usuário `root` por motivos de segurança do navegador embutido (Sandbox).

### Forma Correta (Usuário Comum)
Se você estiver logado como `root` no terminal, saia dele antes de abrir o programa:

```bash
exit
mongodb-compass
```
*O MongoDB Compass também estará disponível diretamente no menu de aplicativos do seu ambiente gráfico (GNOME, KDE, XFCE, etc.).*

### Alternativa para Forçar Modo Root (Apenas se necessário)
Se você precisar obrigatoriamente rodar o comando dentro do terminal `root`, use a flag de segurança abaixo:

```bash
mongodb-compass --no-sandbox
```

---

## Próximos Passos Recomendados
* **Conexão Local:** Se o servidor MongoDB estiver na sua própria máquina, use a string padrão: `mongodb://localhost:27017`
* **Conexão Nuvem (Atlas):** Cole a string de conexão fornecida pelo painel do MongoDB Atlas (começando com `mongodb+srv://...`).
