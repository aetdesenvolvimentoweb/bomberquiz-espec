# Módulo 3 — Conteúdo (admin): eixos, matérias, perguntas

> Identificador dos RFs deste módulo: `CONT-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- Este módulo cobre o **poder administrativo total** sobre o catálogo de conteúdo: eixos temáticos, matérias e perguntas. Operações restritas a `role=admin`.
- O **Módulo 4** (`rf/content-partner.md`) reusa a mesma estrutura de dados de pergunta definida aqui, mas com permissões e workflow específicos do parceiro (fila de aprovação, escopo limitado).
- Toda operação destrutiva (arquivar eixo/matéria/pergunta) é **soft-delete** via campo `status` ou `archived_at` — preserva FKs e estatísticas históricas (mesmo princípio de ADR-0015 para usuários).
- Toda ação administrativa registra entrada em `audit_log` (schema em `arquitetura.md` § Audit log) com o `action` indicado em cada CA.
- Listagens são paginadas conforme `api.md` § Paginação, ordenação e filtros (`page`/`page_size`, default 20, máx. 100).
- Entrada inválida → HTTP 422 com lista de campos; acesso de não-admin → HTTP 403.
- A definição da estrutura mínima da pergunta (4 alternativas, gabarito único, justificativa obrigatória, imagem opcional) está em **ADR-0016**.

## Regras gerais (aplicam-se a vários RFs)

| Regra | Valor |
|---|---|
| Tamanho mín./máx. do nome de eixo | 3–80 caracteres |
| Peso TAP do eixo | inteiro ≥ 0 (nº de questões esperadas na prova para aquele eixo, conforme edital vigente) |
| Tamanho mín./máx. do nome da matéria | 3–120 caracteres |
| Fonte oficial da matéria (texto livre) | até 240 caracteres |
| Tamanho do enunciado da pergunta | 10–2.000 caracteres |
| Número de alternativas | **fixo em 4** (ADR-0016) |
| Tamanho de cada alternativa | 1–500 caracteres |
| Justificativa da resposta | **obrigatória**, 10–2.000 caracteres (ADR-0016) |
| Referência à fonte na pergunta | texto livre opcional, até 240 caracteres |
| Imagem da pergunta | opcional, máx. 1, ≤ 2 MB, `image/jpeg`/`png`/`webp`, armazenada no Cloudinary (ADR-0012, revisado 2026-07-19) |
| Status possíveis de eixo/matéria | `active`, `archived` |
| Status possíveis de pergunta | `draft`, `pending_review` (só parceiro), `published`, `archived` |

## Estrutura de dados (resumo do domínio)

### Eixo Temático
- `id`, `name` (único, case-insensitive), `description?`, `tap_weight` (inteiro, nº de questões do eixo na prova), `status`, `created_at`, `archived_at?`.

### Matéria
- `id`, `axis_id` (FK), `name` (único dentro do eixo, case-insensitive), `official_source?` (texto livre), `status`, `created_at`, `archived_at?`.

### Pergunta
- `id`, `subject_id` (FK matéria), `statement` (enunciado), `alternatives` (array de exatamente 4 strings), `correct_index` (0..3), `explanation` (justificativa, obrigatória), `source_reference?` (texto livre), `image_url?` (URL pública no Cloudinary), `status`, `author_id` (FK users), `created_at`, `updated_at`, `published_at?`, `archived_at?`, `stats_reset_at?`.

---

## CONT-RF-001 — Listar eixos temáticos

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Sessão ativa com `role=admin`.

**Descrição:**
Lista todos os eixos temáticos do sistema, com filtros.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/axes` retorna `{ id, name, description, tap_weight, status, subjects_count, created_at }`, paginado.
- **CA-2:** Filtros: `status` (`active`/`archived`/`all`, padrão `active`), `q` (busca por nome — prefixo, case-insensitive).
- **CA-3:** `subjects_count` reflete apenas matérias `active`.

**Erros previstos:**
- **E-1:** Acesso de não-admin → HTTP 403.

---

