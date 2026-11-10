# Tutorial: Instalação e Configuração do MongoDB Community Server no Debian 13 (Trixie)

Este guia ensina como instalar o servidor oficial do **MongoDB Community** diretamente via pacotes `.deb` (contornando travas de chaves GPG no Debian 13), configurar autenticação de usuários, liberar acesso remoto seguro com restrição de rede via firewall (UFW), conectar com o **MongoDB Compass** e estruturar variáveis de ambiente (`.env`) para sua aplicação.

---

## 1. Instalação Manual dos Pacotes Oficiais (.deb)

### 1.1. Baixar os pacotes oficiais
Abra o terminal com seu usuário comum e faça o download dos pacotes essenciais diretamente do repositório da MongoDB:

```bash
cd ~/Downloads

# 1. Servidor Principal (Core Server)
wget https://repo.mongodb.org/apt/debian/dists/bookworm/mongodb-org/8.0/main/binary-amd64/mongodb-org-server_8.0.4_amd64.deb

# 2. Shell de Comandos (Opcional - gerenciamento via terminal)
wget https://repo.mongodb.org/apt/debian/dists/bookworm/mongodb-org/8.0/main/binary-amd64/mongodb-mongosh_2.3.8_amd64.deb

# 3. Utilitários de Banco (Opcional - ferramentas de backup e restauração)
wget https://repo.mongodb.org/apt/debian/dists/bookworm/mongodb-org/8.0/main/binary-amd64/mongodb-database-tools_100.10.0_amd64.deb
```

---

### 1.2. Instalar os arquivos .deb localmente
Instale os pacotes baixados utilizando o `apt`:

```bash
sudo apt install ./mongodb-org-server_8.0.4_amd64.deb ./mongodb-mongosh_2.3.8_amd64.deb ./mongodb-database-tools_100.10.0_amd64.deb
```

> [!NOTE]
> Avisos do terminal sobre o usuário `_apt` e "Permissão negada" ao final da instalação podem ser desconsiderados. Ocorrem apenas porque os arquivos estão no diretório pessoal do usuário, mas a instalação é concluída com sucesso via root.

---

### 1.3. Iniciar e habilitar o serviço
Por padrão, os serviços no Debian são instalados desligados. Inicie o banco e configure-o para inicializar automaticamente com o sistema:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

---

### 1.4. Verificar o status do servidor
Valide se o serviço está ativo e em execução:

```bash
sudo systemctl status mongod
```

*Verifique se a saída contém a indicação `Active: active (running)`.*

---

### 1.5. Teste de conexão local inicial
Para validar o funcionamento em uma interface gráfica local instalada na mesma máquina:

1. Abra o **MongoDB Compass** (se ainda não tiver instalado, veja o [Guia de Instalação do MongoDB Compass](mongodb-gui-instalacao.md)).
2. Utilize o endereço padrão: `mongodb://localhost:27017`
3. Clique em **Connect**.

---

## 2. Configuração de Acesso Remoto Seguro e Autenticação

Para expor o MongoDB a outros dispositivos da rede local com segurança, é indispensável criar um usuário administrador antes de ativar o controle de acesso por autenticação.

### 2.1. Criar o usuário administrador no mongosh
Acesse o terminal interativo do MongoDB enquanto a autenticação ainda está desativada:

```bash
mongosh
```

Dentro do shell, mude para o banco `admin` e crie o usuário administrativo:

```javascript
use admin

db.createUser({
  user: "adminUser",
  pwd: passwordPrompt(), // Digite a senha segura quando solicitado no terminal
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```

Saia do shell do MongoDB:

```javascript
exit
```

---

### 2.2. Configurar rede e autenticação no `mongod.conf`
Abra o arquivo de configuração principal:

```bash
sudo nano /etc/mongod.conf
```

Localize as seções `net` e `security` e ajuste-as conforme abaixo (substitua `192.168.1.50` pelo IP estático do seu servidor):

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.1.50

security:
  authorization: "enabled"
