# 🛡️ Guia Definitivo: HTTPS Local com mkcert, Nginx e Rust (Axum) no Debian 13

> **Contexto & Motivação:**  
> Navegadores modernos baseados em Chromium e Gecko aplicam restrições rigorosas sob as especificações da W3C (*Potentially Trustworthy Origins*) e HTML Standard §7.4.2. Cabeçalhos de segurança avançados como `Cross-Origin-Opener-Policy: same-origin` (COOP), `Cross-Origin-Embedder-Policy` (COEP) e APIs de plataforma (Web Crypto, Service Workers, WebAuthn, Geolocation e Cookies com `SameSite=None; Secure`) são sumariamente bloqueados ou desativados quando servidos em HTTP puro fora do loopback estrito.
>
> Este guia ensina a configurar um ambiente HTTPS local confiável com certificado válido no **Debian 13 (Trixie)**, terminando TLS no **Nginx 1.26** e repassando o tráfego limpo para o backend **Rust (Axum)** na porta `3001`.

---

## 🧭 Visão Geral da Arquitetura Local

```
  ┌────────────────────────────────────────────────────────┐
  │ Navegador Web / Playwright (Chromium / Firefox / Edge)  │
  └───────────────────────────┬────────────────────────────┘
                              │
                    HTTPS :443 (TLS Confiável)
                    SNI: skanstull.local / 192.168.1.11
                              │
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ Nginx 1.26 (Reverse Proxy & TLS Termination)           │
  │ • Certificado emitido por CA local (mkcert)            │
  │ • Redireciona HTTP :80 -> HTTPS :443                   │
  │ • Injeta cabeçalho: X-Forwarded-Proto https            │
  └───────────────────────────┬────────────────────────────┘
                              │
                    HTTP :3001 (Loopback Puro)
                              │
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ Backend Alpha Engine (Rust / Axum 0.8 / Tokio)          │
  │ • Lê X-Forwarded-Proto e ativa COOP / HSTS / Cookies    │
  │ • Sem overhead criptográfico de TLS na camada da API   │
  └────────────────────────────────────────────────────────┘
```

---

## 📦 Passo 1: Instalação do `mkcert` e Dependências no Debian 13

No Debian 13 (Trixie), o `mkcert` está disponível diretamente nos repositórios oficiais via APT, dispensando instalações manuais de binários ou Homebrew.

Abra o terminal e execute:

```bash
# 1. Atualizar índice de pacotes
sudo apt update

# 2. Instalar o mkcert e as ferramentas de gerenciamento de certificados NSS
sudo apt install -y mkcert libnss3-tools
```

> **Por que `libnss3-tools`?**  
> O Chrome, Chromium e Firefox utilizam o banco de dados NSS para gerenciar CAs confiáveis. O `libnss3-tools` permite que o `mkcert` instale a raiz diretamente no navegador sem que você precise importar certificados manualmente em cada browser.

---

## 🏛️ Passo 2: Criar e Instalar a Autoridade Certificadora (CA) Local

Execute o comando abaixo **uma única vez** no seu sistema:

```bash
mkcert -install
```

**Saída esperada:**
```text
Created a new local CA 
Succeeded in installing the local CA in the system trust store.
Succeeded in installing the local CA in the Chromium and Firefox trust stores.
```

Para verificar onde os arquivos da sua Autoridade Certificadora local foram salvos:
```bash
mkcert -CAROOT
# Padrão: /home/seu_usuario/.local/share/mkcert
```

---

## 🔑 Passo 3: Gerar os Certificados TLS com Suporte a SAN

> ⚠️ **Atenção (Armadilha do `sudo mkcert`):**  
> **NUNCA** execute `sudo mkcert ...` para gerar certificados se você rodou `mkcert -install` com o seu usuário comum. O `sudo` executará como `root`, gerando uma segunda CA desconhecida pelo seu usuário e navegadores, causando o erro:  
> `curl: (60) SSL certificate problem: unable to get local issuer certificate`.
>
> **Fluxo correto:** Gere os certificados com o seu usuário comum e depois mova para `/etc/ssl/localcerts/` com `sudo`.

