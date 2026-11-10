# Instalação e Integração do Redis no Debian 13

Guia completo para instalação do **Redis**, integração com **PHP 8.4** no **Apache2**, comandos de gerenciamento de serviço e práticas recomendadas de segurança e autenticação (senha e ACLs) no Debian 13 (Trixie).

---

## 1. Instalação do Servidor Redis

Atualize os repositórios do sistema e instale o pacote oficial do Redis:

```bash
sudo apt update
sudo apt install -y redis-server
```

Ative o serviço para iniciar automaticamente com o sistema e inicialize-o imediatamente:

```bash
sudo systemctl enable --now redis-server
```

*(Opcional)* Teste se o servidor está respondendo via CLI:

```bash
redis-cli ping
```

Se o serviço estiver ativo e funcionando, ele retornará:

```text
PONG
```

---

## 2. Instalação e Integração com PHP 8.4 (Apache2)

Para utilizar o Redis no PHP 8.4 (geralmente instalado via repositório de Ondřej Surý no Debian), instale a extensão correspondente:

```bash
sudo apt install -y php8.4-redis
```

Ative o módulo explicitamente para a versão 8.4 do PHP:

```bash
sudo phpenmod -v 8.4 redis
```

Reinicie o servidor Apache para carregar a nova extensão:

```bash
sudo systemctl restart apache2
```

### Verificação do Módulo PHP
Para confirmar se a extensão foi carregada com sucesso, execute:

```bash
php -m | grep redis
```

Se a palavra `redis` for retornada no terminal, o módulo está pronto para uso.

---

## 3. Gerenciamento do Serviço Redis

No Debian 13, o gerenciamento do serviço é feito preferencialmente através do `systemd`.

| Ação | Comando |
| :--- | :--- |
| **Verificar status** | `sudo systemctl status redis-server` |
| **Iniciar serviço** | `sudo systemctl start redis-server` |
| **Parar serviço** | `sudo systemctl stop redis-server` |
| **Reiniciar serviço** | `sudo systemctl restart redis-server` |
| **Habilitar início no boot** | `sudo systemctl enable redis-server` |
| **Desabilitar início no boot** | `sudo systemctl disable redis-server` |
| **Parada segura via CLI (salva dados)** | `redis-cli shutdown` |

---

## 4. Segurança e Boas Práticas em Servidores Web / VPS

> [!NOTE]
> Adicionar autenticação por senha ou ACLs no Redis é uma excelente prática de segurança, mesmo quando ele escuta apenas localmente (`127.0.0.1`). A própria documentação oficial recomenda a autenticação como camada de defesa em profundidade.

### Por que configurar senha mesmo em acesso local?
- **Defesa em profundidade:** Caso uma vulnerabilidade na aplicação permita ataques do tipo *Server-Side Request Forgery* (SSRF), o invasor não conseguirá emitir comandos diretamente ao Redis local.
- **Isolamento entre serviços:** Em uma VPS compartilhada com múltiplos aplicativos, sites ou contêineres, a senha impede que um serviço comprometido visualize ou apague dados em cache de outro.
- **Proteção contra falhas de configuração:** Se a diretiva `bind` for alterada acidentalmente ou as regras de firewall falharem, o banco não ficará exposto e desprotegido para a rede externa.

### Boas práticas essenciais
- **Senhas longas e complexas:** O Redis processa requisições com extrema rapidez, permitindo ataques de força bruta em massa contra senhas curtas. Gere senhas aleatórias de 32 caracteres ou mais.
- **Prefira ACLs (Redis 6+):** Crie usuários específicos com permissões restritas em vez de conceder acesso total irrestrito através do usuário `default`.
- **Restrinja comandos perigosos:** No `redis.conf`, considere renomear ou desativar comandos administrativos como `FLUSHALL`, `FLUSHDB` e `CONFIG` caso a aplicação não precise deles.
- **Mantenha o vínculo local e o modo protegido:** Certifique-se de que as diretivas `bind 127.0.0.1 ::1` e `protected-mode yes` permaneçam ativas, e bloqueie a porta padrão `6379` no firewall (UFW ou iptables).

---

## 5. Configuração de Senha e Autenticação (Passo a Passo)

### Passo 1: Gerar uma senha forte
Gere uma chave segura e aleatória de 32 bytes em Base64 usando o OpenSSL:

```bash
openssl rand -base64 32
```

Copie a chave gerada para usá-la na configuração.

---

### Passo 2: Editar o arquivo de configuração
Abra o arquivo de configuração do Redis:

```bash
sudo nano /etc/redis/redis.conf
```

Escolha **uma** das opções abaixo:

#### Opção A: Senha única para o usuário padrão (Mais simples)
Procure pela linha `# requirepass foobared` (use `Ctrl + W` no nano). Remova o `#` do início e substitua `foobared` pela sua senha:

```text
requirepass SUA_SENHA_FORTE_AQUI
```

#### Opção B: Usuário dedicado via ACL (Recomendado no Redis 6+)
Crie um usuário exclusivo para a sua aplicação com permissões específicas. Adicione ao final do arquivo:

```text
user nome_do_usuario on >SUA_SENHA_FORTE_AQUI ~* &* +@all
```

> [!TIP]
> No formato ACL do Redis:
> - `~*` concede acesso a todas as chaves.
> - `&*` concede acesso a todos os canais Pub/Sub.
> - `+@all` libera todos os comandos.
> - Para desativar o usuário padrão sem senha, você pode adicionar também: `user default off`.

Salve e feche o arquivo (`Ctrl + O`, `Enter`, depois `Ctrl + X`).

---

### Passo 3: Reiniciar o serviço Redis
Aplique as alterações reiniciando o serviço:

```bash
sudo systemctl restart redis-server
```

---

### Passo 4: Validar a autenticação
Conecte-se ao cliente de terminal do Redis:

```bash
redis-cli
```

Ao tentar executar um comando sem autenticação:

```text
127.0.0.1:6379> PING
(error) NOAUTH Authentication required.
```

Agora realize a autenticação de acordo com a opção configurada:

- **Se utilizou a Opção A (senha única):**
  ```text
  127.0.0.1:6379> AUTH SUA_SENHA_FORTE_AQUI
  OK
  ```

- **Se utilizou a Opção B (usuário e senha via ACL):**
  ```text
  127.0.0.1:6379> AUTH nome_do_usuario SUA_SENHA_FORTE_AQUI
  OK
  ```

Após receber `OK`, a autenticação está ativa e funcionando. Lembre-se de atualizar as credenciais (ou a string de conexão DSN) na configuração da sua aplicação PHP.
