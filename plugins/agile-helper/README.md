# agile-helper

Plugin do Claude Code com skills de apoio ao fluxo agile de desenvolvimento: criação/manutenção de itens de QA no Jira como criar Épicos, User Stories, Critérios de aceite, e casos de teste, quebra de User Stories em Subtasks, quando solicitado sugere tasks para entrar nas próximas sprints planejadas, e abertura de Pull Requests seguindo o padrão de Título e Descrição.

## Skills incluídas

### `qa-assistant`
Cria e mantém itens de backlog de QA (Epic, User Stories, Critérios de Aceite em
Gherkin, Test Cases) a partir de um requisito — só faz o que for pedido (só o epic, só quebrar
em stories, só melhorar uma descrição, cascata completa, etc.). Também edita/enriquece épicos e
stories que já existem no Jira. Cria ou atualiza o issue formatado em ADF (painel de info, blocos
de código Gherkin).

**Quando usar:** pedir para criar/quebrar/refinar/completar uma story, epic, critério de aceite
ou caso de teste; preparar itens para sprint planning; registrar/atualizar isso no Jira.

**Quando não usar:** perguntas operacionais rápidas sobre o Jira; revisão/teste de código já
implementado.

### `break-down-into-tasks`
Quebra User Stories que já estão no Jira em Subtasks técnicas ligadas diretamente à Story (uma
por critério de aceite, com tag `[front-end]`/`[back-end]`/`[integration]` no título e refletida
no campo Labels). Cria Tasks (trabalho técnico habilitador, sem relação com o usuário final) só
quando pedido explicitamente. Também sugere a composição da próxima sprint com base nas
Subtasks/Tasks já criadas, Priority e story points — sempre como sugestão para o PM revisar.

**Quando usar:** "quebra essa story em subtasks", "cria as subtasks de dev para a US X", "monta a
próxima sprint", "sugere o que entra na sprint".

**Quando não usar:** criar a User Story ou seus critérios de aceite (isso é escopo do
`qa-assistant`); operações simples e pontuais no Jira que não envolvam quebra de trabalho ou
sprint planning.

### `pr-helper`
Prepara e abre o Pull Request de uma atividade (feature/task/subtask/fix/refactor/docs) seguindo
a convenção de branch e template do time: confirma se a branch atual é a certa (ou ajuda a criar
a correta), organiza mudanças não commitadas em commits atômicos e lógicos, lê o log/diff para
redigir o conteúdo, monta a descrição no template oficial (Change Type, PR Purpose, Quality
Checklist, Test Evidence, Additional Notes) e sobe o PR via GitHub CLI (`gh pr create`) depois de
revisado.

**Quando usar:** "sobe o PR dessa atividade", "abre um PR para EDGE-XXXX", "terminei essa task,
prepara o PR".

**Quando não usar:** revisão de código de um PR já aberto; merge, aprovação ou resolução de
conflitos em PR já existente.

## Instalação

Este plugin é distribuído através do marketplace do time. Em um terminal interativo do
Claude Code:

```bash
/plugin install agile-helper@<nome-do-marketplace>
```

Depois de instalado, as skills ficam disponíveis em qualquer projeto onde o plugin estiver
habilitado — não só em um repositório específico.

## Estrutura

```
agile-helper/
├── .claude-plugin/
│   └── plugin.json        # manifesto do plugin
└── skills/
    ├── qa-assistant/SKILL.md
    ├── break-down-into-tasks/SKILL.md
    └── pr-helper/SKILL.md
```
