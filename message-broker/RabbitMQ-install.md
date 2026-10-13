No Debian 13 Trixie, você pode instalar o RabbitMQ utilizando duas abordagens: nativamente por meio dos repositórios oficiais, ou usando o Docker, que é a forma mais recomendada para ambientes de desenvolvimento moderno, pois evita conflitos de dependências do Erlang (a linguagem na qual o RabbitMQ é escrito).
Abaixo estão os dois métodos para você escolher.

Método 1: Instalação via Docker (Recomendado para Desenvolvimento)

O uso do Docker evita a necessidade de configurar repositórios externos e chaves GPG complexas no Debian 13. Além disso, ele já traz o painel web de gerenciamento pronto para uso. [1] (https://www.youtube.com/watch?v=VWjszh6h21o&t=62)
1. Instale o Docker no Debian:bash
sudo apt update
sudo apt install docker.io -y
Use o código com cuidado.
2. Inicie o container do RabbitMQ:
O comando abaixo baixa a imagem oficial do RabbitMQ com o plugin de gerenciamento integrado (-management) e o executa em segundo plano:bash
sudo docker run -d --name meu-rabbitmq -p 5672:5672 -p 1572:15672 rabbitmq:3-management
Use o código com cuidado.

	• A porta 5672 é usada pelo PHP para enviar e receber mensagens.
	• A porta 15672 é usada para acessar a interface visual no seu navegador. [1] (https://www.magenteiro.com/blog/magento-2/magento-2-rabbitmq/), [2] (https://www.youtube.com/watch?v=uUO2uGaI8Gw)

Método 2: Instalação Nativa via Repositório Oficial do RabbitMQ

Se você prefere rodar o RabbitMQ diretamente no sistema operacional sem o Docker, utilize os repositórios atualizados disponibilizados pela própria equipe do RabbitMQ: [1] (https://translate.google.com/translate?u=https://www.rabbitmq.com/docs/install-debian&hl=pt&sl=en&tl=pt&client=sge)
1. Instale as dependências iniciais:bash
sudo apt update
sudo apt install curl gnupg apt-transport-https -y
Use o código com cuidado.
2. Adicione a chave pública de assinatura da equipe do RabbitMQ:bash
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null
Use o código com cuidado.
3. Adicione os repositórios do Erlang e do RabbitMQ para o Trixie:
Crie o arquivo de lista de fontes:bash
sudo nano /etc/apt/sources.list.d/rabbitmq.list
Use o código com cuidado.
Cole as duas linhas abaixo dentro do arquivo, salve e feche (Ctrl+O, Enter, Ctrl+X):text
deb [signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://rabbitmq.com trixie main
deb [signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://rabbitmq.com trixie main
Use o código com cuidado.
4. Instale o servidor do RabbitMQ:bash
sudo apt update
sudo apt install rabbitmq-server -y
Use o código com cuidado.
5. Inicie e habilite o serviço para iniciar junto com o sistema:bash
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
Use o código com cuidado.

Ativando o Painel de Controle Web (Apenas para o Método Nativo)

Se você escolheu o método nativo, a interface visual vem desativada por padrão. Ative-a com o comando: [1] (https://www.youtube.com/watch?v=Vo7VqWR8Wio&t=8)
bash
sudo rabbitmq-plugins enable rabbitmq_management
Use o código com cuidado.
Reinicie o serviço para aplicar:
bash
sudo systemctl restart rabbitmq-server
Use o código com cuidado.

Acessando o Painel do RabbitMQ

Independentemente do método escolhido, abra o seu navegador e acesse:
👉 http://localhost:15672
• Usuário padrão: guest
• Senha padrão: guest [1] (https://www.magenteiro.com/blog/magento-2/magento-2-rabbitmq/)

Como conectar o PHP ao RabbitMQ?

Para que os seus scripts em PHP possam enviar dados para as filas do RabbitMQ, você precisará instalar a extensão AMQP ou utilizar uma biblioteca nativa. A recomendação padrão do ecossistema modernizado do PHP é utilizar o pacote composer php-amqplib/php-amqplib, que funciona perfeitamente sem exigir a compilação de extensões C complexas no seu Debian. [1] (https://dev.to/rafahs/rabbitmq-primeiros-passos-repost-de-2018-3bp)
Se você quiser, posso demonstrar um exemplo prático de código em PHP criando uma fila e enviando uma mensagem de teste para o seu novo servidor RabbitMQ. Qual o foco do seu projeto?
