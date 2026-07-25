# Plano de implementação — Módulo 7 (Geração de Questões por IA)

> Ver [`ai-generation.md`](ai-generation.md) para os RFs (**o quê**); este documento é sobre **como** implementar — o fatiamento, a ordem, e as decisões técnicas tomadas ao longo do ciclo. Progresso e achados de cada fatia ficam registrados em [`../tarefas.md`](../tarefas.md); este arquivo é o roteiro, não o changelog.

**Status:** Fatias 0, 1, 2 e 3 concluídas (2026-07-24, 2026-07-24, 2026-07-25, 2026-07-25). Próximo passo: Fatia 4.

## Contexto

O `bomberquiz-api` tem os Módulos 1, 2, 3 e 5 completos (backend + frontend). Os Módulos 4 (Conteúdo Parceiro), 6 (Assinaturas) e 7 (Geração de Questões por IA) ainda não têm código. O Módulo 7 é **exclusivo para administradores** (ADR-0021, já bem documentado em `ai-generation.md`, `decisoes.md` e `api.md`), então sua implementação não interfere nos Módulos 4/6 nem depende deles. A especificação funcional já é madura e estável: 9 RFs (`AIGEN-RF-001` a `009`) em [`ai-generation.md`](ai-generation.md) e 6 ADRs (0021–0026) em [`../decisoes.md`](../decisoes.md). Este ciclo é sobre **como** implementar, não **o quê** — os RFs/ADRs não são redesenhados aqui.

O módulo tem 4 componentes genuinamente novos no codebase (nenhum precedente para copiar): cliente da API Anthropic, adapter de storage Cloudflare R2, parsing de texto de PDF, e uso de `SELECT ... FOR UPDATE SKIP LOCKED` no Postgres. Todo o resto (CRUD, RBAC, audit log, paginação, scheduler) segue padrões já estabelecidos nos módulos existentes (referência principal: módulo de perguntas em `domain/content/`, `application/content/`, `infra/persistence/drizzle/repositories/question.repository.ts`, `http/routes/admin-questions.routes.ts`).

**Decisão tomada em 2026-07-24:** atualizar a ADR-0022 e o modelo padrão de `claude-sonnet-4-6` para **`claude-sonnet-5`** (mais barato e de qualidade superior na data) — feito como parte da Fatia 0.

## Fatiamento (6 fatias sequenciais, risco decrescente)

| # | Fatia | RFs | Status |
|---|---|---|---|
| 0 | Infra pura greenfield (sem rota HTTP) | base | ✅ concluída 2026-07-24 |
| 1 | Criar job | AIGEN-RF-001 | ✅ concluída 2026-07-24 |
| 2 | Consultar/listar jobs | AIGEN-RF-002/003 | ✅ concluída 2026-07-25 |
| 3 | Worker assíncrono (núcleo de negócio) | AIGEN-RF-004 | ✅ concluída 2026-07-25 |
| 4 | Revisão de questões (editar/aprovar/descartar) | AIGEN-RF-005 a 008 | próximo passo |
| 5 (desejável) | Excluir job | AIGEN-RF-009 | pendente |

Os 4 componentes greenfield ficam isolados na Fatia 0, sem nenhuma rota HTTP — se alguma integração externa se mostrar instável, isso não trava as fatias de negócio. Mesmo raciocínio de risco já usado no Módulo 5 (scheduler isolado antes das fatias de fluxo).

### Fatia 0 — Infra pura de alto risco ✅