## CONT-RF-002 — Criar eixo temático

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Cria novo eixo no catálogo.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/axes` com `{ name, description?, tap_weight }`. Validação contra regras gerais.
- **CA-2:** `name` é único no sistema (case-insensitive). Conflito → E-1.
- **CA-3:** Em sucesso, eixo nasce com `status=active`. HTTP 201 com payload do eixo.
- **CA-4:** Entrada no `audit_log` com `action=create_axis`.
- **CA-5:** `tap_weight=0` é válido (eixo cadastrado mas fora do TAP do ano vigente). Usado pelo "simulado TAP" no Módulo 5.

**Erros previstos:**
- **E-1:** Nome duplicado → HTTP 409.
- **E-2:** Validação Zod falha → HTTP 422.

---

## CONT-RF-003 — Editar eixo temático

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Eixo existe com `status=active`.

**Descrição:**
Atualiza nome e/ou descrição de um eixo. Não altera status (operação dedicada em CONT-RF-004).

**Critérios de aceitação:**
- **CA-1:** `PATCH /admin/axes/:id` com `{ name?, description?, tap_weight? }`. Edição parcial.
- **CA-2:** Se `name` mudou, valida unicidade (CA-2 de CONT-RF-002).
- **CA-3:** Em sucesso, HTTP 200 com payload do eixo. Entrada em `audit_log`.

**Erros previstos:**
- **E-1:** Eixo não encontrado ou `archived` → HTTP 404.
- **E-2:** Nome duplicado → HTTP 409.

---

## CONT-RF-004 — Arquivar / desarquivar eixo

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Soft-delete reversível: marca eixo como `archived` (ou volta para `active`). Eixo arquivado **não aparece** em listagens públicas nem é selecionável para novas matérias, mas suas matérias e perguntas continuam acessíveis (até serem individualmente arquivadas).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/axes/:id/archive` zera/preenche `archived_at` e troca `status`.
- **CA-2:** Arquivar eixo que tem matérias **ativas** emite **aviso** (não bloqueia): "Este eixo tem N matérias ativas. Elas continuarão visíveis até serem arquivadas individualmente."
- **CA-3:** Em sucesso, HTTP 200; `audit_log` registra.
- **CA-4:** Não há cascata automática para matérias ou perguntas.
- **CA-5:** **Label da UI:** botões devem dizer **"Desativar eixo"** e **"Reativar eixo"** (mental model do admin), mesmo que internamente o campo `status` use os valores `active`/`archived` por consistência técnica com perguntas (CONT-RF-012).

**Erros previstos:**
- **E-1:** Eixo não encontrado → HTTP 404.

---

## CONT-RF-005 — Listar matérias

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Lista matérias do catálogo, com filtros.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/subjects` retorna `{ id, axis_id, axis_name, name, official_source, status, questions_count, created_at }`, paginado.
- **CA-2:** Filtros: `axis_id`, `status` (padrão `active`), `q` (nome).
- **CA-3:** `questions_count` reflete apenas perguntas `published`.

---

## CONT-RF-006 — Criar matéria

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Existe ao menos 1 eixo ativo.

**Descrição:**
Cria matéria vinculada a um eixo, com fonte oficial.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/subjects` com `{ axis_id, name, official_source? }`. Validação contra regras gerais.
- **CA-2:** `axis_id` deve apontar para eixo `active`. Eixo `archived` → E-1.
- **CA-3:** `name` é único **dentro do eixo** (case-insensitive). Conflito → E-2.
- **CA-4:** Em sucesso, matéria nasce com `status=active`. HTTP 201. Entrada em `audit_log`.

**Erros previstos:**
- **E-1:** Eixo inexistente ou arquivado → HTTP 422.
- **E-2:** Nome duplicado dentro do eixo → HTTP 409.

---

## CONT-RF-007 — Editar matéria

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Atualiza qualquer campo da matéria, incluindo `axis_id` (move de eixo).

**Critérios de aceitação:**
- **CA-1:** `PATCH /admin/subjects/:id` com qualquer subconjunto de `{ axis_id?, name?, official_source? }`.
- **CA-2:** Mudança de `axis_id` exige novo eixo `active` e re-validação de unicidade do nome no eixo destino.
- **CA-3:** Perguntas vinculadas **acompanham** automaticamente (são filhas da matéria, não do eixo).
- **CA-4:** Entrada em `audit_log`.

