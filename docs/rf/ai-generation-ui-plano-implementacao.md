# Plano de implementação — UI web do Módulo 7 (Geração de Questões por IA)

> Ver [`ai-generation.md`](ai-generation.md) para os RFs (**o quê**) e [`ai-generation-plano-implementacao.md`](ai-generation-plano-implementacao.md) para o backend, já 100% completo. Este documento é sobre **como** implementar a UI web (`bomberquiz-web`) que consome esse backend — o fatiamento, a ordem, e as decisões de design tomadas ao longo do ciclo. Progresso e achados de cada fatia ficam registrados em [`../tarefas.md`](../tarefas.md); este arquivo é o roteiro, não o changelog.

**Status:** Plano criado em 2026-07-25. Fatias 1, 2, 3 e 4 concluídas (2026-07-25, 2026-07-26).

## Contexto

O backend do Módulo 7 está 100% completo (Fatias 0-5): criar job (2 PDFs + matéria + quantidade) → acompanhar status (`pending`→`processing`→`completed`/`failed`) → revisar questões geradas (editar/aprovar/descartar, individual e em lote) → excluir job. Nenhuma tela existe ainda em `bomberquiz-web` para esse fluxo — é o único item pendente do módulo. Este plano fatia o desenvolvimento do frontend em 6 fatias verticais, mesmo espírito de risco decrescente/incremento demonstrável usado no fatiamento do backend, para que cada fatia seja implementável e verificável (inclusive visualmente, via `run-bomberquiz-web`) de forma independente.

## Descobertas da exploração (o que já existe vs. o que é território novo)

- **Rotas admin ficam sob `/painel/*`, não `/admin/*`** — o proxy do Vite dev server encaminha qualquer navegação começando com `/admin` direto pro backend (`vite.config.ts:49`), então uma rota React em `/admin/...` daria 404. Novas rotas: `/painel/geracao-ia`.
- **Cliente HTTP tipado** via `openapi-fetch` (`src/lib/api/client.ts`), gerado por `bun run openapi:generate` a partir de `/openapi.json` do backend rodando localmente. **As rotas do Módulo 7 ainda não estão em `schema.d.ts`** — regenerar é pré-requisito da Fatia 1.
- **`POST /admin/ai-generation/jobs` é multipart**, fora do `.openapi()` do backend (mesma razão do upload de imagem de pergunta) — precisa do padrão de `fetch` cru já usado em `imageRequest()` (`features/content/questions-api.ts:160-188`), não do cliente tipado.
- **Sem precedente de polling** no projeto (só um `setInterval` de countdown local, não-servidor). Vai ser a primeira vez usando `refetchInterval` do React Query — convenção nova, mas natural (backend já recomenda 3s enquanto `pending`/`processing`).
- **Sem precedente de wizard/dropzone/upload-progress.** O único upload existente é de 1 imagem, single-file, `image-upload-field.tsx` — bom template pra estender a 2 slots nomeados (prova + material), mas sem drag-and-drop nem barra de progresso real (nenhuma infra de progress tracking existe; segue o padrão atual: checagem client-side de tamanho/tipo + toast, "UX only", backend re-valida).
- **`review-queue-page.tsx`** (fila de revisão do parceiro) é o precedente mais próximo pra revisão questão-a-questão (tabela + aprovar/rejeitar + toast + invalidação de cache) — mas **não tem ação em lote** nem seleção múltipla; `approve-all`/`discard-all` serão UI nova (botão simples no cabeçalho, sem checkbox por item).
- **`question-form-dialog.tsx`** tem quase o mesmo shape de campos (statement/alternatives/correct_index/explanation/source_reference) — bom molde pra "fork" do dialog de edição de questão gerada, mas cada feature mantém seu próprio Zod schema espelhando os limites do backend (convenção confirmada: comentário `// Espelha RF-X`, sem pacote de validação compartilhado entre `features/content` e a nova `features/ai-generation`).
- **Sem `ConfirmDialog` compartilhado** — todo `AlertDialog` de confirmação é inline no próprio componente (delete pergunta, desativar conta, terminar quiz). Seguimos essa mesma convenção (não introduzir abstração nova fora do escopo pedido).
- **Sem precedente multi-step/wizard** — o mais próximo é `start-quiz-page.tsx` → `answer-quiz-page.tsx` (form page → navigate pra página de resultado). Adaptamos esse padrão, mas usando **job id como route param + refetch**, não `location.state` (que quebraria em refresh/deep-link — problema já documentado em `answer-quiz-page.tsx`).

