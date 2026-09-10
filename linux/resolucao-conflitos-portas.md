# Resolução de conflitos de portas

## Conflito entre Apache2 e Nginx onde as contas dos estudantes não são compartimentalizadas, ou seja, tudo é compartilhado e não há containers envolvidos.

Como configurar para que o Apache2 ou o Nginx iniciem apenas sob demanda, evitando o conflito de ambos tentarem usar a porta 80 ao mesmo tempo.

Como dois serviços não podem escutar a mesma porta simultaneamente, a melhor abordagem no Linux (Debian 13) é desativar a inicialização automática deles com o sistema e permitir que os usuários controlem os serviços através do sudo (com privilégios limitados) ou utilizando Systemd User Services (se configurados para rodar em portas altas, o que não é o caso da porta 80 que exige root) ou, a forma mais limpa e padrão: ajustar o arquivo Sudoers.
Aqui estão as melhores estratégias para resolver isso:
## 1. Desativar a inicialização automática de ambos os servidores
Primeiro, garanta que nenhum dos dois servidores inicie sozinho quando o Debian ligar:

* sudo systemctl disable apache2
* sudo systemctl disable nginx
* sudo systemctl stop apache2 nginx

------------------------------
## 2. Permitir que os estudantes iniciem/parem o serviço (Via Sudoers)
Como a porta 80 é uma porta privilegiada (menor que 1024), apenas o usuário root ou processos com privilégios podem iniciar serviços nela. Para permitir que o "Estudante A" gerencie o Nginx e o "Estudante B" gerencie o Apache2 sem que eles tenham acesso total de root, edite o arquivo sudoers.

   1. Execute o comando: sudo visudo
   2. Adicione as seguintes regras no final do arquivo (substitua estudante_nginx e estudante_apache pelos nomes reais das contas Linux):

# Permitir que o usuário do Nginx inicie e pare apenas o Nginx
estudante_nginx ALL=(ALL) NOPASSWD: /usr/bin/systemctl start nginx, /usr/bin/systemctl stop nginx, /usr/bin/systemctl restart nginx

# Permitir que o usuário do Apache inicie e pare apenas o Apache2
estudante_apache ALL=(ALL) NOPASSWD: /usr/bin/systemctl start apache2, /usr/bin/systemctl stop apache2, /usr/bin/systemctl restart apache2

Como os estudantes vão usar:

* Quando o estudante do Nginx for trabalhar, ele digita no terminal dele:
sudo systemctl start nginx
(E quando terminar: sudo systemctl stop nginx)
* Quando o estudante do Apache for trabalhar, ele faz o mesmo para o apache2:
sudo systemctl start apache2

------------------------------
## 3. Alternativa automatizada: Ativação por Socket do Systemd (Apenas se não usarem ao mesmo tempo)
Se o objetivo é que o serviço suba automaticamente assim que o usuário abrir o navegador e tentar acessar o site, o Systemd possui um recurso chamado Socket Activation. No entanto, como ambos disputam a mesma porta 80, o Systemd não saberia para qual servidor (Apache ou Nginx) redirecionar a requisição de forma nativa sem um intermediário (como um proxy reverso). Portanto, a solução via Sudoers (item 2) é a mais segura e funcional para o seu cenário.
------------------------------
## 💡 Dica extra de convivência: Script de "Troca de Turno"
Para evitar que um estudante esqueça o servidor ligado e bloqueie a porta para o outro, você pode criar um script simples para cada um que desliga o concorrente antes de ligar o próprio.
Exemplo de script para o estudante_nginx (com as devidas permissões no sudoers):

#!/bin/bash
sudo systemctl stop apache2 2>/dev/null
sudo systemctl start nginx
echo "Nginx ativo na porta 80!"

Deseja ajuda para configurar o script de inicialização rápida para os estudantes ou prefere seguir com a configuração manual do Sudoers?

