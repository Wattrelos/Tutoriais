# Parar serviços conflitantes com o Docker

Para remover os serviços Apache2, MariaDB, Redis e Tomcat da inicialização automática do Debian 13, você deve usar o utilitário systemctl.
Execute o comando abaixo no terminal como root (ou utilizando sudo) para desativar a inicialização automática e parar os serviços imediatamente:

sudo systemctl disable --now apache2 mariadb redis-server tomcat10

## O que este comando faz?

* disable: Remove os links simbólicos do systemd, impedindo que os serviços iniciem junto com o sistema operacional.
* --now: Interrompe (para) os serviços imediatamente, evitando que você precise rodar um comando stop separado.

------------------------------
## Como verificar se funcionou?
Para garantir que os serviços foram desativados com sucesso, você pode verificar o status de inicialização de cada um deles com o comando:

systemctl is-enabled apache2 mariadb redis-server tomcat10

O retorno esperado para todos deve ser disabled.
## Notas sobre os nomes dos serviços no Debian:

* Redis: O pacote padrão do Debian registra o serviço como redis-server.
* Tomcat: Dependendo da versão exata instalada no Debian 13, o nome do serviço pode ser tomcat10. Se o comando acima retornar um erro de serviço não encontrado, verifique o nome exato usando systemctl list-unit-files | grep tomcat.