## Decisões de design

1. **Passo 0 (pré-requisito, fora das 6 fatias): regenerar `schema.d.ts`** (`bun run openapi:generate` contra a API local rodando) — só assim o cliente tipado enxerga os endpoints JSON do módulo (list/get/patch/approve/discard/approve-all/discard-all/delete). `POST /jobs` (multipart) nunca aparece lá, segue via `fetch` cru mesmo depois da regeneração.
2. **3 rotas novas sob `/painel/geracao-ia`**, dentro do mesmo `RequireAdmin` → `PanelLayout` já usado pelas demais telas admin, + 1 item novo em `PANEL_NAV_ITEMS`:
   - `/painel/geracao-ia` — histórico (lista paginada de jobs).
   - `/painel/geracao-ia/novo` — formulário de criação (página dedicada, não dialog).
   - `/painel/geracao-ia/:jobId` — detalhe (progresso/erro/revisão, dependendo do status).
3. **Criação de job em página dedicada, não dialog** — desvio deliberado da convenção de CRUD simples (eixo/matéria usam dialog), justificado por ser um formulário maior (2 arquivos + matéria + contagem) que sempre termina navegando pra outro lugar (o job criado) — mesmo raciocínio do fluxo de quiz (`start-quiz-page.tsx`).
4. **Página de detalhe única, com 3 sub-visões por `status`** (não 3 rotas separadas): `pending`/`processing` → progresso com polling; `failed` → mensagem de erro + ação de excluir; `completed` → lista de revisão. Usa `jobId` da URL pra buscar via `GET /admin/ai-generation/jobs/:id`, nunca `location.state` — sobrevive a refresh/deep-link.
5. **Polling via `refetchInterval`** do React Query (novo no projeto): `refetchInterval: (query) => ["pending","processing"].includes(query.state.data?.status) ? 3000 : false` — para automaticamente ao chegar em `completed`/`failed`, conforme cadência já recomendada pelo próprio RF do backend.
6. **Revisão de questões geradas em tabela** (não cards — sem precedente de cards no projeto, e tabela já resolve bem o conteúdo), estrutura igual a `review-queue-page.tsx`: uma linha por questão, badge de `review_status`, ações "Editar"/"Aprovar"/"Descartar". Dialog de edição forkado de `question-form-dialog.tsx` com schema próprio (`features/ai-generation/schemas.ts`). Dialog de descarte forkado de `reject-question-dialog.tsx`, mas com motivo **opcional** (RF-007 não exige motivo, diferente da rejeição de parceiro). Ações em lote = 2 botões no cabeçalho da tabela ("Aprovar todas"/"Descartar todas"), cada um com `AlertDialog` de confirmação — sem seleção múltipla (sem precedente, sem necessidade: já é "todas as pendentes").
7. **Exclusão de job com fluxo de 2 tentativas**: 1ª chamada sem `confirm_discard_pending`; se a API responder 409 com a contagem de pendentes, abre um 2º `AlertDialog` explícito ("ainda há N questões pendentes — descartar tudo e excluir?") e reenvia com `confirm_discard_pending: true`. Ação disponível tanto na lista (ícone por linha) quanto na página de detalhe.
8. **Upload dos 2 PDFs**: componente novo (`ai-generation-pdf-picker.tsx`), estendendo o padrão de `image-upload-field.tsx` pra 2 slots nomeados (prova de referência / material de estudo), checagem client-side de `application/pdf` + ≤20MB por arquivo (mesmo espírito "UX only" já documentado no upload de imagem). Sem barra de progresso real (sem infra no projeto) — indicador simples de "enviando..." (spinner + texto), aceitável para arquivos desse tamanho.
9. **Toast (`sonner`) e invalidação de cache (`queryClient.invalidateQueries`)** seguem exatamente os padrões já existentes em `features/content/*-api.ts`.

## Estrutura de arquivos

