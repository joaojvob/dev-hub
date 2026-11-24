# Dev Hub Pessoal — Especificação Técnica

Aplicação web pessoal (rodando inicialmente local via Docker, mas preparada para deploy em VPS com HTTPS) para centralizar:
1. Gestão de Releases e Homologação integrada ao Jira (substituindo planilhas).
2. Bloco de Notas estilo Notion com suporte a Markdown e blocos de código.
3. Cofre criptografado de credenciais e conexões de banco de dados.
4. Lista de tarefas e lembretes com prazos.

## Stack Obrigatória
- **Backend:** PHP 8.2+, Laravel 11+
- **Frontend:** Inertia.js + Vue 3 (Composition API com `<script setup>`) + Tailwind CSS
- **Autenticação:** Laravel Breeze (Inertia + Vue). Registro público deve ser controlado por `ALLOW_REGISTRATION=false` no `.env`.
- **Banco de Dados:** SQLite (padrão local) ou MySQL, mantendo migrations 100% compatíveis com ambos via Eloquent/Schema Builder.
- **Ícones:** `lucide-vue-next`
- **Editor de Notas:** `@tiptap/vue-3`, `@tiptap/starter-kit`, `@tiptap/extension-code-block-lowlight`
- **Infra:** `docker-compose.yml` (PHP-FPM + Nginx + Node/Vite)

## Regras e Guardrails para o Agente
- Nunca commitar tokens ou senhas reais; usar `.env` e `.env.example`.
- Campos sensíveis no Cofre de Credenciais DEVEM usar o cast `'encrypted'` do Laravel no Model.
- Nunca executar comandos destrutivos de banco (`migrate:fresh`) sem avisar antes.
- Manter controllers enxutos: isolar lógica de integração externa em `app/Services/`.
- Implementar uma fase por vez e validar antes de avançar para a próxima.

---

## Modelagem de Dados (Migrations & Models)

### Módulo 1: Releases & Jira
1. **`releases`**
   - `id`, `name` (ex: `Homolog 2026.11`), `target_branch` (nullable), `status` (`montando`, `em_homologacao`, `publicada`), `notes` (text, nullable), `timestamps`
2. **`release_groups`**
   - `id`, `release_id` (FK cascade), `order` (integer, default 1), `name` (ex: `Procedimentos e SIGTAP`), `timestamps`
3. **`merge_items`**
   - `id`, `release_group_id` (FK cascade), `branch_name` (string), `merge_status` (`pendente`, `mergeada`, `conflito`, `revertida`), `homolog_status` (`aguardando`, `em_teste`, `aprovada`, `reprovada`), `notes` (text, nullable), `timestamps`
4. **`jira_issues`** (espelho local sincronizado via API; relação 1:N com `merge_items`)
   - `id`, `merge_item_id` (FK cascade), `jira_key` (string, index, ex: `VSW-2711`), `summary` (string, nullable), `issue_type` (string, nullable), `priority` (string, nullable), `assignee` (string, nullable), `jira_status` (string, nullable), `last_synced_at` (timestamp, nullable), `timestamps`

### Módulo 2: Notas (Estilo Notion)
5. **`notes`**
   - `id`, `title` (string), `folder` (string, nullable), `content` (longText — HTML ou JSON do TipTap), `tags` (json, nullable), `is_pinned` (boolean, default false), `timestamps`

### Módulo 3: Cofre de Conexões e Senhas
6. **`credentials`**
   - `id`, `name` (string), `category` (`mysql`, `postgres`, `mongodb`, `redis`, `ssh`, `api`, `login`, `other`), `environment` (`local`, `homolog`, `prod`), `host` (string, nullable), `port` (string, nullable), `database` (string, nullable), `username` (text, cast `encrypted`, nullable), `password` (text, cast `encrypted`, nullable), `extra_notes` (text, cast `encrypted`, nullable), `timestamps`

### Módulo 4: Tarefas e Prazos
7. **`tasks`**
   - `id`, `title` (string), `notes` (text, nullable), `due_at` (datetime, nullable), `priority` (`baixa`, `media`, `alta`), `completed_at` (timestamp, nullable), `merge_item_id` (FK nullable, nullOnDelete), `timestamps`

---

## Integração Jira REST API v3
- Variáveis no `.env`: `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`.
- `app/Services/Jira/JiraClient.php`: Usa `Http::withBasicAuth($email, $token)` contra `POST {base_url}/rest/api/3/search/jql`.
- `app/Services/Jira/JiraSyncService.php`: Recebe uma `Release`, coleta todas as `jira_key` vinculadas, consulta em lote via JQL `key in (VSW-1, VSW-2, ...)` e atualiza `summary`, `issue_type`, `priority`, `assignee`, `jira_status` e `last_synced_at`.
- Destacar visualmente na UI divergências (ex: item na release mas status no Jira ainda em desenvolvimento/impedido).

---

## Interface e Navegação (Vue 3 + Tailwind)
- Layout com sidebar fixa (dark mode limpo): `Dashboard`, `Releases`, `Notas`, `Cofre`.
- **Busca Global (`Ctrl + K`):** Busca rápida em `jira_issues` (`jira_key`, `summary`), `notes` (`title`) e `credentials` (`name`, `host`).
- **Dashboard ("Hoje"):**
  - Bloco de tarefas atrasadas / vencendo hoje e próximos 7 dias.
  - Resumo da Release ativa (total de branches, pendentes de merge, divergências do Jira).
  - Notas fixadas (`is_pinned`).
- **Releases:**
  - Visão agrupada por `release_groups` (colapsável), mostrando `branch_name`, badges clicáveis das `jira_issues` (com link externo para o Jira), status do Jira vs. status de merge/homologação, e filtros por responsável/prioridade.
  - Botão **Sincronizar Jira** (sync em lote).
  - Modal de **Entrada Rápida** (colar várias chaves `VSW-XXXX` separadas por vírgula) e **Importador de CSV** compatível com colunas: `Grupo`, `Tema / fluxo`, `Demanda(s) Jira`, `Branch p/ merge`.
  - Botão **Copiar Resumo Markdown** (agrupado por tema, pronto para enviar ao QA/time).
- **Notas:** Sidebar de pastas/notas fixadas + editor TipTap com atalhos Markdown e auto-save (debounce).
- **Cofre:** Lista filtrável por categoria/ambiente; senhas mascaradas com botão de revelar e botão de **Copiar em 1 clique** (incluindo copiar string de conexão para bancos).

---

## Plano de Execução por Fases

- **Fase 1 (Setup + Base):** Inicializar Laravel 11 + Breeze (Inertia/Vue 3) + Tailwind + `lucide-vue-next` + Docker Compose + desabilitar registro público via `.env` + criar todas as migrations, models, relacionamentos e um Seeder com usuário inicial.
- **Fase 2 (Releases & Jira):** CRUD de Releases, Groups e MergeItems + `JiraClient` e `JiraSyncService` + tela agrupada com filtros, entrada rápida, importador de CSV e exportação Markdown.
- **Fase 3 (Notas + Cofre):** Módulo de Notas com TipTap e auto-save + Módulo de Cofre com casts `encrypted`, máscara de senha e cópia em 1 clique.
- **Fase 4 (Tarefas + Dashboard + Ctrl+K):** CRUD de Tarefas (com vínculo opcional a `merge_item_id`), Dashboard consolidado ("Hoje") e modal de busca global (`Ctrl + K`).