**Erros previstos:** idênticos a CONT-RF-006.

---

## CONT-RF-008 — Arquivar / desarquivar matéria

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Soft-delete reversível. Matéria `archived` não pode receber novas perguntas nem aparece em quizzes.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/subjects/:id/archive` alterna `status` e preenche/zera `archived_at`.
- **CA-2:** Arquivar matéria com perguntas `published` emite aviso (não bloqueia): "N perguntas dessa matéria deixarão de aparecer em novos quizzes. Respostas históricas permanecem."
- **CA-3:** Perguntas filhas **não** são automaticamente arquivadas (mantém-se acessíveis para edição, mas não para novos quizzes).
- **CA-4:** Entrada em `audit_log`.
- **CA-5:** **Label da UI:** botões devem dizer **"Desativar matéria"** e **"Reativar matéria"** (mental model do admin), mesmo que internamente o campo `status` use `active`/`archived`. Acomoda o ciclo natural do edital — matérias entram e saem a cada ano sem precisar de exclusão definitiva.

---

## CONT-RF-009 — Listar perguntas

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Lista perguntas do catálogo, com filtros amplos para gestão e revisão.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/questions` retorna `{ id, subject_id, subject_name, axis_name, statement_preview, status, author_id, author_name, has_image, created_at, published_at, archived_at, total_answers, accuracy }`, paginado.
- **CA-2:** Filtros: `subject_id`, `axis_id`, `status` (multi-select, padrão `published,pending_review`), `author_id`, `q` (busca em enunciado, prefixo case-insensitive), `has_image`.
- **CA-3:** `statement_preview` traz os primeiros 160 caracteres do enunciado.
- **CA-4:** `accuracy` = % de acertos sobre `total_answers` (zero se `total_answers=0`).
- **CA-5:** `GET /admin/questions/:id` retorna o detalhe completo da pergunta (todos os campos de CONT-RF-011 CA-1, não só o resumo de CA-1 acima) — usado pela tela de edição do admin, que precisa do enunciado/alternativas/gabarito completos, não do preview truncado. Adicionado durante a implementação (2026-07-19): não estava no inventário original de `api.md`, mas é necessário para qualquer UI de edição funcionar.

---

## CONT-RF-010 — Criar pergunta (admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Existe matéria `active`.

