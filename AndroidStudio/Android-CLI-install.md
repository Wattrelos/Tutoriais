# Instalação da CLI do Android no Linux

A instalação da CLI do Android no Linux pode ser feita rapidamente pelo terminal de duas formas: apenas para o seu usuário (local) ou para todos os usuários do sistema (global).
Escolha o método oficial fornecido pelos [Android Developers](https://developer.android.com/tools/agents/android-cli?hl=pt-br) que melhor atende às suas necessidades:
## Opção 1: Instalação Local (Recomendado)
Instala a CLI apenas na sua conta de usuário sem exigir permissões de administrador (sudo).  

curl -fsSL https://dl.google.com/android/cli/latest/linux_x86_64/install.sh | bash

## Opção 2: Instalação Global
Instala a ferramenta para todos os usuários da máquina. Este método exige privilégios de sudo. 

curl -fsSL https://dl.google.com/android/cli/latest/linux_x86_64/install_root.sh | bash

------------------------------
## Próximos Passos e Pós-Instalação

   1. Verifique se a instalação funcionou:
   Abra um novo terminal e execute o comando abaixo para confirmar que o executável está no seu PATH:
   
   command -v android
   
   Se o comando retornar o caminho do binário, a instalação foi bem-sucedida. 
   2. Inicialize a CLI para Agentes (Opcional):
   Caso vá utilizar assistentes ou automações integradas à CLI, use o comando de inicialização para configurar as ferramentas necessárias:  
   
   android init
   
   3. Mantenha a ferramenta atualizada:
   A CLI do Android recebe melhorias constantes. Atualize-a regularmente executando: 
   
   android update
   
