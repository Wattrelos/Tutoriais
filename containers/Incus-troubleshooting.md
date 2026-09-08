# Troubleshooting

## Diagnóstico: Por que o erro acontece?

1. **`Program to run not set`:**
   No código-fonte do Konsole (`Session.cpp`), ao iniciar uma nova aba/janela, ele procura qual comando deve rodar:
   * Primeiro ele checa o perfil ativo;
   * Se o comando do perfil estiver vazio, ele busca a variável de ambiente `SHELL` (`getenv("SHELL")`);
   * Se a variável `$SHELL` não existir ou for nula, ele emite o aviso **`Program to run not set`** e **encerra a sessão imediatamente**, fechando a janela.
2. **Por que no `xterm` funciona?**
   O `xterm` possui um mecanismo de fallback: quando `$SHELL` não está exportada, ele consulta `/etc/passwd` diretamente via `getpwuid()`. O Konsole não faz essa consulta direta se o perfil e a variável `$SHELL` estiverem vazios.
3. **Por que a variável `SHELL` sumiu dentro do container?**
   O comando `incus exec` inicia a sessão com um ambiente mínimo e **não exporta a variável `SHELL`** para dentro do processo. Embora o `bash` defina internamente `$SHELL=/bin/bash`, ele **não marca essa variável com o atributo `export`** a menos que ela já existisse no ambiente herdado. Por isso, os subprocessos (como o Konsole chamado a partir do terminal) recebem `SHELL` vazio.
4. **`QLayout: Cannot add a nul widget to QHBoxLayout/`:**
   É apenas um aviso estético inofensivo da biblioteca Qt do KDE; não é ele que causa o fechamento do programa.

---

### O que faltou configurar e como resolver

Você pode corrigir isso de três formas (a Opção 1 ou 2 são as mais recomendadas para o seu ambiente com Incus):

#### Opção 1: Injetar a variável `$SHELL` no script de login do Incus (Recomendado)

