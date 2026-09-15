# Incus VM - Recomendações de alocação de recursos

> [!WARNING]
> Não é seguro atribuir toda a memória RAM e CPUs para a máquina virtual. Você deve obrigatoriamente deixar uma margem para o sistema operacional real (hospedeiro).
Se você alocar 100% dos recursos, o teu computador hospedeiro (Debian 13) e a própria máquina virtual irão travar.


## 🚨 O risco real com a Memória RAM
Ao contrário de containers (LXC), as Máquinas Virtuais (VMs) no Incus/QEMU funcionam de forma "rígida". Se você definir limits.memory=16GiB em uma máquina que só tem 16 GB de RAM:

   1. O Incus tentará reservar imediatamente essa memória para a VM.
   2. O Debian 13 (real) ficará sem memória até para as tarefas mais básicas do sistema.
   3. O mecanismo de segurança do Linux (chamado OOM Killer) irá entrar em ação e começará a derrubar processos vitais do teu sistema, ou o computador irá congelar por completo, exigindo um reinício forçado no botão.

## 🧠 O risco com a CPU
Definir limits.cpu=all é um pouco menos perigoso do que a RAM, mas ainda pode causar problemas. Se a VM decidir usar 100% de processamento em todos os núcleos, o sistema real perderá a prioridade. O mouse irá começar a travar, o áudio irá picotar e o computador real parecerá congelado até que a VM termine a tarefa pesada.
------------------------------
## 🛠️ Qual é a margem ideal recomendada?
Para que o teu Debian 13 real funcione perfeitamente enquanto a VM extrai o máximo de desempenho, siga esta tabela de recomendação:

| Se o teu PC real tem: | Aloque para a VM no máximo: | Deixe livre para o PC Real: |
|---|---|---|
| 8 GB RAM | 5 GiB ou 6 GiB | 2 GB a 3 GB |
| 16 GB RAM | 12 GiB ou 13 GiB | 3 GB a 4 GB |
| 32 GB RAM | 26 GiB ou 28 GiB | 4 GB a 6 GB |
| CPUs (Ex: 8 núcleos) | Defina limits.cpu=6 ou 7 | Deixe 1 ou 2 núcleos livres |

💡 Dica de ouro: Em vez de usar limits.cpu=all, configure o número exato de núcleos menos um. Por exemplo, se o teu processador tem 8 núcleos (threads), use limits.cpu=7.

Se você quiser ajustar os valores exatos agora, me diga:

* Quanto você tem de RAM total e quantos núcleos de CPU aparecem no teu computador real? (Você pode ver isso rodando o comando free -h e nproc no terminal do Debian).

# Cenário de referência:

## Processador: Intel Core i7-4770 (que possui 4 núcleos físicos e 8 threads/CPUs lógicas)
## Memória: 16 GB de RAM

### Neste cenário, irá rodar apenas uma VM de cada vez, você pode ser mais agressivo na alocação de recursos, pois a VM ativa poderá aproveitar quase todo o potencial da máquina.
Considerando o processador Intel Core i7-4770 (que possui 4 núcleos físicos e 8 threads/CPUs lógicas) e os 16 GB de RAM, os limites máximos recomendados para extrair o melhor desempenho com segurança são:

* limits.cpu=6 (Deixa 2 threads livres para o Debian real gerenciar o sistema e o QEMU).
* limits.memory=13GiB (Deixa exatamente 3 GB livres para o Debian real não travar).

------------------------------
## Comandos prontos para configurar
Para aplicar esses limites na sua VM atual (modelo-desktop), execute os seguintes comandos no terminal do hospedeiro:
```bash
incus config set modelo-desktop limits.cpu=6
incus config set modelo-desktop limits.memory=13GiB
```

Se preferir criar uma nova VM já com esses limites otimizados, use:
```bash
incus launch images:debian/13 modelo-desktop --vm -c limits.cpu=6 -c limits.memory=13GiB
```

## Por que essas marcas específicas?

   1. Memória (13 GiB): O i7-4770 geralmente possui gráficos integrados (Intel HD Graphics 4600) que podem "roubar" um pouco da memória RAM do sistema (Dynamic Video Memory). Deixar 3 GB livres garante que o Debian 13 real, a interface gráfica do teu monitor principal e o próprio emulador da VM (QEMU) rodem sem nenhuma chance de acionar o travamento por falta de memória.
   2. CPU (6 vCPUs): O i7-4770 é um processador Quad-Core com Hyper-Threading (8 CPUs vistas pelo sistema). Se você der as 8 CPUs para a VM, quando ela exigir 100%, o hospedeiro não terá ciclos de processamento para gerenciar os discos (I/O) e a rede da própria VM, gerando um gargalo. Deixando 6 para a VM, ela voa baixo e o sistema principal continua respondendo instantaneamente.


