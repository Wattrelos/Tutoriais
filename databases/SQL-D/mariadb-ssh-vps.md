# Conectando ao MariaDB da VPS na porta 3306 usando phpMyAdmin local

> Nesse cenário, Debian nos dois lados, aplicação em Rust na VPS (sem stack PHP/web server lá) e acesso SSH protegido por passphrase, o túnel SSH é a solução ideal para usar o phpMyAdmin local.

## Passo 1: Abrir o túnel SSH no Debian local

No terminal do seu PC local, execute:
```bash
ssh -N -L 3307:127.0.0.1:3306 usuario@IP_DA_VPS
```

* -N: Não executa comandos remotos (apenas encaminha a porta).
* -L 3307:127.0.0.1:3306: Mapeia a porta local 3307 para a porta 3306 do MySQL/MariaDB dentro da VPS.
* Ao executar, o terminal pedirá a passphrase da sua chave SSH e ficará "travado" mantendo o túnel ativo. Para fechar, basta pressionar Ctrl + C.
(Dica opcional: se quiser rodar em segundo plano depois de digitar a passphrase, você pode usar o ssh-agent ou adicionar -f após autenticar via chave carregada no agente).

## Passo 2: Configurar o phpMyAdmin no Debian local

Edite o arquivo de configuração do phpMyAdmin no seu PC (geralmente em /etc/phpmyadmin/config.inc.php):
```bash
sudo nano /etc/phpmyadmin/config.inc.php
```
Adicione um novo servidor no final do arquivo:
```php
$i++;
$cfg['Servers'][$i]['verbose'] = 'VPS Rust (via SSH)';
$cfg['Servers'][$i]['host'] = '127.0.0.1';
$cfg['Servers'][$i]['port'] = '3307';
$cfg['Servers'][$i]['connect_type'] = 'tcp';
$cfg['Servers'][$i]['auth_type'] = 'cookie';
```

> Por que 127.0.0.1 e não localhost? No Linux, localhost faz o PHP tentar conectar via socket Unix local (/var/run/mysqld/mysqld.sock), ignorando a porta 3307. Usar 127.0.0.1 força a conexão TCP pelo túnel.

## Passo 3: Acessar

1. Com o túnel SSH aberto no terminal, acesse http://localhost/phpmyadmin no navegador.
2. No menu suspenso Servidor da tela de login, selecione VPS Rust (via SSH).
3. Entre com o usuário e senha do banco de dados configurados na VPS.