- **Schema** (`api/src/infra/persistence/drizzle/schema.ts`): enums (`ai_generation_job_status`, `ai_review_status`) e tabelas `ai_reference_exams`, `ai_generation_jobs`, `ai_generated_questions`, com índices para scan do worker (`status`) e histórico (`created_by, created_at`). Migration `0007_cynical_tyger_tiger.sql` via `bun run db:generate`.
- **PDF**: `api/src/infra/pdf/pdf-signature.ts` (magic bytes `%PDF-`) + `api/src/infra/pdf/pdf-text-extractor.ts` (porta `IPdfTextExtractorPort` + implementação com **`unpdf`** — escolhida por rodar bem em runtimes tipo Bun/edge, sem dependências nativas de Node).
- **LLM**: `api/src/infra/llm/anthropic.adapter.ts` — porta `IQuestionGenerationLlmPort` + adapter usando `@anthropic-ai/sdk`, modelo `claude-sonnet-5`, com `cache_control: { type: "ephemeral" }` no bloco de sistema+exemplos. Geração de JSON estruturado via **tool use forçado** (`tool_choice` apontando para `generate_questions`).
  - **Achado importante**: `strict: true` da Anthropic não suporta `minItems`/`maxItems` diferentes de 0/1 em array, nem `minimum`/`maximum` em `integer` — confirmado via erro 400 real. Workaround: regra de "exatamente 4 alternativas" foi para a `description` (reforçada em validação de domínio na Fatia 3); `correct_index` usa `enum: [0,1,2,3]`.
- **Storage**: `api/src/infra/storage/r2.adapter.ts` — porta `IJobFileStoragePort` + adapter usando **`aws4fetch`** (SigV4 para R2 em ~3KB, sem trazer `@aws-sdk/client-s3` inteiro). `delete()` best-effort (nunca lança).
- **Config**: novas vars em `api/src/config/env.ts` — `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL_AI_GENERATION` (default `claude-sonnet-5`), `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `AI_GENERATION_JOB_CRON`, `AI_GENERATION_TIMEOUT_JOB_CRON`, `AI_GENERATION_MAX_CONCURRENT_JOBS` (default `2`), `AI_GENERATION_MATERIAL_CHUNK_TOKEN_LIMIT` (default `80000`).
- **`SELECT ... FOR UPDATE SKIP LOCKED`**: `drizzle-orm@0.45.2` suporta nativamente via `.for("update", { skipLocked: true })` — sem `sql\`\`` raw. `claimNextPendingJob()` no repositório do job, provado com um e2e de concorrência real (2 conexões Postgres independentes, não o client de teste `max:1`).
- **Testes**: `mock-anthropic.ts` e `mock-r2.ts` (interceptam `fetch` por hostname); `resetDatabase()` com as 3 novas tabelas; testes unitários/integração para os 4 componentes greenfield; smoke test manual com credenciais reais (achou e corrigiu os 2 bugs de schema acima, e um problema de permissão do token R2 — não era bug de código).

Detalhes completos e achados: ver entrada de 2026-07-24 ("Fatia 0") em [`../tarefas.md`](../tarefas.md).

### Fatia 1 — Criar job (AIGEN-RF-001) ✅

- `domain/ai-generation/ai-generation-job.entity.ts`, `ai-generation.errors.ts` (`AiGenerationJobNotFoundError`, `PdfCorruptedError`, `PdfHasNoExtractableTextError` — reservado para a Fatia 3, `DailyJobLimitExceededError`, `FileTooLargeError`, `InvalidPdfContentTypeError`, `InvalidQuestionCountError`), `ai-generation-job.repository.port.ts`.
- `infra/persistence/drizzle/repositories/ai-generation-job.repository.ts` completo (`create`, `findById`, `countCreatedByAdminOnDate`, `countByStatuses`), mantendo `claimNextPendingJob()` à parte da porta (uso exclusivo do worker, Fatia 3).
- `application/ai-generation/create-ai-generation-job.usecase.ts`: valida matéria ativa, assinatura/corrupção dos 2 PDFs, tamanho ≤20MB, limite diário (10/admin/dia, fuso `America/Sao_Paulo`), sobe os 2 PDFs ao R2 antes de persistir, cria job `pending`, grava `audit_log`. Calcula `queue_position`/`estimated_wait_seconds` via heurística (documentada em código — não é invariante de negócio, a refinar com dados reais de duração na Fatia 3).
- `http/schemas/ai-generation.schemas.ts` + `http/routes/admin-ai-generation.routes.ts` — rota multipart plana (`.post(...)`, não `.openapi()`, mesmo padrão de `POST /admin/questions/:id/image`). RBAC: `.use("/admin/ai-generation/*", requireAuth, requireRole("admin"))`.
- `http/app.ts`: `app.route("/", adminAiGenerationRoutes)`.

