# TaskLab — clone de estudo do ClickUp

Recriação funcional do ClickUp em React para estudar como o app funciona por dentro: hierarquia, modelo de dados, views, campos personalizados, automações e módulos extras. Marca e visual são próprios (nome e logo neutros); a mecânica segue o ClickUp.

Tudo roda no navegador, sem backend. Os dados ficam no `localStorage` e o app já abre com um workspace de demonstração.

## Rodar

```bash
npm install
npm run dev      # http://localhost:5173
npm run build    # gera /dist estático (dá pra subir em Vercel, Netlify, cPanel…)
```

No console do navegador, `__store.getState()` mostra o estado inteiro — útil para ver como cada ação muda os dados.

## Mapa de features

| Área | O que tem |
|---|---|
| **Hierarquia** | Workspace → Space → Folder → List → Task → Subtask (aninhadas). Criar, renomear (duplo clique), cor, ícone, privado, duplicar lista, excluir, favoritos, “Tudo” (Everything) |
| **Status** | Por Space, com herança/override em Folder e List. 4 tipos: não iniciado, ativo, concluído, fechado. Editor com cor, ordem e tipo; adicionar status direto no Quadro |
| **Tarefa** | Nome, descrição rich text (com comandos `/`), status, múltiplos responsáveis, seguidores, datas início/entrega, prioridade, tags, estimativa, sprint points, tipo de tarefa (Tarefa/Marco/Bug/Reunião + próprios), ID personalizado (TL-123), recorrência (criar nova ou reabrir), checklists com responsável por item, dependências (aguardando/bloqueando), tarefas vinculadas, anexos (arquivo ou link), tarefas em múltiplas listas, mover, duplicar, converter em subtarefa, arquivar, lixeira |
| **Colaboração** | Comentários com @menção (autocomplete), respostas em thread, reações, comentários atribuídos (resolver/reabrir), feed de atividade/histórico, caixa de entrada com Principal/Não lidas/Mais tarde/Limpas, agrupada por tarefa |
| **Views** | Lista, Quadro (Kanban), Tabela, Calendário, Gantt, Timeline, Carga de trabalho, Formulário. Várias views por localização, fixar, duplicar, renomear |
| **Motor de views** | Filtros avançados E/OU (status, responsável, datas relativas, campos personalizados, dependências…), agrupar por qualquer campo, ordenação multinível, subtarefas recolhidas/expandidas/separadas, ocultar fechadas, modo “Eu”, colunas configuráveis, busca na view |
| **Interações** | Arrastar e soltar entre grupos e reordenar (Lista/Quadro), limite WIP e colunas recolhíveis no Quadro, arrastar no Calendário (inclui “sem data”), mover/redimensionar barras e criar dependências arrastando no Gantt, reatribuir arrastando na Timeline, edição inline em todas as células, ações em massa |
| **Campos personalizados** | Texto, texto longo, número, dinheiro, dropdown, rótulos (multi), data, checkbox, email, telefone, URL, avaliação, progresso manual, pessoas, fórmula (`{Campo} * 1.15`), rollup das subtarefas. Escopo por Space/Folder/List com herança, campo obrigatório |
| **Automações** | Gatilho → condições → ações em sequência. 18 gatilhos (status muda, tarefa criada, data chega, subtarefas resolvidas, desbloqueada…), condições por campo, 23 ações (mudar status, atribuir, comentar com `{{task.name}}`, criar subtarefa, mover/adicionar a lista, aplicar template, notificar…). Herança pela hierarquia, sugestões prontas, log de execução, proteção contra loop |
| **Tabela** | Coluna de nome fixa, ordenação por clique no cabeçalho, rodapé com soma/média/mín/máx/contagem, exportar CSV |
| **Formulários** | Construtor com campos nativos e personalizados, obrigatório, texto de ajuda, lógica condicional, status/responsável/tipo padrão, pré-visualização ao vivo, contagem de respostas |
| **Início (Meu trabalho)** | Hoje e atrasadas, próximas, sem data, concluídas, lembretes, comentários atribuídos, atualizadas recentemente |
| **Docs** | Wikis e páginas aninhadas, ícone, editor rich text com `/`, checklist clicável, tabelas, localização na hierarquia, exportar HTML |
| **Dashboards** | Cards de cálculo, status (rosca/barras), por responsável, por prioridade, criadas x concluídas, tempo rastreado, lista de tarefas, metas, notas. Fonte por localização, arrastar para reorganizar, largura configurável |
| **Metas** | Pastas, alvos numéricos, monetários, verdadeiro/falso e por tarefas, progresso agregado |
| **Tempo** | Cronômetro global no topo, registro manual (“1h 30m”), faturável, planilha semanal por pessoa, CSV |
| **Templates** | Salvar tarefa (com subtarefas e checklists) ou lista inteira (status, campos e tarefas) e reutilizar |
| **Workspace** | ClickApps liga/desliga, membros com papel e capacidade, “simular como” outro membro, tags, tipos de tarefa, exportar/importar JSON, tema escuro, busca global `Ctrl+K`, atalhos (`T` nova tarefa, `[` barra lateral) |

Ficaram de fora desta versão: Chat, Whiteboards, Mind Map, Map view, IA, integrações externas, permissões granulares e sincronização multiusuário em tempo real.

## Como o código está organizado

```
src/
  store/
    store.js        ← todo o estado + mutações (Zustand persistido). updateTask/createTask
                      geram atividade, notificações e disparam as automações
    seed.js         ← workspace de demonstração
  lib/
    selectors.js    ← regras da hierarquia: herança de status e campos, progresso, fórmulas
    query.js        ← motor das views: filtrar → ordenar → agrupar (+ patch ao soltar num grupo)
    automations.js  ← catálogo de gatilhos/condições/ações e avaliação
    actions.js      ← validações da UI (campos obrigatórios, dependências ao concluir)
  components/       ← Sidebar, Topbar, TaskDetail, editor rich text, campos, modais
  views/            ← List, Board, Table, Calendar, Gantt, Timeline, Workload, Form
  pages/            ← Início, Inbox, Docs, Dashboards, Metas, Horas, Templates, Lixeira, Config.
```

### Três ideias centrais para estudar

1. **Uma tarefa, muitas views.** Nenhuma view guarda tarefas. Cada view guarda só uma *config* (filtros, agrupamento, ordenação, colunas) e roda `runView()` sobre o mesmo array de tarefas. Arrastar um card entre colunas é só calcular o *patch* do grupo de destino (`patchForGroup`).
2. **Herança pela hierarquia.** Status, campos personalizados e automações são definidos num nível e valem para tudo abaixo. `getStatusesForList` sobe List → Folder → Space; `fieldAppliesToList` faz o mesmo para campos.
3. **Eventos → automações.** `updateTask` compara antes/depois, gera eventos (`status_changed`, `assignee_added`…) e o motor procura automações cujo gatilho, localização e condições batem. Ações de automação passam pelas mesmas mutações, com limite de profundidade para não entrar em loop.

## Stack

React 18, Vite, Tailwind CSS v4, Zustand (persist), dnd-kit, Recharts, date-fns, lucide-react.
