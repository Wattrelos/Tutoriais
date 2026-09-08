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