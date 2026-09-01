# Tutorial: Instalação Manual do MongoDB Community Server no Debian 13 (Trixie)

Este guia ensina como instalar o servidor oficial do MongoDB diretamente pelos pacotes `.deb`, contornando de forma limpa e segura as travas agressivas de chaves GPG e assinaturas do gerenciador `apt` no Debian 13.

---

## Passo 1: Entrar na pasta de Downloads e Baixar os Pacotes Oficiais

Abra o seu terminal com seu usuário comum e faça o download dos pacotes essenciais diretamente do servidor de repositórios da MongoDB:

```bash
cd ~/Downloads

# 1. Baixar o Servidor Principal (Core Server)
wget https://mongodb.org

# 2. Baixar o Shell de Comandos (Opcional - para gerenciar via terminal)
wget https://mongodb.org

# 3. Baixar os Utilitários de Banco (Opcional - ferramentas de backup/restauração)
wget https://mongodb.org
```

## Passo 2: Instalar os Arquivos `.deb` Localmente

Use o comando `apt install` passando o caminho dos arquivos baixados. O sistema fará a descompactação e instalará as ferramentas necessárias:

```bash
sudo apt install ./mongodb-org-server_8.0.0_amd64.deb ./mongodb-mongosh_8.0.0_amd64.deb ./mongodb-database-tools_8.0.0_amd64.deb
```
*(Nota: Avisos do terminal sobre o usuário `_apt` e "Permissão negada" ao final da instalação podem ser ignorados. Eles ocorrem apenas porque o arquivo está dentro de uma pasta pessoal do usuário, mas a instalação é concluída com sucesso via root).*

---

## Passo 3: Iniciar e Ativar o Serviço do MongoDB

Por padrão, os serviços do Linux são instalados em estado "desligado". Execute os comandos abaixo para iniciar o banco de dados e configurá-lo para ligar automaticamente junto com o computador:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

## Passo 4: Verificar se o Servidor está Ativo

Antes de tentar abrir qualquer interface visual, valide se o motor do banco de dados está rodando corretamente em segundo plano:

```bash
sudo systemctl status mongod
```
*Procure pela linha que indica textualmente: `Active: active (running)` em verde.*

---

## Passo 5: Conectar na Interface Gráfica (MongoDB Compass)

1. Abra o aplicativo **MongoDB Compass** no menu do seu sistema operacional.
2. Na tela inicial, o campo de texto **URI** já virá preenchido com o endereço padrão de conexões locais: `mongodb://localhost:27017`
3. Clique no botão verde **Connect**.

Pronto! Você agora está conectado ao seu servidor MongoDB local e pronto para criar seus bancos de dados, coleções e documentos de forma 100% visual.
