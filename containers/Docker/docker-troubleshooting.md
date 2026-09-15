# Problemas em sincronizar o Docker local com o Docker Desktop

_No Linux, o **painel gráfico (GUI) do Docker Desktop é fixo para a sua própria máquina virtual** (`desktop-linux`). Ele não possui uma opção na interface para gerenciar o daemon nativo do Linux (`default`)._

Para ter o Docker Desktop gerenciando este ambiente, existem **dois caminhos** dependendo do seu objetivo:

---

### Opção 1: Rodar o projeto pelo Docker Desktop (para aparecer no Dashboard visual)

O motivo de ter dado aquele erro antes foi que **o Docker Desktop no Linux só consegue acessar arquivos dentro de `/home`** (limitação do compartilhador de arquivos *VirtioFS* da máquina virtual). Como os arquivos estavam em `/var/www/html`, a VM não enxergava o `nginx.conf`.

Para usar o Docker Desktop:

1. **Pare os contêineres atuais no Docker nativo** (para liberar as portas `80`, `3306`, etc.):
   ```bash
   docker compose down
   ```

2. **Copie ou mova o projeto para dentro da sua pasta de usuário (`/home`)**:
   ```bash
   cp -r /var/www/html/Docker /home/wattrelos/Docker
   ```

3. **Alterne o contexto de volta para o Docker Desktop**:
   ```bash
   docker context use desktop-linux
   ```

4. **Suba o ambiente a partir da sua `/home`**:
   ```bash
   cd /home/wattrelos/Docker
   docker compose up -d
   ```

> Agora o Docker Desktop conseguirá mapear os arquivos perfeitamente e os contêineres aparecerão na janela do aplicativo Docker Desktop.

---

### Opção 2: Manter em `/var/www/html` e usar uma Interface Visual (Recomendado no Linux)

Rodar no Docker Engine nativo (onde ele está rodando agora) é a melhor prática no Linux por:
* Ter **desempenho nativo** (sem a sobrecarga de memória e CPU da máquina virtual QEMU).
* Ter acesso direto a qualquer pasta do sistema (`/var/www`, `/mnt`, etc.).

Se você quer uma interface visual bonita para acompanhar esses contêineres no Docker nativo:

1. **Portainer (Já está instalado e rodando na sua máquina):**
   * Você já tem o Portainer ativo. Basta acessar no navegador:
     👉 **[https://localhost:9443](https://localhost:9443)** *(aceite o aviso de certificado autoassinado)*
   * Ele exibe gráficos de uso, logs em tempo real, botões de restart/stop e terminal web para todos os contêineres nativos.

2. **Extensão Docker na própria IDE:**
   * Pelo painel lateral da IDE, a extensão do Docker conecta direto no socket nativo `/var/run/docker.sock`, permitindo iniciar, parar e ver logs com um clique.