```bash
# 1. Gerar o certificado na sua pasta atual com seu usuário comum:
mkcert \
  -cert-file skanstull.local.pem \
  -key-file skanstull.local-key.pem \
  localhost 127.0.0.1 ::1 \
  skanstull.local "*.skanstull.local" \
  alpha.local beta.local \
  192.168.1.11

# 2. Mover para a pasta segura do Nginx e ajustar permissões:
sudo mkdir -p /etc/ssl/localcerts
sudo mv skanstull.local.pem /etc/ssl/localcerts/skanstull.local.pem
sudo mv skanstull.local-key.pem /etc/ssl/localcerts/skanstull.local-key.pem
sudo chown -R root:www-data /etc/ssl/localcerts
sudo chmod 750 /etc/ssl/localcerts
sudo chmod 640 /etc/ssl/localcerts/*.pem

# 3. Registrar a CA local no repositório de certificados do sistema operacional (Debian):
# Isso garante que o cURL e o OpenSSL reconheçam o certificado globalmente:
sudo mkdir -p /usr/local/share/ca-certificates
sudo cp "$(mkcert -CAROOT)/rootCA.pem" /usr/local/share/ca-certificates/mkcert_local_ca.crt
sudo update-ca-certificates
```

> 💡 **Dica de Ouro (Script Automático):**  
> Você também pode executar diretamente o script utilitário pronto no repositório:
> ```bash
> sudo ./scripts/dev/setup-local-tls.sh
> ```

> 💡 **Dica de Ouro (IP da LAN):**  
> Ao incluir o IP da sua máquina na rede local (`192.168.1.11`), você poderá abrir o site no navegador do celular ou de outro computador conectado ao mesmo Wi-Fi sem alertas de segurança (basta exportar e instalar a CA do `mkcert -CAROOT` no aparelho móvel).

---

## 🌐 Passo 4: Mapeamento de Host Local (`/etc/hosts`)

Garanta que os domínios locais apontem para a sua máquina. Edite `/etc/hosts`:

```text
127.0.0.1   localhost skanstull.local www.skanstull.local alpha.local beta.local
::1         localhost ip6-localhost ip6-loopback
```

---

## 🚀 Passo 5: Configuração do Nginx 1.26 com Terminação TLS

Atualize a configuração do site em `/etc/nginx/sites-available/skanstull`:

```nginx
# ==============================================================================
# 1. Redirecionamento HTTP (Porta 80) para HTTPS (Porta 443)
# ==============================================================================
server {
    listen 80;
    listen [::]:80;
    server_name skanstull.local www.skanstull.local localhost 192.168.1.11;

    # Força transição segura permanente (HTTP 301)
    return 301 https://$host$request_uri;
}

# ==============================================================================
# 2. Servidor Seguro HTTPS (Porta 443 com HTTP/2)
# ==============================================================================
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name skanstull.local www.skanstull.local localhost 192.168.1.11;

    # Certificados gerados pelo mkcert
    ssl_certificate /etc/ssl/localcerts/skanstull.local.pem;
    ssl_certificate_key /etc/ssl/localcerts/skanstull.local-key.pem;

    # Parâmetros modernos de segurança SSL/TLS
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # Logs de auditoria local
    access_log /var/log/nginx/skanstull_access.log;
    error_log /var/log/nginx/skanstull_error.log;

    # Encaminhamento para o backend Axum (Rust) na porta 3001
    location / {
        proxy_pass http://127.0.0.1:3001;

        # Suporte a WebSockets / Server-Sent Events (SSE)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Cabeçalhos essenciais de proxy reverso
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # ⚠️ CRÍTICO: Informa ao Axum que a comunicação do cliente é HTTPS
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # Timeouts generosos para chamadas assíncronas
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

### Testar e Reiniciar o Nginx:
```bash
# Validar sintaxe da configuração
sudo nginx -t

