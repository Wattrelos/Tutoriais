# Incus - troubleshooting 002

### Por que a integração havia falhado antes?

Durante a análise na máquina, identifiquei três detalhes cruciais:

1. **O binário do FreeRDP no Debian 13 chama-se `xfreerdp3`:**
   * O pacote `freerdp3-x11` instala `/usr/bin/xfreerdp3` (sem criar o link `xfreerdp` por padrão). Quando o script anterior rodava `xfreerdp`, o terminal retornava *"comando não encontrado"* e fechava a sessão em 0,1 segundo, voltando para o SDDM.
2. **Confirmação do Certificado SSL do XRDP (`/cert:ignore`):**
   * O FreeRDP 3 aguarda uma confirmação interativa de confiança do certificado autoassinado. Sem a flag `/cert:ignore`, ele ficava travado em segundo plano esperando alguém digitar `[Y/N]` num terminal inexistente.
3. **As contas `joao` e `maria` não existiam no Debian físico:**
   * O SDDM não permitia login porque não encontrava esses usuários no `/etc/passwd` do host.

---

### O que já foi configurado e aplicado na sua máquina (`192.168.1.7`)

1. **Instalação das dependências:** `freerdp3-x11`, `zenity` e `netcat-openbsd` estão instalados.
2. **Symlink universal:** Criado `/usr/bin/xfreerdp -> /usr/bin/xfreerdp3`.
3. **Usuários `joao` e `maria` criados no Host:**
   * **Senha padrão:** `aluno123`
   * Ambos adicionados ao grupo `incus-admin` (testei e confirmei que ambos já conseguem gerenciar os containers sem sudo).
4. **Script Orquestrador Híbrido (`/usr/local/bin/iniciar-sessao-sddm.sh`):**
   * Se logar como **`joao`** $\rightarrow$ abre direto em tela cheia no container `aluno-joao`.
   * Se logar como **`maria`** $\rightarrow$ abre direto em tela cheia no container `aluno-maria`.
   * Se logar como **`aluno`** $\rightarrow$ abre o menu gráfico do **Zenity** para escolher entre os alunos disponíveis!
5. **Integração com o SDDM:** Sessão registrada em `/usr/share/xsessions/incus-aluno.desktop` e configurada em `/etc/sddm.conf.d/10-lab-session.conf`. O serviço `sddm` já foi reiniciado.

---

### Como testar agora na máquina física:

Vá até o monitor da máquina real e teste:

* **Teste 1 (Login Nominal do João):**
  * **Usuário:** `joao`
  * **Senha:** `aluno123`
  * *Resultado:* Em menos de 2 segundos, a tela se tornará a área de trabalho XFCE do container `aluno-joao`. Ao clicar em "Sair / Encerrar Sessão", o container desliga e a máquina volta para o SDDM.

* **Teste 2 (Login Nominal da Maria):**
  * **Usuário:** `maria`
  * **Senha:** `aluno123`
  * *Resultado:* Entra direto no container `aluno-maria`.

* **Teste 3 (Menu Seletor via conta genérica):**
  * **Usuário:** `aluno`
  * **Senha:** `aluno123`
  * *Resultado:* Aparece a janela do Zenity com a lista de alunos para você clicar.

O tutorial completo com o script de instalação de 1 clique está salvo e documentado em: [`Incus-automatizando-login-sddm.md`](/tutoriais/containers/Incus-automatizando-login-sddm.md).