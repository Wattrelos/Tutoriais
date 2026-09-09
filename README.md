# 📚 Base de Conhecimento e Tutoriais

Este repositório reúne tutoriais práticos e de consulta rápida para configuração de ambientes de desenvolvimento, ferramentas, servidores, bancos de dados e projetos no Linux (com foco em **Debian 13 Trixie**).

A base foi estruturada para permitir que um ambiente de desenvolvimento completo e funcional possa ser recriado do zero em poucos minutos.

---

## 🗂️ Índice de Tutoriais

### 🐧 Linux & Sistema Operacional
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Comandos Úteis](linux/comandos-uteis.md) | Comandos práticos de terminal, administração de pacotes e sistema no Debian 13. |
| 📄 [Instalação da Steam](linux/steam.md) | Habilitação de repositórios `non-free`, arquitetura i386 e instalação do cliente Steam no KDE. |

### 🐙 Git, GitHub & Forgejo
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Padronizar Branch Única (main)](git/padronizar-branch-main.md) | Como unificar branches `master` e `main` locais e remotas sem perder histórico. |
| 📄 [Instalação do Forgejo](git/forgejo-instalacao.md) | Guia definitivo para rodar o servidor Git Forgejo de forma nativa com Apache2 e MariaDB. |
| 📄 [Importar Repositórios do GitHub para Forgejo](git/importar-issues-github-para-forgejo.md) | Migração web completa incluindo commits, branches, issues, PRs e labels. |
| 📄 [Exportar Issues do GitHub para Markdown (gh2md)](git/gh2md.md) | Scripts e automações para baixar issues do GitHub em arquivos `.md`. |

### 🐳 Containers, Docker & Incus
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Instalação do Incus (Containers de Sistema)](containers/Incus%20-LXD-install.md) | Proposta e guia completo de implantação do Incus no Debian 13 com pool Btrfs para laboratórios. |
| 📄 [Ambientes Gráficos por Aluno com Incus](containers/incus-desktop-grafico-alunos.md) | Desktops XFCE4 individuais com terminal, sem conflitos de portas (Apache2 vs Nginx) e login automatizado. |
| 📄 [Automação de Login no SDDM com Incus](containers/Incus-automatizando-login-sddm.md) | Integração com SDDM e FreeRDP para login direto no desktop do container e seletor gráfico de alunos. |
| 📄 [Instalação do Docker](containers/docker-instalacao.md) | Configuração do Docker CE através do repositório oficial no Debian 13. |
| 📄 [Baixando Imagens no Docker Desktop](containers/docker-baixando-imagens.md) | Como buscar e efetuar pull de imagens pela interface visual e busca rápida. |

### 🌐 Servidores Web
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Instalação do Apache2](web-servers/apache-instalacao.md) | Configuração do servidor HTTP Apache2 no Debian 13. |
| 📄 [Instalação do NGINX](web-servers/nginx-instalacao.md) | Instalação e ativação do NGINX via repositórios oficiais. |

### 🗄️ Bancos de Dados & Cache
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Instalação do MariaDB](databases/mariadb-instalacao.md) | Setup do SGBD MariaDB e execução do `mysql_secure_installation`. |
| 📄 [MongoDB Community Server](databases/mongodb-server-instalacao.md) | Instalação manual por pacotes `.deb` contornando restrições de chaves GPG no Debian 13. |
| 📄 [MongoDB Compass (GUI)](databases/mongodb-gui-instalacao.md) | Instalação da interface visual do MongoDB e correção de execução via usuário root. |
| 📄 [Instalação e Integração do Redis](databases/redis-instalacao.md) | Instalação do Redis Server e módulo para PHP 8.4 com Apache2. |

### 🛠️ Ferramentas de Desenvolvimento & SDKs
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Antigravity & Antigravity IDE](dev-tools/antigravity-ide.md) | Instalação e configuração do ambiente de desenvolvimento potencializado por IA. |
| 📄 [Android Studio](dev-tools/android-studio.md) | Setup completo, bibliotecas de compatibilidade 32-bits e atalhos no Debian 13. |
| 📄 [Java (JDK)](dev-tools/java.md) | Instalação do OpenJDK e apontamento no Antigravity IDE. |
| 📄 [Composer (PHP)](dev-tools/composer.md) | Instalação segura do gerenciador de dependências PHP. |
| 📄 [Behat & Mink](dev-tools/behat.md) | Configuração do framework BDD compatível com Symfony 7+. |

### 🚀 Projetos: Alpha Engine
| Tutorial | Descrição |
| :--- | :--- |
| 📄 [Instalação e Setup de Tenants](projetos/alpha-engine/instalacao.md) | Especificação técnica de onboarding de tenants SaaS e provisionamento zero-state. |
| 📄 [Matriz de Requisitos](projetos/alpha-engine/requisitos.md) | Matriz completa de requisitos de sistema, extensões PHP e permissões de diretório. |
| 📄 [Configuração de Firewall (UFW)](projetos/alpha-engine/configurar-ufw.md) | Regras recomendadas de portas e segurança de rede. |
| 📄 [Limpeza de Cache Twig](projetos/alpha-engine/limpar-cache-twig.md) | Como expurgar o cache compilado de templates Twig gerados pelo `www-data`. |

---

## 📌 Padrão de Contribuição e Nomenclatura

Ao adicionar novos tutoriais nesta base de conhecimento, siga as seguintes diretrizes:

1. **Localização**: Adicione o arquivo dentro da pasta correspondente ao domínio da tecnologia (`linux/`, `git/`, `containers/`, `web-servers/`, `databases/`, `dev-tools/`, `projetos/<nome>/`).
2. **Nomes de Arquivos**:
   - Utilize sempre formato `kebab-case` minúsculo sem acentos, espaços ou símbolos especiais (ex: `meu-novo-tutorial.md`).
3. **Estrutura do Documento**:
   - **Título H1**: Título claro e objetivo no topo do documento.
   - **Resumo**: Breve parágrafo explicativo sobre o propósito do tutorial.
   - **Passo a Passo**: Seções numeradas com blocos de código com sintaxe destacada (ex: ` ```bash `).
4. **Atualizar o README**:
   - Sempre adicione o novo tutorial à tabela correspondente neste `README.md`.