Detalhes completos e achados: ver entrada de 2026-07-24 ("Fatia 1") em [`../tarefas.md`](../tarefas.md).

### Fatia 2 — Consultar status + histórico (AIGEN-RF-002/003) ✅

- `domain/ai-generation/ai-generated-question.entity.ts` (entidade completa, sem invariante de construtor nesta fatia) + `ai-generated-question.repository.port.ts` + `DrizzleAiGeneratedQuestionRepository`.
- `ai-generation-job.repository.port.ts`: `countPendingCreatedBefore(createdAt)` + `list(filters)` paginado (join `subjects`).
- `application/ai-generation/calculate-queue-position.ts` — helper puro extraído da fórmula da Fatia 1, reaproveitado por `create-ai-generation-job.usecase.ts` (refatorado) e pelo `get-ai-generation-job.usecase.ts` novo.
- `application/ai-generation/get-ai-generation-job.usecase.ts` e `list-ai-generation-jobs.usecase.ts`.
- Rotas `GET /admin/ai-generation/jobs/:id` e `GET /admin/ai-generation/jobs` via `.openapi()`.
- Testes: como o worker ainda não existe, jobs `completed`/`failed` com questões são semeados **direto via repositório** (mesmo artifício já usado no Módulo 3 para simular submissões de parceiro antes do Módulo 4 existir).

Detalhes completos e achados: ver entrada de 2026-07-25 ("Fatia 2") em [`../tarefas.md`](../tarefas.md).

### Fatia 3 — Worker assíncrono (núcleo de negócio, AIGEN-RF-004) ✅

- `ai-reference-exam.repository.port.ts` + `DrizzleAiReferenceExamRepository` (cache por SHA-256, ADR-0025) — sem entidade de domínio, mesma razão de `audit_log` (sem ciclo de vida).
- Funções puras em `application/ai-generation/` (mesmo padrão de `calculate-queue-position.ts` da Fatia 2, não em `domain/`): `estimate-token-count.ts`, `identify-reference-questions.ts`, `split-material-into-chunks.ts` (empacotamento guloso por parágrafo), `distribute-question-count.ts`, `parse-generated-questions.ts` (validação semântica pós tool-use — a checagem de "exatamente 4 alternativas" que o schema estrito da Anthropic não consegue expressar, ver Fatia 0), `build-system-prompt.ts`.
- `application/ai-generation/process-ai-generation-job.usecase.ts` — o "worker use case", mesmo papel de `ExpireAndAbandonQuizzesUseCase`: reivindica jobs via `claimNextPendingJob()` (agora parte da porta) em transação **curta**, processa fora de transação (chamadas de rede de 30-90s nunca devem segurar lock de banco), extrai texto, resolve cache de exemplos, chama o LLM com prompt caching, persiste questões, finaliza job, remove PDFs do R2 em qualquer desfecho.
- `application/ai-generation/timeout-ai-generation-jobs.usecase.ts` — job de manutenção (CA-10), limiar de 5 min como constante fixa de domínio (não env var).
- Ajuste cirúrgico em `anthropic.adapter.ts`: `questionCount` (recebido desde a Fatia 0 mas nunca usado) agora entra num bloco de texto separado, fora do `system` cacheado.
- Registro de 2 novos jobs no scheduler existente (`index.ts`), reutilizando `infra/scheduler/scheduler.ts` tal como está — **sem infraestrutura de fila nova** (confirma ADR-0023).
- **Nota de concorrência:** em deploy single-machine (premissa atual do projeto, `fly.toml` com 1 máquina), o `protect: true` do croner já evita sobreposição; o `SKIP LOCKED` garante que nenhum job é processado 2×. O tick reivindica até `AI_GENERATION_MAX_CONCURRENT_JOBS` jobs e processa todos concorrentemente via `Promise.allSettled` — o próximo tick só começa depois que o lote termina. Se o app escalar horizontalmente no futuro, o controle de "máx. 2 simultâneos" via contagem vira best-effort — aceitável por ser controle de custo, não invariante de correção.