**Descrição:**
Admin cadastra nova pergunta. Diferente do parceiro (Módulo 4), a pergunta criada por admin **vai direto para `published`** sem fila de aprovação (ADR-0016).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/questions` com `{ subject_id, statement, alternatives[4], correct_index, explanation, source_reference?, image_url? }`. Todas as validações de regras gerais.
- **CA-2:** `alternatives` é array de exatamente 4 strings não-vazias e não-duplicadas (case-sensitive sobre conteúdo trimmed).
- **CA-3:** `correct_index` ∈ {0,1,2,3}.
- **CA-4:** `subject_id` aponta para matéria `active`. Matéria arquivada → E-1.
- **CA-5:** Em sucesso, pergunta nasce com `status=published`, `author_id=<admin>`, `published_at=now()`. HTTP 201 com payload da pergunta. Entrada em `audit_log` com `action=publish_question_direct`.
- **CA-6:** Admin pode optar por salvar como **rascunho** (`?as_draft=true`) — pergunta nasce com `status=draft`, sem `published_at`. Não aparece em quizzes.

**Erros previstos:**
- **E-1:** Matéria arquivada/inexistente → HTTP 422.
- **E-2:** Validação falha (campos, número de alternativas, índice) → HTTP 422.

---

## CONT-RF-011 — Editar pergunta

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Edita qualquer campo de qualquer pergunta (de admin ou de parceiro). Admin tem poder total — sem janela de tempo, sem restrição por número de respostas registradas.

**Critérios de aceitação:**
- **CA-1:** `PATCH /admin/questions/:id` com qualquer subconjunto editável: `{ subject_id?, statement?, alternatives?, correct_index?, explanation?, source_reference?, image_url?, reset_stats? }`.
- **CA-2:** Mudar `subject_id` exige matéria `active`.
- **CA-3:** Se `alternatives` for editado, mantém regra de exatamente 4 distintas; `correct_index` deve continuar válido.
- **CA-4:** **Reset de estatísticas:** o payload aceita `reset_stats: boolean` (default `false`). Quando `true`, todas as respostas vinculadas à pergunta são desconsideradas das estatísticas a partir deste momento (`stats_reset_at = now()`); respostas antigas continuam armazenadas para auditoria/replay. Ver ADR-0016.
- **CA-5:** Frontend do admin **sugere automaticamente** `reset_stats=true` quando `correct_index` mudou (a UI exibe checkbox pré-marcado); para mudanças em `statement` ou `alternatives` o checkbox é exibido desmarcado para o admin decidir.
- **CA-6:** Em sucesso, HTTP 200 com payload completo da pergunta. `updated_at` é atualizado. Entrada em `audit_log` com diff resumido.

**Erros previstos:**
- **E-1:** Pergunta `archived` → HTTP 409 ("desarquive antes de editar").
- **E-2:** Validação falha → HTTP 422.

---

## CONT-RF-012 — Arquivar / desarquivar / excluir pergunta

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Soft-delete reversível (`archive`) é o caminho padrão — pergunta `archived` deixa de aparecer em quizzes mas respostas históricas permanecem. Hard-delete só é permitido para perguntas **sem nenhuma resposta registrada** (cenário "cadastrei errado e ninguém respondeu ainda").

**Critérios de aceitação:**
- **CA-1:** `POST /admin/questions/:id/archive` alterna `status` entre `archived` e o status anterior; `archived_at` é preenchido/zerado. Entrada em `audit_log`.
- **CA-2:** `DELETE /admin/questions/:id` executa hard-delete **somente se** `total_answers = 0`. Em sucesso, a pergunta é removida fisicamente do banco junto com a imagem no Cloudinary (best-effort). Entrada em `audit_log` com `action=delete_question` registra o ID, autor e matéria antes da exclusão (para auditoria).
- **CA-3:** Tentativa de hard-delete com `total_answers > 0` → E-1 com mensagem "Esta pergunta já foi respondida — arquive-a em vez de excluir, para preservar o histórico dos clientes."
- **CA-4:** Hard-delete é irreversível; UI exige confirmação explícita ("Excluir definitivamente").

**Erros previstos:**
- **E-1:** Hard-delete bloqueado por existência de respostas → HTTP 409.

---

## CONT-RF-013 — Upload de imagem da pergunta

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Faz upload da imagem opcional vinculada a uma pergunta (diagrama, equipamento, planta-baixa). Armazenada no Cloudinary (ADR-0012, revisado 2026-07-19 — antes R2).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/questions/:id/image` (multipart) valida tipo (`image/jpeg`/`png`/`webp`) e tamanho (≤ 2 MB). Backend valida MIME real, não confiar no header.
- **CA-2:** Upload via porta `IQuestionImageStoragePort` (em `application/`), adapter Cloudinary em `infra/storage/cloudinary.adapter.ts`. URL devolvida grava em `questions.image_url`.
- **CA-3:** Substituição: novo upload sobre imagem existente tenta deletar a antiga no Cloudinary (best-effort).
- **CA-4:** `DELETE /admin/questions/:id/image` zera `image_url` e tenta deletar do Cloudinary.

**Erros previstos:**
- **E-1:** Tipo inválido → HTTP 422.
- **E-2:** Tamanho excedido → HTTP 413.
- **E-3:** Falha no Cloudinary → HTTP 502.

---

## CONT-RF-014 — Listar perguntas pendentes de revisão (fila do admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Inbox de perguntas enviadas por parceiros (Módulo 4) que aguardam aprovação.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/questions/pending` retorna apenas perguntas com `status=pending_review`, ordenadas por `created_at` ascendente (mais antigas primeiro — FIFO).
- **CA-2:** Mesmos campos da listagem normal (CONT-RF-009 CA-1) + `submitted_at` (`created_at` da pergunta) + `partner_pending_count` (quantas outras pendências o mesmo parceiro tem).
- **CA-3:** Paginação padrão 20.
- **CA-4:** Endpoint também aceita filtro por `author_id` (revisar todas as pendências de um parceiro de uma vez).

---

