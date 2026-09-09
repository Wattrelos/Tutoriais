# ADR 002: Arquitetura de Ambientes Virtuais com Interface Gráfica e Isolamento por Aluno via Incus

## Status

Aceito

## Contexto

Estamos arquitetando um ambiente de laboratório para ensino de tecnologia e desenvolvimento de software. Os computadores físicos (estações de trabalho com 16 GB de RAM e armazenamento gerenciado em Btrfs) são compartilhados por dezenas de turmas em horários e turnos distintos (manhã, tarde, noite).

Durante as aulas práticas, os estudantes necessitam:
1. Executar e instalar servidores web, bancos de dados e ferramentas com acesso administrativo (`root`), como Apache2, Nginx, MariaDB e Docker.
2. Utilizar um ambiente gráfico amigável (Desktop) com navegador web, terminal gráfico, gerenciador de arquivos e editores de código (como VS Code / Mousepad).
3. Garantir que as configurações, portas e serviços de um estudante não interfiram ou quebrem os ambientes dos colegas que usam a mesma máquina física em outros horários.

---

## O Problema Técnico do Modelo Tradicional

* **Conflito de Portas e Serviços:** No modelo de contas multiusuário tradicional do Linux (ex: `/home/aluno1`, `/home/aluno2`), a pilha de rede (*Network Stack*) é compartilhada. Se o Aluno A do turno da manhã instala o Apache2 na porta 80 (`0.0.0.0:80`), o Aluno B do turno da noite é incapaz de subir o Nginx na mesma porta, recebendo o erro `Address already in use`.
* **Conflito de Pacotes no Host:** O gerenciador `apt` é global. Modificações em bibliotecas (`/usr/lib`) ou arquivos de configuração (`/etc/`) impactam todos os usuários da estação.

---

## Alternativas Consideradas

### 1. Contas Locais Convencionais no Host com Permissão Sudo
* **Prós:** Simples de criar inicialmente.
* **Contras:** Sem isolamento de rede (colisão de portas Apache/Nginx); risco crítico de alunos danificarem o sistema operacional hospedeiro com comandos destrutivos; inviável para avaliações práticas isoladas.

### 2. Máquinas Virtuais Convencionais (VirtualBox / VMware)
* **Prós:** Isolamento de rede e sistema operacional.
* **Contras:** Consumo excessivo de memória RAM (2 a 4 GB por VM); inicialização lenta (40 a 60 segundos); arquivos de disco `.vdi` pesados que inviabilizam manter 30 ou 40 alunos por máquina sem esgotar o disco.

### 3. Containers de Sistema Incus com XFCE4 e XRDP (Sessão Gráfica Dedicada)
* **Prós:** 
  * Cada aluno tem seu próprio endereço IP na bridge `incusbr0` e sua própria pilha de rede: Aluno A roda Apache2 na porta 80 e Aluno B roda Nginx na porta 80 sem qualquer conflito.
  * Consumo mínimo de memória: apenas ~250 MB a 350 MB de RAM em idle com XFCE4 e terminal.
  * Inicialização quase instantânea (< 2 segundos).
  * Eficiência de disco massiva via Copy-on-Write no pool Btrfs: todos os containers compartilham a mesma imagem base, consumindo espaço apenas para os arquivos modificados.
  * Segurança total com *Unprivileged Containers* (root seguro mapeado via User Namespaces).
* **Contras:** Requer uma camada leve de cliente RDP (FreeRDP ou Remmina) na máquina host física para projetar o display do container no monitor.

---

## Decisão

Adotamos **Containers de Sistema Incus com interface gráfica XFCE4 e XRDP** como o padrão oficial para os ambientes didáticos individuais de laboratório.

A dinâmica operacional seguirá o modelo de **1 container ativo por vez**:
1. O host Debian físico inicializa uma sessão gráfica mínima.
2. O aluno entra com sua identificação na estação.
3. Um script de automação (`iniciar-ambiente-aluno.sh`) inicia o container individual do aluno (`incus start aluno-$USER`) e projeta a interface XFCE4 em tela cheia via FreeRDP/Remmina.
4. Ao encerrar a aula, o logout desliga o container (`incus stop`), devolvendo 100% dos recursos de hardware para o aluno do turno seguinte.

---

## Justificativa da Escolha

1. **Eliminação Definitiva de Conflitos de Serviços:** Como cada container possui sua própria Network Namespace, cada aluno possui seu próprio `localhost` e porta 80 independente, permitindo testes simultâneos ou alternados de stacks concorrentes (Apache, Nginx, Caddy, etc.).
2. **Densidade e Custo de Hardware:** Uma estação com 16 GB de RAM pode hospedar com tranquilidade mais de 40 containers de alunos cadastrados em disco no Btrfs, mantendo execução com desempenho nativo.
3. **Resiliência e Recuperação Rápida:** Professores e técnicos conseguem restaurar qualquer ambiente corrompido em menos de 2 segundos através de snapshots instantâneos (`incus snapshot restore <aluno> base`).

---

## Consequências

### Positivas:
* Autonomia pedagógica total com privilégios de `root` isolados dentro de cada container.
* Ambiente gráfico completo com navegador, terminal gráfico e suporte a ferramentas acadêmicas.
* Isolamento garantido de dados e projetos contra exclusão acidental por colegas de outros turnos.
* Ausência de sobrecarga de virtualização pesada.

### Negativas / Requisitos de Implementação:
* O host físico deve ter o cliente RDP (`freerdp3-x11` ou `remmina`) instalado.
* Necessidade de padronizar um script de entrada e saída na tela de login para garantir o ciclo de vida do container (`start` no login e `stop` no logout).

---

## Referências e Implementação

* Guia Passo a Passo de Implementação: [Ambientes Gráficos Isolados por Aluno com Incus](../../../containers/incus-desktop-grafico-alunos.md)
* Guia de Instalação e Particionamento do Incus: [Incus - Guia de Instalação](../../../containers/Incus%20-LXD-install.md)