Detalhes completos, achados (incl. um bug real de timestamp corrigido durante o smoke test) e resultado do smoke manual: ver entrada de 2026-07-25 ("Fatia 3") em [`../tarefas.md`](../tarefas.md).

### Fatia 4 — Revisão de questões (AIGEN-RF-005 a 008)

- `update-ai-generated-question.usecase.ts`, `approve-ai-generated-question.usecase.ts` (reaproveita `IQuestionRepository` do Módulo 3 para criar o registro `published` em `questions`), `discard-ai-generated-question.usecase.ts`, `approve-all-.../discard-all-...usecase.ts`.
- Teste e2e do fluxo completo: criar job → processar (mock) → editar → aprovar individual → descartar → lote.

### Fatia 5 (desejável) — Excluir job (AIGEN-RF-009)

- `delete-ai-generation-job.usecase.ts` + rota `DELETE`. Prioridade "Desejável" no RF — pode ficar para depois sem bloquear nada.

## Arquivos críticos

- `api/src/infra/persistence/drizzle/schema.ts`
- `api/src/infra/llm/anthropic.adapter.ts`
- `api/src/infra/storage/r2.adapter.ts`
- `api/src/infra/pdf/pdf-text-extractor.ts`
- `api/src/application/ai-generation/create-ai-generation-job.usecase.ts`
- `api/src/application/ai-generation/get-ai-generation-job.usecase.ts`
- `api/src/application/ai-generation/list-ai-generation-jobs.usecase.ts`
- `api/src/application/ai-generation/process-ai-generation-job.usecase.ts`
- `api/src/application/ai-generation/timeout-ai-generation-jobs.usecase.ts`
- `api/src/infra/persistence/drizzle/repositories/ai-generation-job.repository.ts`
- `api/src/infra/persistence/drizzle/repositories/ai-generated-question.repository.ts`
- `api/src/infra/persistence/drizzle/repositories/ai-reference-exam.repository.ts`
- `api/src/config/env.ts`
- `../decisoes.md` (ADR-0022 revisada)

## Verificação

Por fatia, na ordem:
1. `bun run db:generate` (se houver mudança de schema) e revisar o SQL gerado manualmente antes de commitar.
2. `bun run test:db:up && bun run test:db:migrate`.
3. Rodar os testes da fatia recém-feita **individualmente** (com `timeout`), não via `bun run test` completo — há instabilidade conhecida de alguns arquivos de integração/e2e no Bun/Windows deste ambiente (ver Backlog em `../tarefas.md`).
4. Usar o skill `run-bomberquiz-api` (`api/.claude/skills/run-bomberquiz-api/SKILL.md`) para subir a API local e obter um admin de teste autenticado.
5. A partir da Fatia 3: smoke test manual ponta-a-ponta — `curl` multipart com 2 PDFs pequenos reais + `question_count` baixo para `POST /admin/ai-generation/jobs`, poll em `GET .../jobs/:id` até `completed`, conferir `prompt_tokens`/`completion_tokens`/`cached_tokens` e a qualidade das questões. **Único passo que gasta créditos reais da API Anthropic — evitar repetir sem necessidade.**
6. Conferir que os PDFs somem do R2 (bucket de teste, nunca o de produção) após o job terminar.
7. Atualizar `../tarefas.md` ao final de cada fatia não-trivial, registrando o que foi feito (convenção já seguida nos módulos anteriores), e atualizar a tabela de status no topo deste arquivo.
