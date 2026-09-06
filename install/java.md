# Java
## Para configurar o Java no Google Antigravity IDE instalado no Debian 13, você precisa garantir que o Kit de Desenvolvimento Java (JDK) esteja instalado no sistema operacional e, em seguida, apontar o caminho correto no IDE. [1, 2] 
Siga os passos abaixo para realizar a configuração completa:
## 1. Instalar o JDK no Debian 13
O Debian 13 traz o OpenJDK em seus repositórios oficiais. Abra o terminal e execute os seguintes comandos: 

```bash
# Atualize a lista de pacotes
sudo apt update && sudo apt upgrade -y
# Instale a versão LTS recomendada do JDK (ex: OpenJDK 21 ou superior)
sudo apt install openjdk-21-jdk -y
```

Para verificar se o Java foi instalado corretamente e identificar o caminho do executável, utilize:

```bash
java --version
```

## 2. Localizar o caminho de instalação do Java
Geralmente, o Debian instala o Java no diretório /usr/lib/jvm/. Você precisará desse caminho para o ambiente de desenvolvimento. Você pode listar os caminhos disponíveis com:

```bash
sudo update-alternatives --config java
```

Copie o caminho exibido na tela (ex: /usr/lib/jvm/java-21-openjdk-amd64).
## 3. Configurar no Antigravity IDE
O Antigravity IDE consome as variáveis de ambiente locais do sistema ou permite a definição manual de SDKs: 

   1. Abra o Antigravity IDE.
   2. Acesse o menu de configurações do projeto (Project Structure ou Settings).
   3. Vá até a seção SDKs ou Java Compiler.
   4. Clique em Add SDK (ou sinal de +) e selecione JDK.
   5. Cole ou navegue até o caminho do Java que você localizou no passo anterior (/usr/lib/jvm/java-21-openjdk-amd64).
   6. Salve as alterações e reinicie o projeto.

## 4. Definir a variável JAVA_HOME (Opcional)
Se o IDE não reconhecer o Java automaticamente, configure a variável global adicionando-a ao seu arquivo de perfil (ex: ~/.bashrc ou ~/.zshrc):

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

Depois, atualize o terminal com source ~/.bashrc e abra o IDE diretamente pelo terminal para que ele herde as variáveis de ambiente.

