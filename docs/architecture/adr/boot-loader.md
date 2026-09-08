# ADR 001: Escolha do Carregador de Inicialização (Bootloader) para o Host Incus

## Status

Aceito

## Contexto

Estamos projetando a arquitetura de um novo host/servidor de infraestrutura que executará o Incus para orquestração de contêineres de sistema e máquinas virtuais.
Durante o planejamento do particionamento e armazenamento do host, foram definidas as seguintes premissas técnicas:

   1. O sistema operacional base (Host OS) será instalado em um sistema de arquivos tradicional ext4.
   2. O Incus utilizará uma partição ou disco dedicado formatado em Btrfs, cujo ciclo de vida, subvolumes, snapshots e cotas serão gerenciados exclusivamente pelo próprio daemon do Incus (incusd).
   3. O hardware de destino possui suporte nativo e obrigatório a UEFI.

Diante desse cenário, precisamos definir qual carregador de inicialização adotar como padrão para o host, avaliando principalmente os impactos em resiliência, velocidade de recuperação e complexidade de manutenção.
## Alternativas Consideradas## 1. GNU GRUB (Grand Unified Bootloader)

* Prós: Amplamente adotado, maduro e com suporte a ferramentas legadas.
* Contras: Arquitetura complexa baseada em scripts de geração dinâmica de menu (grub-mkconfig / update-grub). Adiciona uma camada de complexidade desnecessária para leitura de partições simples em hardware moderno UEFI. Maior superfície de erro humano ou automação durante atualizações de Kernel.

## 2. systemd-boot (antigo Gummiboot)

* Prós: Extremamente leve, rápido e integrado de forma nativa ao ecossistema systemd (que o Incus já utiliza fortemente para cgroups v2 e fatias de recursos). Segue a Boot Loader Specification, onde cada entrada de kernel é um arquivo de configuração de texto estático e simples.
* Contras: Funciona exclusivamente em ambientes UEFI e possui suporte limitado a esquemas exóticos de criptografia antes do carregamento do kernel (não aplicável ao nosso caso).

## Decisão
Decidimos adotar o systemd-boot como o carregador de inicialização oficial para o host Incus.
## Justificativa da Escolha:

* Casamento de Arquitetura: Como o sistema operacional base utilizará ext4, o systemd-boot precisa apenas ler a partição EFI (FAT32) e apontar para a raiz. Ele não precisa inicializar ou entender o pool Btrfs, pois o Incus assumirá essa responsabilidade após o boot do espaço de usuário (userspace).
* Manutenibilidade e Confiabilidade: Em servidores de produção, a simplicidade dos arquivos .conf estáticos do systemd-boot reduz drasticamente o risco de falhas catastróficas de inicialização após atualizações de Kernel, algo mais comum no GRUB devido à sua esteira complexa de scripts.
* Velocidade de Recuperação (MTTR): O systemd-boot interage de forma direta com o firmware UEFI, minimizando o tempo de inicialização do host e acelerando o retorno dos serviços/contêineres em caso de reinicializações programadas ou falhas de energia.

## Consequências## Positivas:

* Ambiente mais enxuto: Menos pacotes e scripts legados rodando no host.
* Automação simplificada: Provisionadores de infraestrutura (como Ansible ou scripts de instalação) podem injetar ou modificar entradas de boot manipulando arquivos de texto simples em /boot/loader/entries/.
* Isolamento de Armazenamento: O pool Btrfs do Incus fica 100% isolado da lógica de boot do sistema operacional, evitando conflitos de leitura de subvolumes pelo carregador de inicialização.

## Negativas / Requisitos de Implementação:

* Tamanho da Partição EFI (ESP): Exige atenção no particionamento inicial. Devemos alocar no mínimo 512 MB (recomendado 1 GB) para a partição ESP montada em /boot ou /efi, garantindo espaço suficiente para armazenar múltiplos Kernels e imagens initramfs.
* Restrição de Hardware: Remove a possibilidade de migrar este disco ou imagem de sistema para hipervisores ou hardwares legados que operem apenas em modo BIOS/Legacy.

------------------------------
Podemos usar este documento como base para criar os scripts de automação de particionamento ou partir para a sintaxe exata do comando incus admin init para o pool Btrfs. O que prefere fazer agora?