No arquivo [/usr/local/bin/entrar-container.sh](file:///var/www/html/tutoriais/containers/Incus%20-LXD-install.md#L258-L267) descrito na seção 7 do tutorial, adicione a flag `--env SHELL=/bin/bash` na chamada do `incus exec`:

```bash
#!/bin/bash
CONTAINER="$USER"

# Garante que o container esteja iniciado
incus start "$CONTAINER" 2>/dev/null

# Transfere a sessão exportando o SHELL explicitamente
exec incus exec "$CONTAINER" --env SHELL=/bin/bash -- /bin/bash --login
```

#### Opção 2: Definir a variável permanentemente no Incus

Você pode instruir o Incus a sempre injetar o `SHELL=/bin/bash` nas instâncias:

* **Para o container atual:**
  ```bash
  incus config set aluno01 environment.SHELL=/bin/bash
  ```
* **Para a imagem modelo (`modelo-lab`):**
  ```bash
  incus config set modelo-lab environment.SHELL=/bin/bash
  ```
  *(Assim, todas as novas cópias criadas para alunos já herdarão essa variável configurada)*.

#### Opção 3: Exportar o `SHELL` dentro do Container

Se você estiver dentro do container (via terminal ou xterm):

1. **Teste imediato no xterm:**
   ```bash
   export SHELL=/bin/bash
   konsole
   ```
   *(ou chamando diretamente `konsole -e /bin/bash`)*.

2. **Tornar permanente para todas as sessões dentro do container:**
   Adicione ao `/etc/environment` do container:
   ```bash
   echo "SHELL=/bin/bash" | sudo tee -a /etc/environment
   ```
   Ou no `~/.bashrc`:
   ```bash
   echo "export SHELL=/bin/bash" >> ~/.bashrc
   ```

---

### Dica Adicional (Perfil padrão do Konsole)

Caso queira garantir que o Konsole nunca dependa de variáveis de ambiente no container ou no usuário, crie um perfil padrão com o comando explícito:

```bash
mkdir -p ~/.local/share/konsole ~/.config

# Cria o perfil padrão apontando para o bash
cat << 'EOF' > ~/.local/share/konsole/Default.profile
[General]
Name=Default
Parent=FALLBACK/
Command=/bin/bash
EOF

# Define esse perfil como ativo
cat << 'EOF' > ~/.config/konsolerc
[Desktop Entry]
DefaultProfile=Default.profile
EOF
```

---
# Troubleshooting

### Por que agora o `xterm` e o `konsole` abrem e fecham na hora?

O fechamento imediato é causado pela combinação de duas coisas:

1. **O comando `exit` no `~/.bashrc`:**  
   No passo 4 da Seção 7 do tutorial, foi configurado:
   ```bash
   echo "sudo /usr/local/bin/entrar-container.sh" | sudo tee -a /home/aluno01/.bashrc
   echo "exit" | sudo tee -a /home/aluno01/.bashrc
   ```
   Toda vez que você abre o terminal, o bash lê o `.bashrc`. Se o script `entrar-container.sh` encontrar **qualquer erro e for abortado**, o bash passa imediatamente para a linha seguinte (`exit`), **fechando a janela em milissegundos sem deixar você ler o erro**.

2. **O script `/usr/local/bin/entrar-container.sh` está falhando:**  
   Existem dois motivos principais para ele estar falhando:

   * **Causa principal (O bug do `$USER` no `sudo`):**  
     No script original consta:
     ```bash
     CONTAINER="$USER"
     ```
     Quando o script é executado via `sudo`, o Linux redefine `$USER` para **`root`**!  
     O script tenta executar `incus exec root ...`. Como **não existe container chamado `root`** (o container chama-se `aluno01`), o Incus retorna `Error: Instance not found` e encerra. Em seguida, o `exit` do `.bashrc` fecha a janela.

   * **O container ainda não existe ou não foi iniciado:**  
     Se o container com o nome do usuário (`aluno01`) não foi clonado a partir da imagem modelo (`incus copy modelo-lab aluno01`), o Incus também falha.

---

### Como corrigir

#### 1. Corrigir o script `/usr/local/bin/entrar-container.sh` no Host

Use a variável `${SUDO_USER:-$USER}` (que captura o usuário real que invocou o `sudo`) e já adicione a flag `--env SHELL=/bin/bash`:

```bash
sudo tee /usr/local/bin/entrar-container.sh << 'EOF'
#!/bin/bash
# Captura o nome do aluno que executou o sudo (ex: aluno01)
CONTAINER="${SUDO_USER:-$USER}"

# Garante que o container esteja iniciado
incus start "$CONTAINER" 2>/dev/null

# Transfere a sessão exportando o SHELL para o container
exec incus exec "$CONTAINER" --env SHELL=/bin/bash -- /bin/bash --login
EOF

sudo chmod 755 /usr/local/bin/entrar-container.sh
```

#### 2. Confirmar se o container do aluno existe

Certifique-se de que o container com o mesmo nome do usuário do host foi criado:

```bash
# Como usuário administrador no host:
incus list
# Se aluno01 não existir, crie-o a partir do modelo:
incus copy modelo-lab aluno01
incus start aluno01
```

#### 3. Dica para ver o erro caso a janela volte a fechar

Para conseguir ver a mensagem de erro sem a janela fechar na sua cara, comente temporariamente o `exit` do `.bashrc` do aluno:

```bash
sudo sed -i 's/^exit/#exit/' /home/aluno01/.bashrc
```

Assim, se o script falhar, a janela do terminal permanecerá aberta exibindo exatamente a mensagem de erro do Incus. Quando tudo estiver funcionando, você pode descomentar o `exit`.

---

> Se desejar, posso atualizar o arquivo [Incus -LXD-install.md](file:///var/www/html/tutoriais/containers/Incus%20-LXD-install.md#L240-L249) para que o script já use `${SUDO_USER:-$USER}` e `--env SHELL=/bin/bash` por padrão no tutorial.