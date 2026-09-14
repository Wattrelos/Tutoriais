# POEAA - Patterns of Enterprise Application Architecture

> O termo PoEAA refere-se ao livro clássico "Patterns of Enterprise Application Architecture" (Padrões de Arquitetura de Aplicações Corporativas), escrito por Martin Fowler em 2002.
Este livro é uma das maiores referências no desenvolvimento de software, pois catalogou dezenas de padrões de projeto para resolver os problemas mais comuns enfrentados ao construir sistemas corporativos complexos (como persistência de dados, concorrência, distribuição e lógica de negócios).
Os padrões do PoEAA são amplamente divididos em categorias principais:

## 1. Organização da Lógica de Negócios (Business Logic)
Define como estruturar o código que dita as regras do sistema.

* Transaction Script: Organiza a lógica de negócios em procedimentos únicos, onde cada procedimento lida com uma única solicitação da tela (ideal para sistemas simples).
* Domain Model (Modelo de Domínio): Uma abordagem orientada a objetos onde os dados e o comportamento (regras) estão juntos no mesmo objeto (essencial para regras complexas e base do Domain-Driven Design ou DDD).
* Table Module: Uma única classe gerencia a lógica de negócios para todas as linhas de uma tabela ou view do banco de dados.

## 2. Padrões de Persistência de Dados (Data Source Architectural Patterns)
Focados em como a aplicação se comunica com o banco de dados.

* Table Data Gateway / Row Data Gateway: Objetos que atuam como portas de entrada para uma tabela inteira ou para uma linha específica, isolando o código SQL da lógica de negócios.
* Active Record: Uma abordagem onde o próprio objeto de domínio contém tanto os dados quanto os métodos de persistência (como save() e update()). Muito popular em frameworks como Ruby on Rails e Laravel.
* Data Mapper: Uma camada de mapeamento que separa completamente os objetos de domínio do banco de dados. Os objetos de domínio não sabem como são salvos, o que garante total isolamento. É a base de ORMs modernos como Hibernate e Entity Framework.

## 3. Padrões de Mapeamento Objeto-Relacional (O/R Mapping)
Resolvem o problema de salvar estruturas de objetos (herança, associações) em tabelas relacionais.

* Identity Map: Garante que cada objeto do banco de dados seja carregado apenas uma vez na memória, evitando duplicatas e inconsistências durante uma requisição.
* Unit of Work (Unidade de Trabalho): Mantém uma lista de todos os objetos afetados por uma transação comercial (novos, alterados ou deletados) e coordena a gravação dessas mudanças de uma só vez, otimizando o uso do banco.
* Lazy Load (Carregamento Preguiçoso): Interrompe o carregamento de dados pesados do banco até o momento exato em que eles são necessários no código.

## 4. Padrões de Arquitetura Web (Web Presentation)
Como estruturar a interface e o fluxo de navegação.

* Model-View-Controller (MVC): Divide a aplicação em três componentes para separar a interface (View), os dados/regras (Model) e o controle de fluxo (Controller).
* Page Controller: Um objeto lida com as requisições de uma página ou ação específica do site.
* Front Controller: Um único manipulador central (Handler) recebe todas as requisições do sistema e as distribui para os comandos adequados (padrão padrão de frameworks como Spring MVC).

------------------------------
## Por que o PoEAA ainda importa?
Apesar de ter sido escrito em 2002, os conceitos do PoEAA moldaram quase todos os frameworks modernos de desenvolvimento de software (Node.js, .NET, Java Spring, Django, etc.). Entender esses padrões ajuda o desenvolvedor a escolher a arquitetura certa: por exemplo, saber quando usar um modelo simples como Active Record ou quando a complexidade exige um Data Mapper com Domain Model.