## CONT-RF-015 — Aprovar pergunta de parceiro

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Pergunta com `status=pending_review`.

**Descrição:**
Admin revisa e aprova a pergunta, publicando-a.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/questions/:id/approve` com `{ notes? }` opcional.
- **CA-2:** Em sucesso, `status=published`, `published_at=now()`, `reviewed_by=<admin>`, `reviewed_at=now()`.
- **CA-3:** Entrada em `audit_log` com `action=approve_question`. Notas, se houver, ficam armazenadas para feedback ao parceiro (consultáveis em PART-RF-007).
- **CA-4:** **Notificação ao parceiro por dois canais:** (a) badge no app (contador `unread_review_events` consumido por PART-RF-008); (b) **e-mail transacional** via Resend (ADR-0012) com assunto "Sua pergunta foi aprovada", contendo enunciado-resumo e link direto para a pergunta. Em caso de aprovação com edição prévia pelo admin, o e-mail menciona "aprovada com alterações".
- **CA-5:** Aprovação **pode incluir edições** — admin pode editar (CONT-RF-011) antes de aprovar; se houver edição prévia, isso aparece no histórico do parceiro como "aprovada com alterações".

**Erros previstos:**
- **E-1:** Pergunta não está em `pending_review` → HTTP 409.

---

## CONT-RF-016 — Rejeitar pergunta de parceiro

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Pergunta com `status=pending_review`.

**Descrição:**
Admin recusa a pergunta. **Motivo obrigatório** para que o parceiro entenda o que ajustar.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/questions/:id/reject` com `{ reason }` (10–500 caracteres, obrigatório).
- **CA-2:** Em sucesso, `status=draft` (volta para o parceiro corrigir), `reviewed_by=<admin>`, `reviewed_at=now()`, `rejection_reason=<reason>`.
- **CA-3:** Entrada em `audit_log` com `action=reject_question`. **Notificação ao parceiro por dois canais:** (a) badge no app (incrementa `unread_review_events`); (b) **e-mail transacional** via Resend (ADR-0012) com assunto "Sua pergunta precisa de ajustes", contendo o motivo da rejeição na íntegra e link direto para reedição.
- **CA-4:** O parceiro pode reeditar e reenviar (volta para `pending_review`) — sem limite de iterações no MVP.

**Erros previstos:**
- **E-1:** Pergunta não está em `pending_review` → HTTP 409.
- **E-2:** Motivo ausente ou abaixo do mínimo → HTTP 422.

---

## CONT-RF-017 — Cálculo diário do nível de dificuldade

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (job agendado).

**Descrição:**
Job recorrente que recalcula o `difficulty_level` de toda pergunta `published` com base em `accuracy` e `total_answers`. Executado **diariamente às 00:00** (fuso `America/Sao_Paulo`).

**Critérios de aceitação:**
- **CA-1:** O job percorre todas as perguntas com `status=published` e recalcula:
  - `total_answers < 30` → `difficulty_level = unrated` ("Não classificada").
  - `accuracy ≥ 70%` → `easy`.
  - `40% ≤ accuracy < 70%` → `medium`.
  - `accuracy < 40%` → `hard`.
- **CA-2:** `accuracy` considera apenas respostas posteriores a `stats_reset_at` (se existir); respostas anteriores ao reset não entram no cálculo.
- **CA-3:** O resultado é persistido em `questions.difficulty_level` (enum: `unrated`/`easy`/`medium`/`hard`) e em `questions.difficulty_recomputed_at`.
- **CA-4:** Job é idempotente (rodar duas vezes no mesmo dia produz o mesmo resultado).
- **CA-5:** Falhas do job ficam em log e não afetam o resto do sistema — o valor anterior permanece até a próxima execução bem-sucedida.
- **CA-6:** Perguntas `archived` e `draft` não são processadas (mantém o último valor calculado, se houver).

---

## Pendências deste módulo — resolvidas em 2026-05-28

Todas resolvidas (CONT-P-01 a CONT-P-04: fórmula de dificuldade, versionamento de perguntas, hierarquia entre admins, reset parcial vs. total de estatísticas) — respostas completas em [`../requisitos.md`](../requisitos.md) § Questões em aberto → Resolvidas.