```
src/features/ai-generation/
  ai-generation-api.ts    # hooks: useCreateAiGenerationJob (fetch cru multipart),
                          # useAiGenerationJob(id) (com polling), useAiGenerationJobs(filters),
                          # useUpdateGeneratedQuestion, useApproveGeneratedQuestion,
                          # useDiscardGeneratedQuestion, useApproveAllGenerated,
                          # useDiscardAllGenerated, useDeleteAiGenerationJob
  schemas.ts              # createJobFormSchema, editGeneratedQuestionFormSchema, discardReasonFormSchema

src/pages/admin/
  ai-generation-jobs-page.tsx          # histórico: lista + filtros (status/matéria) + paginação + excluir
  ai-generation-new-job-page.tsx       # formulário de criação
  ai-generation-job-detail-page.tsx    # branch por status: progresso / falha / revisão
  ai-generation-question-edit-dialog.tsx  # fork de question-form-dialog.tsx
  ai-generation-discard-dialog.tsx        # fork de reject-question-dialog.tsx (motivo opcional)
  ai-generation-pdf-picker.tsx            # extensão de image-upload-field.tsx (2 slots)
```

Rotas em `src/app/router.tsx` (dentro do bloco `RequireAdmin`/`PanelLayout` já existente) + entrada nova em `PANEL_NAV_ITEMS` (`src/components/panel-layout.tsx`).

## Fatiamento (6 fatias)

| # | Fatia | Entrega | Status |
|---|---|---|---|
| 1 | Fundação | Regenerar schema, 3 rotas + nav, histórico vazio (só prova guard/navegação) | ✅ concluída 2026-07-25 |
| 2 | Criar job | Formulário completo (matéria + contagem + 2 PDFs), upload multipart, navega pro detalhe | ✅ concluída 2026-07-25 |
| 3 | Acompanhar job | Polling (pending/processing) + visão de falha (failed) | ✅ concluída 2026-07-25 |
| 4 | Histórico completo | Filtros (status/matéria) + paginação + resumo por job na lista | ✅ concluída 2026-07-26 |
| 5 | Revisar questões | Visão "completed": tabela + editar/aprovar/descartar individual | pendente |
| 6 | Lote + excluir | Aprovar todas/descartar todas + exclusão de job (fluxo de confirmação em 2 passos) | pendente |

### Fatia 1 — Fundação ✅

- `bun run openapi:generate` (API local rodando) — `schema.d.ts` ganhou os 8 endpoints JSON do módulo (list/get/patch/approve/discard/approve-all/discard-all/delete; `delete` aparece na mesma entrada `/admin/ai-generation/jobs/{id}` do `get`). `POST /jobs` (multipart) não aparece, como esperado.
- `features/ai-generation/ai-generation-api.ts`: `useAiGenerationJobs(filters)` (`page`/`pageSize` fixos por enquanto — filtros de status/matéria e paginação de verdade ficam para a Fatia 4).
- 3 rotas (`/painel/geracao-ia`, `/painel/geracao-ia/novo`, `/painel/geracao-ia/:jobId`) + item "Geração por IA" em `PANEL_NAV_ITEMS`. `AiGenerationJobsPage` lista jobs reais (tabela: matéria/material/status/questões/criado em) com estado vazio "Nenhum job de geração ainda."; `/novo` e `/:jobId` são páginas placeholder ("Em construção") até as Fatias 2/3/5.
- **Desvio do plano original**: em vez de um teste de guard dedicado às 3 rotas novas (redundante — `RequireAdmin` é agnóstico de rota e já tem cobertura própria em `require-admin-guard.test.tsx`), foi escrito um teste de página para `AiGenerationJobsPage` (lista populada, estado vazio, link "Nova geração"), seguindo o padrão de `axes-page.test.tsx`; `panel-layout.test.tsx` foi atualizado para cobrir o 5º link de navegação.
- **Verificação**: `bun run typecheck` limpo. `bun run test`: 64 pass (+3 novos, +1 assert no teste de nav), zero regressão. Verificação visual via `run-bomberquiz-web` (Playwright): as 3 rotas renderizam corretamente (nav item ativo, lista vazia + botão, placeholders de `/novo` e `/:jobId`).

### Fatia 2 — Criar job (AIGEN-RF-001) ✅

