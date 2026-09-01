O cache das páginas do Twig é compilado e salvo no servidor em storage/cache/twig_slim/. Como esses arquivos compilados são gerados pelo usuário do servidor web (www-data), o atalho Ctrl + F5 do navegador limpa apenas o cache do seu browser, enquanto o PHP continuava entregando os templates antigos já compilados no disco.

🛠️ Como Limpar o Cache do Twig Sempre que Editar um Template
Para facilitar seu desenvolvimento, criamos um utilitário que expurga com segurança todo o cache compilado do Twig pelo próprio servidor web.

Você pode acioná-lo de duas formas simples:

Opção A: Via Terminal (Linux)
bash
curl -s http://127.0.0.1/clean_cache.php
Opção B: Via Navegador
Basta acessar o endereço local em seu navegador:

http://localhost/clean_cache.php
Todas as suas alterações nos arquivos .twig (como a atualização do título para "AgSonhos e Construções") já estão 100% atualizadas e visíveis no painel!