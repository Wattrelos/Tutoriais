# Incus-troubleshooting-005

## Questão: Necessito criar contas para cada aluno?


### Resumo
1. **Dentro da VM (`aluno-joao`):** **NÃO precisa.**
2. **No SDDM do Host (máquina real):** **Depende do método de login** que você preferir adotar no laboratório (Login Nominal vs Menu Interativo).

---

### 1. Preciso criar a conta do `joao` dentro da VM? ❌ NÃO!

**Você não precisa mexer em nada dentro da VM após cloná-la.**

* **Por que?** O isolamento entre os alunos **não é feito por contas de usuário**, mas sim pela **Máquina Virtual inteira**. 
* A VM `aluno-joao` é um computador virtual exclusivo do João (com seu próprio disco Btrfs, memória e sistema de arquivos isolados).
* Como a imagem base (`modelo-desktop`) já foi congelada com o usuário padrão `aluno` e com o autologin do SDDM configurado para ele, ao clonar para `aluno-joao`, a VM já ligará diretamente na área de trabalho KDE pronta para o João usar.
* O João terá privilégios de `sudo` dentro da sua VM sem qualquer risco de afetar a máquina real ou a VM de outros colegas.

---

### 2. Preciso criar a conta no SDDM da máquina física (Debian 13 real)?

O script orquestrador do tutorial ([`/usr/local/bin/iniciar-ambiente-aluno.sh`](file:///var/www/html/tutoriais/containers/Incus/incus-desktop-grafico-alunos.md#L274-L323)) foi projetado de forma inteligente para permitir **duas estratégias de uso**:

---

#### Abordagem A: Conta Única Genérica com Menu Seletor (⭐ Mais Prática e Recomendada)
> **NÃO precisa criar contas para cada aluno no Host!**

Você cria no Debian físico apenas **uma única conta** genérica chamada `aluno`:
```bash
# No Debian físico (Host):
sudo useradd -m -s /bin/bash aluno
echo "aluno:aluno123" | sudo chpasswd
sudo usermod -aG incus-admin aluno
```

* **Como funciona na prática:**
  1. Qualquer estudante senta na máquina física e loga com o usuário `aluno` (senha `aluno123`).
  2. O script detecta que o usuário logado é `aluno` e abre automaticamente uma janela gráfica (**Zenity**) com a lista de alunos disponíveis (`joao`, `maria`, etc., obtida via `incus list`).
  3. O estudante seleciona seu nome com o mouse e clica em **OK**.
  4. A VM dele (`aluno-joao`) é ligada e abre em tela cheia no `virt-viewer`.
* **Vantagem:** Se entrar um aluno novo na turma, você só precisa clonar a VM (`incus copy modelo-desktop aluno-pedro`) e o nome dele já aparecerá na lista instantaneamente, sem precisar cadastrar usuários no Debian real.

---

#### Abordagem B: Login Nominal (Cada aluno tem seu login no monitor físico)
> **SIM, precisa criar a conta simbólica no Host.**

Se você quiser que o João digite `joao` no teclado e a Maria digite `maria` no monitor físico:
```bash
# No Debian físico (Host), crie a conta nominal do João:
sudo useradd -m -s /bin/bash joao
echo "joao:aluno123" | sudo chpasswd
sudo usermod -aG incus-admin joao
```

* **Como funciona na prática:**
  1. O João faz login no SDDM físico como `joao`.
  2. O script detecta que `$USER = "joao"` e busca diretamente a VM `aluno-joao` (`VM="aluno-${USER}"`).
  3. A VM abre direto em tela cheia sem passar por nenhum menu.
* **Importante:** Essa conta no host é apenas "simbólica" (para autenticação no SDDM e permissão do grupo `incus-admin`). O aluno não usará a pasta pessoal do host nem terá poderes de administrador na máquina real.

---

### Resumo Comparativo

| Onde? | Precisa criar `joao`? | Motivo |
| :--- | :---: | :--- |
| **Dentro da VM** | ❌ **Não** | O desktop já abre no usuário padrão `aluno` com autologin. A VM inteira já é exclusiva do João. |
| **No Host (Modo Menu Zenity)** | ❌ **Não** | Usa apenas o usuário genérico `aluno`. Uma janelinha permite ao estudante escolher sua VM na hora. |
| **No Host (Modo Login Nominal)** | ✔️ **Sim** | Necessário para o SDDM do host autenticar o usuário antes de disparar o script da VM. |