- `ai-generation-pdf-picker.tsx`: componente único (`AiGenerationPdfPicker`), reaproveitado 2x na página (prova de referência / material de estudo) em vez de um componente combinado de "2 slots" — mais simples e ainda extensão direta de `image-upload-field.tsx` (mesma checagem client-side de tipo `application/pdf` + ≤20MB, "UX only").
- `features/ai-generation/schemas.ts`: `createJobFormSchema` (matéria obrigatória, `question_count` inteiro 1-30). Os 2 PDFs ficam fora do schema Zod — geridos como `File | null` em `useState`, mesmo padrão de `ImageUploadField`.
- `useCreateAiGenerationJob()` em `ai-generation-api.ts`: `fetch` cru multipart (mesmo padrão de `imageRequest` em `questions-api.ts`), já que `POST /admin/ai-generation/jobs` fica fora do `.openapi()` do backend.
- `AiGenerationNewJobPage`: formulário completo (Select de matéria só ativas, input de quantidade, 2 `AiGenerationPdfPicker`), submit bloqueado com toast se algum PDF faltar, `navigate("/painel/geracao-ia/:jobId")` com o `job_id` retornado (202).
- **Verificação**: `bun run typecheck` limpo. `bun run test`: 68 pass (+4 novos: bloqueio sem PDFs, criação com sucesso + payload multipart correto + navegação, erro do servidor via toast, rejeição de arquivo não-PDF antes do envio). **Smoke manual real** (com consentimento explícito do usuário sobre o custo): formulário preenchido e submetido de verdade via Playwright contra a API local rodando (worker incluso) — job criado, worker processou de verdade (`claude-sonnet-5`, 1436+427 tokens), 1 questão gerada a partir do material de estudo enviado, job apareceu como "Concluído" na lista. Job e questão removidos via `DELETE /admin/ai-generation/jobs/:id` ao final.

### Fatia 3 — Acompanhar job (AIGEN-RF-002) ✅

- `useAiGenerationJob(id)` em `ai-generation-api.ts`: `refetchInterval` condicional (3s enquanto `pending`/`processing`, para em `completed`/`failed`) — primeiro uso de polling no projeto.
- `useDeleteAiGenerationJob()` (adiantado desta fatia, não da 6): a visão de falha já precisa de uma ação de excluir funcional. Implementado com o fluxo de 2 tentativas completo (retry com `confirm_discard_pending: true` no 409) — a Fatia 6 só vai reaproveitar este hook na lista, sem precisar refazer nada.
- `AiGenerationJobDetailPage`: 3 sub-visões por status — `pending`/`processing` (spinner + posição na fila ou mensagem de progresso), `failed` (mensagem de erro real do backend + botão "Excluir job" com `AlertDialog` de confirmação, e um segundo `AlertDialog` só se houver pendentes), `completed` (resumo simples — a tabela de revisão fica pra Fatia 5). 404/erro de rede tratados com mensagem + link de volta ao histórico.
- **Verificação**: `bun run typecheck` limpo. `bun run test`: 75 pass (+7 novos: fila, processando, falha+exclusão, 409 com confirmação extra, concluído, 404, polling real via fake timers confirmando 2ª chamada após 3s). **Smoke manual real** (com consentimento explícito do usuário): job criado de verdade via Playwright, observado transicionando `Pendente` → `Processando` → `Concluído` na tela via polling real (sem reload), confirmado por screenshots em cada tick. Visão de falha também testada com um PDF sem texto extraível (sem custo de Anthropic, falha na extração) — mensagem de erro real exibida, exclusão via UI confirmada ponta a ponta (dialog → `DELETE` real → toast → navegação de volta ao histórico).

### Fatia 4 — Histórico completo (AIGEN-RF-003) ✅

