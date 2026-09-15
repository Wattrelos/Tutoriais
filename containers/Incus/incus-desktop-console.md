
## 6. Criação do Container Modelo (Golden Image)

Em vez de baixar e configurar um container do zero para cada aluno, criamos uma imagem modelo com as ferramentas de aula pré-instaladas.

```bash
# 1. Cria e inicializa um container base Debian
incus launch images:debian/13 modelo-lab

# 2. Acessa o shell do container modelo
incus exec modelo-lab -- bash
```

Dentro do container `modelo-lab`, instale as ferramentas necessárias para as aulas:

```bash
# Atualiza pacotes internos
apt update && apt upgrade -y

# Instala ferramentas essenciais de desenvolvimento
apt install -y build-essential gdb git curl wget nano vim sudo python3 python3-pip python3-venv htop net-tools

# Cria uma senha para o root do container (opcional)
echo "root:aluno123" | chpasswd

# Sai do container
exit
```

Com o container configurado, pare-o e crie um **snapshot de referência**:

```bash
# Para o container
incus stop modelo-lab

# Cria um snapshot base que servirá de ponto de restauração instantâneo
incus snapshot create modelo-lab base
```

---

## 7. Métodos de Acesso para os Alunos

Aqui resolvemos o desafio de **conectar o aluno ao seu container com root, sem dar acesso root ao host físico**.

### Método 1: Login Direto no Terminal Local (Ideal para Estações de Laboratório)

Neste modelo, o aluno faz login na máquina física com seu usuário normal, mas ao abrir o terminal, ele entra imediatamente no seu container com privilégios de `root`:

#### 1. Criar o container individual a partir do modelo
```bash
# Clona o modelo instantaneamente (em menos de 1 segundo via Copy-on-Write)
incus copy modelo-lab aluno01
incus start aluno01
```

#### 2. Criar um script de entrada restrita no host
Crie o arquivo `/usr/local/bin/entrar-container.sh`:
```bash
sudo tee /usr/local/bin/entrar-container.sh << 'EOF'
#!/bin/bash
# Captura o usuário real que invocou o sudo (ex: aluno01)
CONTAINER="${SUDO_USER:-$USER}"

# Garante que o container esteja iniciado
incus start "$CONTAINER" 2>/dev/null

# Transfere a sessão exportando o SHELL explicitamente para evitar falhas em emuladores como o Konsole
exec incus exec "$CONTAINER" --env SHELL=/bin/bash -- /bin/bash --login
EOF

sudo chmod 755 /usr/local/bin/entrar-container.sh
```

#### 3. Configurar permissão segura no Sudoers
Adicione uma regra em `/etc/sudoers.d/alunos-incus` para permitir que os alunos executem apenas este script com privilégios administrativos:
```bash
sudo tee /etc/sudoers.d/alunos-incus << 'EOF'
# Permite que qualquer usuário execute o script de entrada sem senha
ALL ALL=(ALL) NOPASSWD: /usr/local/bin/entrar-container.sh
EOF
```

#### 4. Criar a conta do aluno no host com inicialização automática
```bash
# Cria o usuário aluno01 no Debian host
sudo adduser --gecos "" --disabled-login aluno01
echo "aluno01:senhaAluno123" | sudo chpasswd

# Configura o shell ou o .bashrc para chamar o script automaticamente
echo "sudo /usr/local/bin/entrar-container.sh" | sudo tee -a /home/aluno01/.bashrc
echo "exit" | sudo tee -a /home/aluno01/.bashrc
```

> **Resultado:** Quando o `aluno01` faz login no Debian do laboratório, o terminal cai diretamente dentro do seu container com `root`. Ao digitar `exit`, ele é desconectado da máquina física. Ele não tem acesso ao sistema de arquivos do host!

> [!TIP]
> **Dica para Testes e Diagnóstico:** Se o terminal fechar instantaneamente durante os testes iniciais, comente temporariamente a linha `exit` em `/home/aluno01/.bashrc` (`sudo sed -i 's/^exit/#exit/' /home/aluno01/.bashrc`) para visualizar eventuais mensagens de erro de inicialização. Para mais detalhes e soluções de problemas conhecidos com emuladores de terminal (Konsole, xterm), consulte o guia [Incus-troubleshooting.md](Incus-troubleshooting.md).

