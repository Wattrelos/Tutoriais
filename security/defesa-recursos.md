# DROP, REJECT e TARPIT: Qual o melhor para economizar recursos? 

>O DROP é geralmente a melhor opção para economizar recursos do servidor, mas ele consome um tipo específico de recurso do atacante. O DROP descarta o pacote silenciosamente sem enviar nenhuma resposta de volta, poupando banda e CPU do seu sistema.
A diferença entre DROP, REJECT e TARPIT ajuda a entender onde cada lado gasta mais energia:

## 1. Ação DROP (Silêncio total)

* O que faz no servidor: Descarta o pacote imediatamente na camada de rede e não gasta banda de upload nem processamento para montar uma resposta.
* O que gasta do atacante: Consome tempo e threads (processos) de conexão, pois o cliente do atacante fica aguardando o tempo limite (timeout) estourar.
* Ponto fraco: Ferramentas automatizadas modernas enviam milhares de requisições assíncronas por segundo, então o timeout quase não pesa na CPU ou banda do atacante.

## 2. Ação REJECT (Recusa imediata)

* O que faz no servidor: Envia um pacote de erro (ICMP Port Unreachable ou TCP RST) avisando que a porta está fechada.
* O que gasta do atacante: Quase nada, pois a conexão falha na mesma hora e o atacante passa imediatamente para o próximo alvo.
* Desvantagem: Gasta banda de upload do seu servidor e entrega a informação de que existe um firewall ativo no IP.

## 3. Ação TARPIT (Armadilha de lentidão)

* O que faz no servidor: Aceita a conexão TCP, mas reduz a janela de comunicação a zero, prendendo a conexão aberta indefinidamente.
* O que gasta do atacante: Esgota memória RAM e portas locais do atacante, pois o sistema operacional dele precisa manter o estado daquela conexão aberto sem conseguir fechar nem enviar dados.
* Desvantagem: Exige módulos extras no firewall (como xt_TARPIT no iptables) e consome um pouco mais de memória da tabela de conexões do seu próprio servidor.

## Resumo prático

* Use DROP como padrão no Fail2ban e no firewall para proteger o servidor com o menor custo possível de CPU e banda.
* Use TARPIT apenas se você quiser ativamente travar bots ou scanners simples usando módulos avançados de kernel.
Se você quiser aplicar essa regra no seu ambiente, posso mostrar como configurar o DROP (ou o TARPIT) no seu sistema. Qual firewall você está usando com o Fail2ban no momento (UFW, iptables, firewalld ou nftables)?
