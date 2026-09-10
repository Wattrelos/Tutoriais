# Guia de Solução de Erros de Permissão (Sudoers) da conta padrão "aluno"

## 1. Corrigir o arquivo Sudoers
No Debian, às vezes o grupo %sudo não vem ativado por padrão se o sistema foi instalado sem uma configuração específica. 

   1. Como root, abra o arquivo de configuração com o comando seguro:
```bash   
   visudo
```
   
   2. Procure por esta linha exata no arquivo:
```bash   
   %sudo   ALL=(ALL:ALL) ALL
```
   
   3. Se ela tiver um # na frente (ex: # %sudo ...), apague o # para ativá-la.
   4. Se ela não existir, vá até o final do arquivo e adicione manualmente uma nova linha para o seu usuário:
```bash
   aluno   ALL=(ALL:ALL) ALL
```
   
   5. Para salvar no editor padrão (nano): Pressione Ctrl + O, depois Enter para confirmar, e Ctrl + X para sair.

## 2. Garantir a atualização do grupo (Sem reiniciar a máquina)
Para não precisar reiniciar a VPS inteira, você pode forçar o terminal atual do usuário a carregar os novos grupos imediatamente.
Logue novamente como aluno e execute:
```bash   
su - aluno
```
Ou use este comando para recarregar o grupo na sessão atual:
```bash   
newgrp sudo
```

Depois disso, teste rodar o comando do Incus:
```bash   
sudo incus exec aluno-joao -- bash
```