---

### Método 2: Acesso Remoto via SSH ou Rede Interna

Se os alunos acessam um servidor central de laboratório ou utilizam seus próprios notebooks:

#### 1. Instalar o OpenSSH dentro do container
```bash
incus exec aluno01 -- apt install -y openssh-server
incus exec aluno01 -- systemctl enable --now ssh
incus exec aluno01 -- sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
incus exec aluno01 -- systemctl restart ssh
```

#### 2. Mapear uma porta específica para cada aluno (Proxy Device)
O Incus possui um recurso nativo de proxy reverso de rede:
```bash
# Redireciona a porta 2201 da máquina host para a porta 22 do container aluno01
incus config device add aluno01 ssh-proxy proxy listen=tcp:0.0.0.0:2201 connect=tcp:127.0.0.1:22
```

O aluno se conecta de qualquer lugar da rede da faculdade (ou via VS Code Remote SSH):
```bash
ssh root@<IP_DA_MAQUINA_HOST> -p 2201
```

---

## 8. Controle de Recursos e Quotas (cgroups)

Para evitar que scripts com loops infinitos travem a máquina física compartilhada, configure limites rígidos por container:

```bash
# Limita o uso de CPU a 2 núcleos
incus config set aluno01 limits.cpu=2

# Limita o uso de memória RAM a 2 GiB
incus config set aluno01 limits.memory=2GiB

# Força a interrupção de processos que estourarem a memória (evita swap excessivo)
incus config set aluno01 limits.memory.enforce=hard

# Limita o espaço em disco do container a 15 GiB (em pools Btrfs ou ZFS)
incus config device set aluno01 root size=15GiB
```

---

## 9. Guia Prático de Manutenção para Professores e Técnicos

### Restaurar o Container Quebrado em 2 Segundos
Se o aluno excluir arquivos críticos do sistema ou quebrar dependências:
```bash
incus stop aluno01
incus snapshot restore aluno01 base
incus start aluno01
```

### Script de Automação para Criar Turmas Inteiras
Crie um script `/usr/local/bin/criar-turma.sh`:
```bash
#!/bin/bash
TURMA="so2026"
TOTAL_ALUNOS=30

for i in $(seq -w 1 $TOTAL_ALUNOS); do
    NOME="aluno-${TURMA}-${i}"
    echo "Provisionando container para $NOME..."
    
    # 1. Clona a partir do modelo base
    incus copy modelo-lab "$NOME"
    
    # 2. Aplica limites de recursos
    incus config set "$NOME" limits.cpu=2 limits.memory=2GiB
    
    # 3. Inicia o container
    incus start "$NOME"
    
    # 4. Tira o snapshot inicial para rollback
    incus snapshot create "$NOME" base
done

echo "Todos os $TOTAL_ALUNOS containers foram provisionados com sucesso!"
```

### Limpeza ao Final do Semestre
Para desalocar os recursos de todos os containers de uma turma:
```bash
incus delete --force aluno-so2026-01 aluno-so2026-02 aluno-so2026-03
```
Ou listar e remover em lote:
```bash
incus list -c n --format csv | grep 'aluno-so2026' | xargs -r incus delete --force
```

---

## 10. Conclusão da Proposta

A adoção do **Incus com Debian** transforma o laboratório acadêmico:
1. **Para os Alunos:** Liberdade total com acesso `root`, projetos contínuos sem perdas por congelamento de disco, isolamento contra exclusão acidental de arquivos de terceiros e aprendizado de Linux real em padrão de mercado.
2. **Para a TI:** Garantia inegociável de segurança com containers não-privilegiados, partição dedicada que impede travamento do host por disco cheio, isolamento de recursos por cgroups e sem risco de contaminação da rede física.
3. **Para os Professores:** Agilidade nas aulas, padronização de imagens didáticas e restauração instantânea de qualquer ambiente danificado em segundos.