# Recarregar o serviço Nginx
sudo systemctl reload nginx
```

---

## 🦀 Passo 6: Integração no Backend Rust (Axum)

Graças ao cabeçalho `proxy_set_header X-Forwarded-Proto https;`, o backend em Rust não precisa gerenciar chaves criptográficas nem lidar com a complexidade de certificados X.509. 

No middleware de cabeçalhos de segurança ([`apps/api/src/middlewares/security_headers.rs`](/apps/api/src/middlewares/security_headers.rs)), o Axum inspeciona a requisição:

```rust
// Inspeciona se a conexão externa é segura via TLS proxy
let is_https = headers
    .get("x-forwarded-proto")
    .and_then(|h| h.to_str().ok())
    .map(|proto| proto.eq_ignore_ascii_case("https"))
    .unwrap_or(false);

// Se for HTTPS ou localhost, emite Cross-Origin-Opener-Policy com segurança total
if is_https || is_localhost {
    headers.insert(
        HeaderName::from_static("cross-origin-opener-policy"),
        HeaderValue::from_static("same-origin"),
    );
}
```

### Benefícios desta Arquitetura:
1. **Desempenho Extremo:** A decriptação TLS é delegada ao Nginx (em C nativo com OpenSSL/HTTP2), liberando os threads do runtime Tokio do Rust exclusivamente para a lógica de negócio, regras fiscais e queries no banco.
2. **Zero Alertas no Navegador:** Como a CA raiz do `mkcert` está instalada no sistema operacional, o cadeado verde aparece limpo no Chrome/Edge/Firefox, sem modal de advertência.
3. **Paridade Rigorosa Dev/Prod:** Comportamentos como cookies seguros (`SameSite=None; Secure`), Service Workers e isolamento de janelas funcionam de forma idêntica à nuvem de produção.

---

## 🧪 Passo 7: Validação e Testes Automatizados

### 1. Teste via cURL no Terminal:
```bash
curl -I https://skanstull.local
```
**Resposta esperada (HTTP/2 200 com COOP e HSTS):**
```http
HTTP/2 200 
server: nginx
date: Mon, 19 Oct 2026 13:25:00 GMT
content-type: text/html
strict-transport-security: max-age=31536000; includeSubDomains; preload
cross-origin-opener-policy: same-origin
cross-origin-resource-policy: same-site
```

### 2. Validação no Playwright E2E:
Você pode apontar a `BASE_URL` diretamente para o proxy HTTPS:
```bash
BASE_URL=https://skanstull.local npm run test:e2e:agent
```
Como o `mkcert` instalou a CA na base do Chromium, o Playwright executará os testes com `ignoreHTTPSErrors: false` de maneira completamente transparente e com telemetria 100% verde!

---

## 📚 Tabela Comparativa: OpenSSL Autoassinado vs. mkcert

| Critério | OpenSSL Manual (`openssl req`) | `mkcert` (Recomendado) |
| :--- | :--- | :--- |
| **Aceitação no Navegador** | ❌ Alerta vermelho permanente ("Sua conexão não é segura") | ✅ Confiável nativamente (Cadeado 100% verde) |
| **Configuração de SAN (Subject Alt Name)** | Complexa (exige arquivo `v3.ext` com IPs e DNS) | Automática (basta passar os nomes como argumentos) |
| **Integração com NSS (Firefox/Chrome)** | Manual e propensa a falhas em cada perfil | Automática via comando `mkcert -install` |
| **Suporte a Múltiplos Dispositivos** | Difícil de gerenciar | Simples: basta instalar o arquivo de `mkcert -CAROOT` |
| **Adequação ao Ciclo TACL (Playwright)** | Exige `ignoreHTTPSErrors: true` | Executa sob as regras estritas da W3C |
