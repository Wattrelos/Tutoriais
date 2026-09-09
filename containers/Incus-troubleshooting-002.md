# Troubleshooting 002: Erros com sudo -i e Configuração do Usuário

## 1. Entrar como Root no Container

Ao executar comandos administrativos dentro do container Incus, você já está conectado diretamente como `root` via `incus exec modelo-desktop -- bash`.

Caso esteja em um terminal com seu usuário comum (`wattrelos`), acesse o ambiente root completo com:

```bash
sudo -i
```

O prompt mudará de `wattrelos@ServidorVPS` para `root@ServidorVPS`.

---

## 2. Script de Criação do Usuário no Container

Execute o bloco abaixo dentro do container `modelo-desktop` para provisionar o usuário padrão `aluno`:

```bash
# 1. Cria o usuário 'aluno' com pasta home e shell bash
useradd -m -s /bin/bash aluno

# 2. Define uma senha padrão didática
echo "aluno:aluno123" | chpasswd

# 3. Adiciona o aluno ao grupo sudo
usermod -aG sudo aluno

# 4. Permite que o aluno use sudo sem exigir senha dentro do container
echo "aluno ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-aluno-init
chmod 0440 /etc/sudoers.d/90-aluno-init

# 5. Copia o .xsession para a home do aluno
if [ -f /etc/skel/.xsession ]; then
    cp /etc/skel/.xsession /home/aluno/.xsession
    chown aluno:aluno /home/aluno/.xsession
else
    # Se não existir, cria apontando para o XFCE
    echo "startxfce4" > /home/aluno/.xsession
    chown aluno:aluno /home/aluno/.xsession
fi
```

---

## 3. Sair do Root

Quando terminar a configuração, digite `exit` para retornar ao terminal padrão:

```bash
exit
```
