# 📊 Instalação, Hardening (Basic Auth) e Exposição Segura do Prometheus na VPS — Skanstull

> **Contexto Skanstull:** Na VPS de produção (Debian 13), o **Prometheus 2.53** escuta exclusivamente em loopback (`127.0.0.1:9090`), raspando métricas da API Axum (`127.0.0.1:3001/metrics` a cada 5s) e do Node Exporter (`127.0.0.1:9100`). Ao expor o painel via **Cloudflare + Nginx** em `https://prometheus.agsonhos.com.br`, é obrigatório protegê-lo com **Basic Auth** (`web.yml`).

---

## 1. Gerar o Hash Bcrypt da Senha

O Prometheus exige que a senha esteja criptografada em **bcrypt** (`$2y$` ou `$2b$`). No terminal da VPS, gere o hash usando `htpasswd` ou Python:

* **Com `htpasswd` (`apache2-utils`):**
  ```bash
  sudo apt install -y apache2-utils
  htpasswd -nB -C 12 admin
  ```
* **Com Python (`python3-bcrypt`):**
  ```bash
  python3 -c 'import bcrypt; print(bcrypt.hashpw(b"SUA_SENHA_FORTE", bcrypt.gensalt(rounds=12)).decode())'
  ```

Copie apenas o hash gerado (ex.: `$2y$12$...`).

---

## 2. Criar e Proteger o Arquivo `/etc/prometheus/web.yml`

Crie `/etc/prometheus/web.yml` com o usuário e o hash bcrypt:

```yaml
basic_auth_users:
  admin: $2y$12$cole_aqui_o_hash_gerado_no_passo_anterior
```

Ajuste as permissões para que apenas o usuário do serviço `prometheus` leia o arquivo e valide a sintaxe com `promtool`:

```bash
sudo chown prometheus:prometheus /etc/prometheus/web.yml
sudo chmod 640 /etc/prometheus/web.yml
sudo promtool check web-config /etc/prometheus/web.yml
```

---

## 3. Atualizar o Auto-Scrape no `prometheus.yml`

No repositório (`monitoring/prometheus/prometheus.yml`), o Prometheus também raspa a si mesmo (`job_name: 'prometheus'` em `127.0.0.1:9090`). Com Basic Auth ativo, adicione as credenciais nesse job em `/etc/prometheus/prometheus.yml` para evitar `401 Unauthorized` na tela **Targets**:

```yaml
  - job_name: 'prometheus'
    scrape_interval: 10s
    basic_auth:
      username: 'admin'
      password: 'SUA_SENHA_FORTE'
    static_configs:
      - targets: ['127.0.0.1:9090']
```

> **Dica:** Certifique-se também de que `monitoring/prometheus/alert_rules.yml` esteja copiado para `/etc/prometheus/alert_rules.yml` e valide a configuração completa antes de reiniciar:
> ```bash
> sudo promtool check config /etc/prometheus/prometheus.yml
> ```

---

## 4. Iniciar o Prometheus com `--web.config.file` e Bind em Localhost

### Opção A: Systemd (Instalação Nativa na VPS)

Edite `/etc/systemd/system/prometheus.service` garantindo `--web.listen-address=127.0.0.1:9090` (para nunca expor a porta `9090` em `0.0.0.0`) e `--web.config.file=/etc/prometheus/web.yml`:

```ini
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.listen-address=127.0.0.1:9090 \
  --web.config.file=/etc/prometheus/web.yml \
  --web.external-url=https://prometheus.agsonhos.com.br
```

Recarregue e reinicie o serviço:

```bash
sudo systemctl daemon-reload
sudo systemctl restart prometheus
sudo systemctl status prometheus --no-pager
```

### Opção B: Docker / Docker Compose

Mapeie `/etc/prometheus/web.yml` para dentro do contêiner e passe a flag no `command`:

```yaml
command:
  - '--config.file=/etc/prometheus/prometheus.yml'
  - '--web.config.file=/etc/prometheus/web.yml'
  - '--web.external-url=https://prometheus.agsonhos.com.br'
```

---

## 5. Atualizar o Data Source do Grafana

Na pilha do **Skanstull**, o Grafana é provisionado via arquivo (`monitoring/grafana/provisioning/datasources/prometheus.yaml` com `editable: false`). Como ele consulta `http://127.0.0.1:9090`, atualize o provisionamento no servidor (`/etc/grafana/provisioning/datasources/prometheus.yaml`) incluindo `basicAuth`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://127.0.0.1:9090
    isDefault: true
    editable: false
    basicAuth: true
    basicAuthUser: admin
    secureJsonData:
      basicAuthPassword: SUA_SENHA_FORTE
    jsonData:
      timeInterval: 5s
      httpMethod: POST
```

Reinicie o Grafana para aplicar:

```bash
sudo systemctl restart grafana-server
```

---

## 6. Proxy Reverso no Nginx (`https://prometheus.agsonhos.com.br`)

Mantenha a porta `9090` **fechada no UFW** e crie o virtual host em `/etc/nginx/sites-available/prometheus.agsonhos.com.br` apontando para `http://127.0.0.1:9090` (com o certificado SSL Origin CA da Cloudflare ou Let's Encrypt):

```nginx
server {
    listen 443 ssl http2;
    server_name prometheus.agsonhos.com.br;

    ssl_certificate     /etc/ssl/cloudflare/agsonhos.com.br.pem;
    ssl_certificate_key /etc/ssl/cloudflare/agsonhos.com.br.key;

    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Ative o site e recarregue o Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/prometheus.agsonhos.com.br /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 7. Vincular no `.env` do Backoffice Skanstull

No arquivo `.env` da aplicação Rust em produção (`/opt/skanstull/.env`), configure as URLs públicas de observabilidade para que os botões do Painel Administrativo apontem corretamente:

```env
GRAFANA_URL=https://grafana.agsonhos.com.br
PROMETHEUS_URL=https://prometheus.agsonhos.com.br
```

---

## 8. Validação Rápida

```bash
# Sem senha -> deve retornar HTTP 401 Unauthorized
curl -I http://127.0.0.1:9090/targets

# Com senha -> deve retornar HTTP 200 OK
curl -I -u admin:SUA_SENHA_FORTE http://127.0.0.1:9090/targets
```

> Referência oficial: [Prometheus Basic Authentication Guide](https://prometheus.io/docs/guides/basic-auth/)
