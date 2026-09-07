# Ambientes isolados para cada aluno

_Para dar acesso root total a um aluno sem que ele quebre o sistema operacional real da máquina local ou apague as configurações dos colegas, você não pode usar o Docker tradicional (que compartilha o mesmo kernel e restringe o root). Em vez disso, você deve usar Containers de Sistema (LXC) ou Máquinas Virtuais Leves._

As três melhores soluções para dar um Linux "vazio" com root para cada aluno na mesma máquina são:

------------------------------
## 1. Incus / LXD (Containers de Sistema) — A mais leve e eficiente
Diferente do Docker (que isola apenas um aplicativo), o Incus (sucessor do LXD mantido pela comunidade Linux) cria um container que se comporta exatamente como uma máquina virtual vazia, mas rodando direto no kernel do hospedeiro.

* Como funciona para o aluno: O aluno loga na conta dele e digita um comando como incus launch images:debian/13 meu-ambiente. O sistema cria em 2 segundos um Debian 13 totalmente limpo. O aluno digita incus exec meu-ambiente -- bash e entra direto como root em um prompt limpo.
* Isolamento: O aluno pode dar apt install nginx, apt install mariadb-server, parar serviços, alterar o arquivo /etc/hosts interno, e nada disso afetará o computador real ou os outros alunos.
* Persistência: O container fica salvo na área do aluno. O colega do próximo turno não tem acesso a ele.

## 2. Proxmox VE no Laboratório — A mais profissional (Padrão de Mercado)
Se você tiver um servidor centralizado no laboratório, você pode instalar o Proxmox VE (um sistema operacional de virtualização gratuito e de código aberto).

* Como funciona: Cada aluno recebe um usuário e senha para acessar o painel web do Proxmox pelo navegador de qualquer computador.
* O Ambiente: O professor pode criar um "Template" de um Debian 13 totalmente limpo. Com um clique, o aluno clona esse template e ganha o seu próprio servidor virtual (LXC ou VM) com acesso root via console web.
* Vantagens: O aluno experimenta exatamente o ambiente de uma VPS profissional (como AWS, DigitalOcean ou Linode).

## 3. Multipass ou QEMU Local — Máquinas Virtuais Rápidas
Se os computadores do laboratório forem potentes (com boas CPUs e RAM), você pode usar o Multipass (da Canonical) ou scripts baseados em QEMU/KVM.

* Como funciona: No primeiro login do aluno, um script inicia uma mini máquina virtual Debian isolada em segundo plano. O aluno abre o terminal e se conecta a ela via SSH já logado como root. Ele pode destruir e recriar essa máquina virtual quantas vezes quiser através de comandos simples como multipass launch e multipass purge.

------------------------------
## Solução Recomendada
Para o seu cenário de dar um prompt com root isolado na mesma máquina, o Incus (ou LXD) no Debian 13 é a solução técnica perfeita. Ele consome pouquíssima memória RAM (menos de 50MB por ambiente parado) e dá ao aluno a experiência exata de ter acabado de formatar uma máquina e contratado uma VPS limpa.