- `AiGenerationJobsFilters` (`ai-generation-api.ts`) ganhou `status`/`subjectId` opcionais; `useAiGenerationJobs` agora passa `status`/`subject_id` na query (schema já tinha os 2 campos desde a regeneração da Fatia 1).
- `AiGenerationJobsPage`: filtro de status e de matéria (`<select>` nativo, sentinel `"all"`, mesmo padrão de `questions-page.tsx`; matéria via `useSubjects({status:"all", page:1, pageSize:100})`), estado de página com o componente `Pagination` compartilhado (Anterior/Próxima + "Página X de Y"), nova coluna "Revisão" com 3 badges (`{approved} aprovadas`/`{pending} pendentes`/`{discarded} descartadas`) quando `job.summary` existe, `—` quando `null` (job ainda `pending`/`processing`). `error_message` de jobs `failed` (AIGEN-RF-003 CA-5) exibido como texto pequeno mutado abaixo do nome do material.
- **Achado durante a verificação visual**: `summary` do backend não é `null` só para jobs `completed` — é calculado (podendo ser `{0,0,0}`) para qualquer job em `FINISHED_STATUSES` (`completed` **e** `failed`), já que a lógica olha para `ai_generated_questions` existentes, não para o status em si. Um job `failed` sem nenhuma questão gerada mostra corretamente "0 aprovadas/0 pendentes/0 descartadas" (não `—`) — comportamento correto do contrato, não um bug.
- **Verificação**: `bun run typecheck` limpo. `bun run test`: 81 pass (+9 na página de lista: listagem, estado vazio, botão "Nova geração", badges de resumo, `—` quando sem resumo, `error_message` em failed, filtro de status, filtro de matéria, paginação), zero regressão. **Verificação visual via `run-bomberquiz-web`** — sem custo de Anthropic, é só leitura: como o único job do banco de dev pertence a outro admin (histórico é escopado por `created_by`, AIGEN-RF-003 CA-4 — corretamente mostrou vazio para o admin de teste), foram inseridos 2 jobs sintéticos direto no Postgres de dev (1 `completed` com 2 aprovadas/1 pendente, 1 `failed` com `error_message`) só para exercitar a tela; confirmados visualmente os 2 filtros funcionando contra a API real (status=failed e matéria específica cada um reduzindo a 1 linha corretamente) e os badges de resumo corretos. Os 2 jobs sintéticos e suas questões foram removidos do banco ao final (por ID exato); o job pré-existente de outro admin não foi tocado.

### Fatia 5 — Revisar questões geradas (AIGEN-RF-005 a 007)

- Sub-visão "completed" de `AiGenerationJobDetailPage`: tabela de questões geradas.
- `ai-generation-question-edit-dialog.tsx` (editar), `ai-generation-discard-dialog.tsx` (descartar com motivo opcional), aprovar direto (sem dialog, como no fluxo de parceiro).
- Verificação manual: editar, aprovar (confirmar que a pergunta aparece em "Perguntas" como publicada) e descartar uma questão real.

### Fatia 6 — Ações em lote + excluir job (AIGEN-RF-008/009)

- Botões "Aprovar todas"/"Descartar todas" na visão de revisão, cada um com `AlertDialog`.
- Exclusão de job **na lista** (`AiGenerationJobsPage`): reaproveita `useDeleteAiGenerationJob()`, já implementado e testado na Fatia 3 (fluxo de 2 tentativas completo) — só falta o botão/ícone por linha e os 2 `AlertDialog`, sem lógica nova de rede.
- Verificação manual: aprovar/descartar em lote, excluir um job com pendentes (confirmando o segundo diálogo) e um sem pendentes (direto) a partir da lista.

## Arquivos críticos

- `web/src/lib/api/client.ts` / `web/src/lib/api/schema.d.ts`
- `web/src/app/router.tsx`
- `web/src/components/panel-layout.tsx`
- `web/src/features/content/questions-api.ts` (padrão de upload multipart a reaproveitar)
- `web/src/pages/admin/review-queue-page.tsx` / `question-form-dialog.tsx` / `reject-question-dialog.tsx` / `image-upload-field.tsx` (moldes)
- `web/src/features/ai-generation/ai-generation-api.ts` (novo)
- `web/src/pages/admin/ai-generation-*.tsx` (novos)

## Verificação (por fatia)

1. `bun run typecheck` limpo.
2. Testes unitários (Vitest + RTL) por página/hook novo, seguindo o padrão de mock existente (`vi.mock("@/lib/api/client")`, `vi.mock("sonner")`, `QueryClient` + `MemoryRouter` de teste) — ver `tests/unit/axes-page.test.tsx` e `tests/unit/review-queue-page.test.tsx` como moldes.
3. Verificação visual real via skill `run-bomberquiz-web` (Playwright headless) ao final de cada fatia de risco (2, 3, 5, 6) — não basta typecheck/vitest passando para features de UI, por convenção já estabelecida.
4. Regressão: suíte de testes existente do frontend continua verde.
5. Ao final da Fatia 6 (módulo 100% completo, frontend + backend): atualizar `requisitos/docs/tarefas.md` (nova entrada) e a linha de backlog "UI web do Módulo 7".
