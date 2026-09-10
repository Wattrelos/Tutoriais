Compreendido. Como você utiliza múltiplos arquivos de configuração combinados para subir o ambiente, você precisa passar as mesmas flags -f na mesma ordem para que o Docker Compose reconheça a estrutura completa dos seus serviços.
Aqui estão os comandos exatos que você deve usar agora:
## 1. Reiniciar TODOS os containers do seu ambiente
Para reiniciar todos os serviços mapeados nesses dois arquivos de uma vez só:

docker compose -f docker-compose.yml -f docker-compose.postgres.yml restart

## 2. Reiniciar apenas um container específico (ex: o Postgres)
Se você quiser reiniciar apenas o banco de dados sem mexer na aplicação ou no Redis/RabbitMQ, adicione o nome do serviço ao final do comando.
Baseado no seu docker ps, o serviço do banco provavelmente se chama postgres ou tgos_postgresql dentro do arquivo YAML:

docker compose -f docker-compose.yml -f docker-compose.postgres.yml restart <nome_do_servico_no_yaml>

------------------------------
## 💡 Dica prática para o dia a dia
Se achar esse comando muito longo para digitar sempre que precisar reiniciar, você pode usar o comando direto do Docker (que não depende dos arquivos YAML, apenas do nome final do container):

# Reinicia diretamente o container do banco pelo nome dele
docker restart tgos_postgresql
# Reinicia diretamente a sua aplicação Java
docker restart tgos_app

Se o comando com os arquivos -f der algum erro de serviço não encontrado, você poderia me dizer qual é o nome do serviço do banco de dados que está escrito dentro do seu arquivo docker-compose.postgres.yml?

