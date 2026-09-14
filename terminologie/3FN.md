# 3FN - Terceira Forma Normal

> A 3FN ou Terceira Forma Normal é uma regra de modelagem de banco de dados relacionais que serve para eliminar a redundância de dados e evitar inconsistências ao estruturar tabelas.
Uma tabela está na 3FN se, e somente se, ela atende a duas condições estritas:

   1. Já está na 2FN (Segunda Forma Normal).
   2. Não possui dependências transitivas.

Em termos simples: todos os campos que não são chaves devem depender exclusivamente da chave primária, e não de outros campos comuns da tabela. O renomado cientista da computação Bill Kent resumiu isso na famosa frase: "Cada campo deve depender da chave, de toda a chave, e de nada mais além da chave".
------------------------------
## O Problema: Identificando a dependência transitiva
Imagine uma tabela de Funcionarios que não está na 3FN:

| ID_Funcionario (Chave) | Nome | ID_Departamento | Nome_Departamento |
|---|---|---|---|
| 1 | Ana Silva | D10 | Tecnologia |
| 2 | Carlos Souza | D10 | Tecnologia |
| 3 | Bruno Costa | D20 | Recursos Humanos |


* Por que isso é um problema? ID_Funcionario define o ID_Departamento. Porém, o Nome_Departamento depende diretamente do ID_Departamento, e não do funcionário. Se o departamento D10 mudar de nome para "TI", você terá que atualizar múltiplas linhas, correndo o risco de esquecer alguma e gerar dados inconsistentes.

## A Solução: Aplicando a 3FN
Para normalizar e atingir a 3FN, dividimos a tabela em duas, movendo o campo transitivo para um lugar próprio:
Tabela 1: Funcionarios

| ID_Funcionario (Chave) | Nome | ID_Departamento (Chave Estrangeira) |
|---|---|---|
| 1 | Ana Silva | D10 |
| 2 | Carlos Souza | D10 |
| 3 | Bruno Costa | D20 |

Tabela 2: Departamentos

| ID_Departamento (Chave) | Nome_Departamento |
|---|---|
| D10 | Tecnologia |
| D20 | Recursos Humanos |

------------------------------
## Vantagens da 3FN

* Economia de espaço: O nome "Tecnologia" é escrito apenas uma vez.
* Integridade dos dados: Se o nome do departamento mudar, você altera em apenas um único registro.
* Inserção simplificada: Você pode cadastrar um novo departamento mesmo que ele ainda não tenha nenhum funcionário contratado.