```

> [!TIP]
> Caso queira que o MongoDB escute em todas as interfaces de rede da máquina, você pode utilizar `bindIp: 0.0.0.0` (sempre mantendo o firewall ativo para restringir a origem).

Salve com `Ctrl + O`, confirme com `Enter` e saia com `Ctrl + X`.

---

### 2.3. Reiniciar e validar o serviço
Aplique as alterações reiniciando o `mongod`:

```bash
sudo systemctl restart mongod
sudo systemctl status mongod
```

---

### 2.4. Liberar a porta no firewall (UFW)
Restrinja o acesso à porta `27017` exclusivamente para os dispositivos da sua faixa de rede local (exemplo: `192.168.1.0/24`):

```bash
sudo ufw allow from 192.168.1.0/24 to any port 27017 proto tcp
sudo ufw reload
```

Verifique se a regra foi aplicada corretamente:

```bash
sudo ufw status
```

---

### 2.5. Testar a string de conexão (URI)
A partir de outra máquina na mesma rede, a String de Conexão (URI) autenticada será:

```text
mongodb://adminUser:SUA_SENHA@192.168.1.50:27017/?authSource=admin
```

---

## 3. Conexão Remota via MongoDB Compass

Ao conectar remotamente usando o menu de opções avançadas (**Advanced Connection Options**), a falha mais comum é a falta de indicação do banco de autenticação (`admin`) ou a opção `Direct Connection` desmarcada.

### 3.1. Preenchimento das Opções Avançadas

#### 1. Aba General
- **Connection String Scheme:** Selecione `mongodb` *(não use `mongodb+srv`, pois é exclusivo para clusters gerenciados na nuvem, como MongoDB Atlas)*.
- **Host:** Informe o IP e a porta do servidor (ex.: `192.168.1.50:27017`).
- **Direct Connection:** Marque como **Ativado** *(evita que o Compass tente descobrir nós de Replica Set em instâncias standalone)*.

#### 2. Aba Authentication
- **Authentication Method:** Selecione `Username / Password`.
- **Username:** `adminUser` (ou o usuário que você configurou).
- **Password:** A senha definida durante a criação.
- **Authentication Database:** `admin` *(obrigatório, pois o usuário administrador foi criado no banco `admin`)*.
- **Authentication Mechanism:** `Default` (ou `SCRAM-SHA-256`).

#### 3. Aba TLS/SSL
- **SSL/TLS Connection:** Deixe em `Off` *(a não ser que você tenha configurado certificados TLS/SSL no servidor)*.

---

### 3.2. Diagnóstico e Resolução de Erros Comuns

| Mensagem de Erro no Compass | Causa Provável | Solução |
| :--- | :--- | :--- |
| `Authentication failed` | Usuário, senha ou **Authentication Database** (`admin`) incorretos. | Revise as credenciais e garanta que o banco de autenticação esteja preenchido como `admin`. |
| `connect ECONNREFUSED` | O servidor recusou a conexão na porta `27017`. | Verifique se o serviço está ativo (`systemctl status mongod`) e se o IP do servidor está configurado em `bindIp` no `/etc/mongod.conf`. |
| `Server selection timed out` | Os pacotes foram descartados/bloqueados antes de alcançar o servidor. | Verifique as regras de firewall (`sudo ufw status`) e confirme se o IP da máquina cliente está dentro da faixa permitida. |

---

## 4. Integração com Aplicações (.env)

Para gerenciar a conexão com o MongoDB de forma limpa e segura em projetos (Node.js, Python, PHP, etc.), separe os parâmetros em variáveis de ambiente no arquivo `.env`:

### 4.1. Estrutura recomendada para o `.env`

```env
# Configurações individuais de conexão
DB_HOST=192.168.1.50
DB_PORT=27017
DB_NAME=nome_do_seu_banco
DB_USER=adminUser
DB_PASS=suaSenhaSegura123
DB_AUTH_SOURCE=admin

# URI completa (útil para ORMs/ODMs como Mongoose, Prisma, TypeORM ou PyMongo)
MONGODB_URI=mongodb://${DB_USER}:${DB_PASS}@${DB_HOST}:${DB_PORT}/${DB_NAME}?authSource=${DB_AUTH_SOURCE}&directConnection=true
```

---

### 4.2. O que cada variável representa
- `DB_HOST`: Endereço IP ou hostname do servidor MongoDB (`192.168.1.50`).
- `DB_PORT`: Porta TCP do serviço (`27017`).
- `DB_NAME`: Nome do banco de dados específico onde a aplicação fará operações de leitura e escrita.
- `DB_USER` e `DB_PASS`: Credenciais de autenticação. Se a senha contiver caracteres especiais (`@`, `:`, `/`, `#`, `%`), utilize URL encoding (`encodeURIComponent`).
- `DB_AUTH_SOURCE`: Banco onde as credenciais foram criadas (`admin` para usuários administrativos ou o próprio `DB_NAME` para usuários de escopo local).
- `directConnection=true`: Força conexão direta à instância standalone, evitando timeouts na tentativa de descobrir Replica Sets.

---

### 4.3. Boas práticas de segurança
- **Nunca comite o `.env` no Git:** Mantenha o `.env` listado no seu `.gitignore` para evitar vazamento de credenciais no repositório.
- **Mantenha um `.env.example`:** Disponibilize um arquivo modelo com valores fictícios para orientar a configuração por outros membros da equipe.
- **Princípio do menor privilégio em produção:** Evite conectar sua aplicação como `adminUser`. Crie um usuário com permissão restrita de leitura e escrita apenas no banco da aplicação:
  ```javascript
  use nome_do_seu_banco
  db.createUser({
    user: "appUser",
    pwd: passwordPrompt(),
    roles: [{ role: "readWrite", db: "nome_do_seu_banco" }]
  })
  ```
