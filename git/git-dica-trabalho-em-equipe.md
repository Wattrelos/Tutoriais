# Guia Prático: Git e Trabalho em Equipe para Iniciantes

Trabalhar em equipe com Git pode parecer assustador no início, principalmente quando surgem os temidos **conflitos de versão**. No entanto, seguindo um fluxo organizado e entendendo o porquê de cada etapa, a colaboração se torna natural e segura.

---

## 🎯 Regra de Ouro: O Fluxo Diário de Trabalho

Muitos iniciantes acreditam que devem baixar as alterações (`git pull`) antes de salvar o seu trabalho. **Isso é um equívoco perigoso!**

O fluxo correto e seguro para o dia a dia é:

```text
1. Editar arquivos  ➔  2. git add / git commit  ➔  3. git pull  ➔  4. Resolver conflitos (se houver)  ➔  5. git push
```

```bash
# 1. Prepare e confirme suas alterações locais
git add .
git commit -m "feat: implementa tela de login"

# 2. Baixe as alterações feitas pelos seus colegas
git pull

# 3. Envie tudo atualizado para o repositório remoto
git push
```

---

## 1. Por que commitar ANTES de baixar as mudanças (`git pull`)?

Fazer o commit antes de atualizar com o servidor traz três grandes camadas de proteção:

* **🛡️ Proteção do seu código:** O Git só protege aquilo que foi commitado. Se você executar `git pull` com alterações soltas no diretório de trabalho (*uncommitted*), o Git pode travar a operação ou sobrepor alterações, tornando o descarte ou a recuperação muito mais complexos.
* **🕒 Histórico claro e rastreável:** Ao commitar antes, o Git identifica com precisão o momento em que a sua linha do tempo se separou da linha do tempo do seu colega. Isso facilita a identificação e a resolução de conflitos.
* **⏪ Ponto de restauração seguro (`git merge --abort`):** Se após o `git pull` surgir um conflito complexo e você se perder, basta rodar:
  ```bash
  git merge --abort
  ```
  O seu repositório voltará imediatamente ao estado exato do seu commit, antes de puxar as mudanças. Sem o commit prévio, essa rede de segurança não existiria!

> 💡 **E se o trabalho estiver pela metade e eu não quiser commitar ainda?**  
> Use o comando **`git stash`**. Ele "guarda no bolso" suas alterações temporárias, permitindo que você rode o `git pull` com a pasta limpa e depois recupere seu rascunho com `git stash pop`.

---

## 2. O que faz o comando `git config --global pull.rebase false`?

Ao executar `git pull`, o Git precisa decidir como juntar os commits novos vindos da nuvem com os seus commits locais. Existem duas estratégias principais:

### 🔹 Opção A: `pull.rebase false` (Merge Tradicional) — *Recomendado para Iniciantes*

Cria automaticamente um **commit de merge** para entrelaçar as duas linhas do tempo.

```bash
# Configuração recomendada (executar uma única vez no seu computador):
git config --global pull.rebase false
```

* **Vantagens:**
  * **Muito mais seguro e transparente:** Mantém o histórico real e a ordem cronológica dos acontecimentos.
  * **Resolução única:** Se houver conflitos entre o seu trabalho e o do colega, você resolve tudo de uma só vez no commit de merge.
* **Desvantagem:**
  * Cria linhas ramificadas no histórico visual e commits de merge automáticos (ex: `Merge branch 'main' of ...`).

---

### 🔸 Opção B: `pull.rebase true` (Rebase) — *Usado por equipes mais experientes*

Em vez de criar um commit de merge, ele "descola" temporariamente seus commits locais, aplica os commits que vieram do servidor e depois reaplica os seus commits no topo da fila.

* **Vantagens:** Histórico limpo e perfeitamente linear (uma linha reta de commits).
* **Desvantagens:**
  * Reescreve a linha do tempo local.
  * Se houver conflitos, você terá que resolvê-los commit por commit (se você tiver feito 4 commits locais que tocam no mesmo arquivo, terá que resolver o conflito até 4 vezes seguidas).

---

## 3. O que fazer quando der Conflito? (Passo a Passo)

Conflito não é erro! Ele apenas significa que você e seu colega alteraram a mesma linha do mesmo arquivo.

1. **Abra o arquivo conflitante:** Procure pelas marcações do Git:
   ```text
   <<<<<<< HEAD
   Seu código aqui (o que você fez)
   =======
   Código do colega aqui (o que veio do git pull)
   >>>>>>> origin/main
   ```
2. **Edite e escolha a versão final:** Converse com sua dupla, decida o que deve ficar, apague as marcações (`<<<<<<<`, `=======`, `>>>>>>>`).
3. **Conclua o Merge:**
   ```bash
   git add .
   git commit -m "fix: resolve conflitos de integração"
   git push
   ```

---

## 4. Prevenção de Conflitos: Divisão de Tarefas e Gestão Visual (Kanban)

> **💡 O melhor conflito é aquele que não acontece!**  
> Mais de 80% dos conflitos dolorosos de merge entre estudantes não são problemas técnicos do Git, mas sim **falhas de comunicação e sobreposição de tarefas**.

Se dois alunos decidem mexer no mesmo arquivo (por exemplo, ambos editando `app.js` ou a mesma rota) ao mesmo tempo sem alinhamento prévio, o conflito no Git é praticamente garantido.

### Como o Kanban e a Divisão de Tarefas Ajudam:

1. **Evita a sobreposição no mesmo arquivo/módulo:**
   * Aluno A assume o cartão: *"Criar formulário de login (`login.html` / `login.css`)"*.
   * Aluno B assume o cartão: *"Criar model de usuários e conexão com banco (`User.js` / `db.js`)"*.
   * Como cada um atua em partes isoladas do projeto, o Git junta tudo automaticamente sem nenhum conflito.
2. **Definição clara de "Dono da Tarefa" (WIP - Work in Progress):**
   * No quadro Kanban (seja no **Trello**, **GitHub Projects** ou post-its), cada cartão em *“Em Progresso”* deve ter um único responsável.
   * Dois alunos não devem atuar na mesma tarefa simultaneamente (a menos que seja em *Pair Programming*, onde compartilham a mesma tela/máquina).
3. **Casos em que ambos precisam mexer no mesmo arquivo:**
   * Arquivos centrais (como rotas, `package.json`, configurações ou menu de navegação) costumam sofrer alterações de todos.
   * **A regra aqui é o alinhamento rápido:** *"Vou adicionar a dependência X no `package.json` agora, já vou commitar e subir para você poder puxar antes de instalar a sua"*.

---

## 5. Dicas Rápidas para Trabalhar em Equipe sem Estresse

1. **Adote um Kanban simples:** Use o próprio **GitHub Projects** ou **Trello** para deixar visível quem está fazendo o quê.
2. **Trabalhe em Branches separadas:** Evite que todo mundo faça commits diretamente na `main`:
   ```bash
   # Cria e entra em uma nova branch para a sua funcionalidade
   git checkout -b feature/minha-tarefa
   ```
3. **Puxe alterações com frequência (`git pull`):** Antes de começar a programar no dia, atualize seu repositório local para começar sempre com o código mais recente.
4. **Comunique-se sempre:** Avise a equipe antes de alterar arquivos centrais ou estruturas compartilhadas.
5. **Visualize o histórico de forma amigável no terminal:**
   ```bash
   git log --oneline --graph --all
   ```

