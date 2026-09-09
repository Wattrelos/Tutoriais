# Gestão de contas de usuário

## Como criar uma conta de usuário com privilégios de sudo no Debian 13

Para criar uma nova conta de usuário com privilégios de sudo no Debian 13, você deve criar o usuário e adicioná-lo ao grupo sudo. Esse processo exige acesso à conta de root ou a outro usuário que já possua permissões administrativas.
Aqui está o passo a passo sequencial para realizar a configuração:

   1. Acesse o terminal como root
   Digite o comando abaixo e insira a senha do administrador:
   ```bash
   su -
   ```
   2. Crie o novo usuário
   Substitua nome_do_usuario pelo nome desejado:
   ```bash
   adduser nome_do_usuario
   ```
   
   O sistema solicitará que você defina uma senha e preencha informações opcionais.
   3. Adicione o usuário ao grupo sudo
   Execute o comando para dar permissões administrativas ao novo perfil:
   ```bash
   usermod -aG sudo nome_do_usuario
   ```
   
   4. Instale o pacote sudo (se necessário)
   Caso o comando sudo não esteja instalado no seu Debian 13, instale-o com:
   ```bash
   apt update && apt install sudo -y
   ```
   
   5. Teste o acesso do novo usuário
   Troque para a conta criada para verificar se o sudo está funcionando:
   ```bash
   su - nome_do_usuario
   ```

   ```bash
   sudo whoami
   ```
   
   Se o comando retornar "root", a configuração foi concluída com sucesso.

## Excluir conta e remover diretório home

#### Parar conta em execução antes de excluir (opcional)

```bash
# Encontrar PID do usuário
ps -u nome_do_usuario
```
#### Encerra processos do usuário

```bash
# Encerra processos do usuário
kill -9 PID
```


```bash
# Excluir o usuário e remover o diretório home
userdel -r nome_do_usuario
```

#### Outro comando para remover usuário

```bash
# Força a remoção do usuário e do diretório home
deluser --remove-home nome_do_usuario
```

```bash
# Remove o usuário do grupo sudo
usermod -G sudo nome_do_usuario
```

## Desabilitar senha do usuário
```bash
# Desabilitar senha do usuário
passwd -d nome_do_usuario
```

## Habilitar senha do usuário

```bash
# Habilitar senha do usuário
passwd nome_do_usuario
```

