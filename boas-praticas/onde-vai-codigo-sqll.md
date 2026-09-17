# Onde vai o Código SQL
O código SQL deve ficar exclusivamente na camada de acesso a dados (Infraestrutura) e nunca deve contaminar a lógica de negócio (Domínio) ou a camada de aplicação. Além do Query Builder, o SQL tem locais muito bem definidos para existir e locais onde sua presença é considerada um forte antipadrão.
Abaixo está a divisão clara de onde o SQL deve e não deve estar:
## 🟢 Onde o SQL DEVE ou PODE ficar

* Data Mapper: É o local mais adequado para o SQL bruto. O Data Mapper isola completamente a entidade de domínio do banco de dados, sendo responsável por mapear as linhas do banco para objetos e vice-versa.
* Repository (Implementação): Na abordagem clássica (sem ORM pesado), a implementação concreta do repositório é onde as queries residem. Ela expõe uma interface limpa para o domínio (ex: findById) mas executa o SQL internamente.
* DAO (Data Access Object): Semelhante ao Data Mapper, foca estritamente em operações de tabela. Se você usa DAOs, o SQL fica dentro deles.
* Procedures / Visões (Banco de Dados): Embora fiquem fora da aplicação, lógicas complexas de busca ou relatórios pesados podem residir diretamente no banco de dados.

## 🔴 Onde o SQL NUNCA deve ficar

* Unit of Work (UoW): O UoW gerencia transações e o rastreamento de mudanças (o quando salvar). Ele não sabe como interagir com o banco e apenas coordena os Repositories ou Mappers para que eles executem o trabalho.
* Entidades de Domínio: Modelos de negócio (como a classe Cliente ou Pedido) devem ser puros e ignorantes sobre persistência. Eles nunca devem conter strings SQL ou chamadas diretas de banco.
* Services (Aplicação/Domínio): Casos de uso e regras de negócio coordenam o fluxo, mas não devem conhecer a estrutura de tabelas ou dialetos SQL.
* Controllers / Handlers (Apresentação): A camada que recebe as requisições HTTP (ou CLI) jamais deve manipular queries, sob o risco de expor o sistema a vulnerabilidades como SQL Injection e quebrar o encapsulamento.

------------------------------
## Resumo das Camadas e Padrões

| Padrão / Componente | Aceita SQL? | Motivo |
|---|---|---|
| Data Mapper | 🟢 Sim | Sua única função é traduzir dados estruturados em objetos. |
| Repository (Concreto) | 🟢 Sim | Abstrai a persistência; se não usar ORM, usará SQL internamente. |
| DAO | 🟢 Sim | Mapeia diretamente as operações de uma tabela específica. |
| Unit of Work | 🔴 Não | Apenas gerencia o escopo da transação e ordem de execução. |
| Domain Entities | 🔴 Não | Devem conter apenas regras de negócio e lógica interna. |
| Services / Controllers | 🔴 Não | Não devem depender de detalhes de infraestrutura ou bancos específicos. |

