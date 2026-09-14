## RBAC granular - Role-based access control (Controle de Acesso Baseado em Função)

> O RBAC granular é uma evolução do modelo tradicional de RBAC que divide as permissões em níveis muito específicos, permitindo controlar não apenas o que um usuário pode acessar, mas quais ações exatas ele pode executar em recursos específicos ou campos de dados.

## O que significa "granular"?

No RBAC tradicional, as funções são amplas (ex: papel de "Editor" ou "Gerente"). No RBAC granular, essas permissões são detalhadas ao extremo:

* Escopo restrito: Em vez de dar acesso a todos os relatórios, a função permite ver apenas os relatórios da própria filial ou departamento.
* Ações específicas: Separação rigorosa entre ler, criar, editar, deletar, aprovar ou publicar um único tipo de registro.
* Nível de campo/objeto: Permissão para visualizar o cadastro de um cliente, mas sem acesso a dados sensíveis como CPF ou cartão de crédito.

## Diferença entre RBAC Tradicional e Granular

| Característica | RBAC Tradicional | RBAC Granular |
|---|---|---|
| Nível de Permissão | Amplo (ex: gerenciar usuários) | Específico (ex: redefinir senha de usuários comuns, mas não de administradores) |
| Complexidade | Baixa a média | Alta |
| Controle de Recursos | Acesso a telas ou módulos inteiros | Acesso a linhas, colunas ou ações pontuais de uma API |

## Vantagens no Desenvolvimento de Software

* Princípio do Menor Privilégio: Garante que cada usuário (ou sistema) tenha apenas os privilégios estritamente necessários para realizar sua tarefa.
* Segurança aprimorada: Reduz o impacto de vazamentos de contas ou de ações mal-intencionadas.
* Flexibilidade para regras de negócio complexas: Essencial em sistemas corporativos (B2B), de saúde ou financeiros, onde diferentes funcionários da mesma categoria possuem alçadas de aprovação distintas.

