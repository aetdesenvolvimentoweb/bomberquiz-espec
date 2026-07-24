# Módulo 7 — Geração de Questões por IA

> Identificador dos RFs deste módulo: `AIGEN-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- Funcionalidade **exclusiva para admins** — garante controle de custos da API LLM e responsabilidade editorial sobre o conteúdo gerado (ADR-0021).
- Aceita apenas PDFs com **texto selecionável** (nativos). PDFs digitalizados/escaneados não são processados — o job falha com mensagem clara (ADR-0021).
- O processamento é **assíncrono**: a criação do job retorna imediatamente com `HTTP 202`; o admin acompanha o progresso via polling ou notificação in-app (AIGEN-RF-002).
- Questões geradas seguem **exatamente** a mesma estrutura definida em ADR-0016 (4 alternativas, 1 correta, justificativa obrigatória, referência opcional, imagem opcional). Nenhuma regra de conteúdo é relaxada por ser gerada por IA.
- Quando aprovadas, as questões vão diretamente para `status=published` (privilégio de admin, mesmo princípio de CONT-RF-010) — sem fila de revisão adicional.
- Todo job e toda aprovação/descarte registram entrada em `audit_log` (schema em `arquitetura.md` § Audit log). Listagens paginadas conforme `api.md` § Paginação, ordenação e filtros.
- Acesso de não-admin → HTTP 403.

## Regras gerais

| Regra | Valor |
|---|---|
| Questões por job | mínimo 1, máximo 30 |
| Tamanho máximo por PDF | 20 MB |
| Formato aceito | `application/pdf`; texto selecionável (MIME + verificação de extração real) |
| Jobs em processamento simultâneo (global) | máximo 2 (fila FIFO se excedido) |
| Jobs criados por admin por dia | máximo 10 |
| Retenção dos PDFs no R2 | excluídos automaticamente ao job atingir `completed` ou `failed` (ADR-0024) |
| Tempo máximo de processamento | 5 minutos por job; após isso → `failed` com `timeout` |
| Modelo LLM padrão | `claude-sonnet-5` (ADR-0022, revisado 2026-07-24) |

## Fluxo geral

```
Admin cria job (PDFs + config)
    │
    ▼
Job nasce em status=pending
    │
    ▼
Scheduler/worker (AIGEN-RF-004) assume o job
    ├── Extrai texto dos dois PDFs
    ├── Identifica questões de referência (prova de exemplo)
    ├── Monta prompt com: exemplos + material + instruções de formato
    ├── Chama a API Claude (ADR-0022)
    └── Parseia resposta JSON → grava ai_generated_questions
    │
    ▼
Job → status=completed (ou failed se erro)
    │
    ▼
Admin revisa questão a questão (edita, aprova ou descarta)
    │
    ▼
Aprovadas → questions (published) no banco principal
Descartadas → ficam registradas no job (audit)
```

## Modelo de dados

### `ai_reference_exams`

Tabela de cache: armazena exemplos de questões extraídos de cada arquivo de prova de referência, indexados pelo hash do conteúdo. Permite reutilização entre jobs sem reprocessar o mesmo PDF (ADR-0025).

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | uuid | PK |
| `file_hash` | string | SHA-256 do conteúdo do PDF (chave de cache — unique) |
| `extracted_questions` | jsonb | Array de até 8 questões-exemplo extraídas para few-shot |
| `source_preview` | text | Primeiros 500 caracteres do PDF (identificação/debug) |
| `created_at` | timestamptz | — |
| `last_used_at` | timestamptz | Atualizado a cada reutilização (para purga futura de entradas stale) |

### `ai_generation_jobs`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | uuid | PK |
| `created_by` | FK users | Admin que criou o job |
| `subject_id` | FK subjects | Matéria de destino das questões |
| `status` | enum | `pending` · `processing` · `completed` · `failed` |
| `question_count_requested` | integer | Quantidade pedida pelo admin |
| `question_count_generated` | integer | Quantidade retornada pelo LLM (pode diferir) |
| `reference_pdf_key` | string | Chave R2 do PDF da prova de referência (excluído pós-job) |
| `material_pdf_key` | string | Chave R2 do PDF do material de estudo (excluído pós-job) |
| `reference_exam_id` | FK ai_reference_exams? | Cache de exemplos utilizado (null se cache miss sem questões reconhecíveis) |
| `reference_text_preview` | text | Primeiros 500 caracteres extraídos da prova (log/debug) |
| `material_name` | string | Nome do arquivo do material (exibido na UI) |
| `model_used` | string | ID do modelo LLM utilizado |
| `prompt_tokens` | integer | Tokens de entrada consumidos (auditoria de custo) |
| `completion_tokens` | integer | Tokens de saída consumidos |
| `cached_tokens` | integer | Tokens lidos do cache de prompt (ADR-0026; 0 se sem cache hit) |
| `error_message` | text? | Preenchido somente em `failed` |
| `created_at` | timestamptz | — |
| `processing_started_at` | timestamptz? | Quando o worker assumiu |
| `completed_at` | timestamptz? | Quando finalizou (sucesso ou falha) |

### `ai_generated_questions`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | uuid | PK |
| `job_id` | FK ai_generation_jobs | — |
| `generation_index` | integer | Posição na resposta do LLM (1-based) |
| `review_status` | enum | `pending` · `approved` · `discarded` |
| `question_data` | jsonb | Estrutura completa: `{ statement, alternatives[4], correct_index, explanation, source_reference? }` |
| `edited` | boolean | `true` se o admin alterou algum campo antes de aprovar |
| `approved_question_id` | FK questions? | Preenchido ao aprovar (referencia o registro criado) |
| `reviewed_by` | FK users? | Admin que aprovou/descartou |
| `reviewed_at` | timestamptz? | — |
| `created_at` | timestamptz | — |

---

## AIGEN-RF-001 — Criar job de geração

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Sessão ativa com `role=admin`.

**Descrição:**
Admin inicia um job de geração enviando os dois PDFs e configurando os parâmetros. O job é enfileirado e processado de forma assíncrona.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/ai-generation/jobs` com `multipart/form-data` contendo:
  - `reference_pdf` — PDF da prova de referência do TAP (obrigatório).
  - `material_pdf` — PDF do material de estudo (obrigatório).
  - `subject_id` — UUID da matéria de destino (obrigatório; deve existir e estar `active`).
  - `question_count` — inteiro entre 1 e 30 (obrigatório).
- **CA-2:** Validações antes de aceitar:
  - `Content-Type` de cada arquivo deve ser `application/pdf`.
  - Tamanho ≤ 20 MB por arquivo. Acima → E-3.
  - Arquivo não pode estar corrompido (tentativa de leitura básica da estrutura PDF). Falha → E-4.
  - `subject_id` deve apontar para matéria `active`. Arquivada → E-5.
- **CA-3:** Ambos os PDFs são enviados ao R2 em bucket privado (`ai-generation/` prefix) com chave `{job_id}/reference.pdf` e `{job_id}/material.pdf`. Em sucesso, job nasce com `status=pending`.
- **CA-4:** Resposta: HTTP 202 com `{ job_id, status: "pending", estimated_wait_seconds }`. Frontend usa `job_id` para polling (AIGEN-RF-002).
- **CA-5:** Limite diário: 10 jobs criados pelo mesmo admin no dia corrente (fuso `America/Sao_Paulo`). Excedido → E-6.
- **CA-6:** Limite global: se já houver 2 jobs em `processing`, o novo job entra em `pending` e aguarda na fila FIFO. Não é erro — o admin é informado da posição na fila no campo `queue_position` da resposta.
- **CA-7:** Entrada em `audit_log` com `action=create_ai_generation_job`, `target_type=ai_generation_job`, registrando `subject_id`, `question_count_requested`, nomes dos arquivos.

**Erros previstos:**
- **E-1:** Acesso por não-admin → HTTP 403.
- **E-2:** Campo obrigatório ausente → HTTP 422.
- **E-3:** Arquivo > 20 MB → HTTP 413 com indicação do arquivo.
- **E-4:** PDF ilegível ou corrompido → HTTP 422, `"O arquivo {name} não é um PDF válido ou está corrompido."`.
- **E-5:** Matéria arquivada ou inexistente → HTTP 422.
- **E-6:** Limite diário de 10 jobs excedido → HTTP 429, `"Limite de 10 jobs por dia atingido. Tente novamente amanhã."`.

---

## AIGEN-RF-002 — Consultar status e questões geradas (polling)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Retorna o estado atual do job e, quando concluído, a lista de questões geradas com seu status de revisão.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/ai-generation/jobs/:id` retorna:
  ```json
  {
    "id": "...",
    "status": "pending|processing|completed|failed",
    "subject_id": "...",
    "subject_name": "...",
    "material_name": "...",
    "question_count_requested": 20,
    "question_count_generated": 18,
    "queue_position": 0,
    "model_used": "claude-sonnet-5",
    "prompt_tokens": 42000,
    "completion_tokens": 3200,
    "error_message": null,
    "created_at": "...",
    "completed_at": "...",
    "questions": [
      {
        "id": "...",
        "generation_index": 1,
        "review_status": "pending|approved|discarded",
        "question_data": { "statement": "...", "alternatives": ["..."], "correct_index": 0, "explanation": "...", "source_reference": "..." },
        "edited": false,
        "approved_question_id": null,
        "reviewed_at": null
      }
    ],
    "summary": { "pending": 15, "approved": 2, "discarded": 1 }
  }
  ```
- **CA-2:** Campo `questions` só é populado quando `status=completed`. Em `pending` ou `processing`, retorna `questions: []` e `summary: null`.
- **CA-3:** Em `status=failed`, `questions` pode conter resultado parcial se o LLM retornou algo antes de falhar (casos raros). `error_message` explica o motivo.
- **CA-4:** `queue_position` = 0 quando o job está em processamento ou concluído; ≥ 1 quando ainda aguarda na fila (AIGEN-RF-001 CA-6).
- **CA-5:** Endpoint leve — alvo P95 < 200ms. Frontned pode fazer polling a cada 3 s enquanto `status ∈ {pending, processing}`.
- **CA-6:** Administrador acessa apenas jobs criados por ele mesmo. Job de outro admin → HTTP 404 (não vaza existência).

**Erros previstos:**
- **E-1:** Job inexistente ou de outro admin → HTTP 404.

---

## AIGEN-RF-003 — Listar histórico de jobs

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Listagem paginada dos jobs criados pelo admin, com estado e contadores de revisão, para acompanhamento e auditoria.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/ai-generation/jobs` retorna `{ id, status, subject_name, material_name, question_count_requested, question_count_generated, summary: {pending, approved, discarded}, created_at, completed_at }`, paginado.
- **CA-2:** Filtros: `status` (multi-select), `subject_id`.
- **CA-3:** Ordenação padrão: `created_at DESC`.
- **CA-4:** Admin vê apenas os próprios jobs. Lista de outro admin → HTTP 403 (não há endpoint cross-admin nesta versão).
- **CA-5:** Jobs com `status=failed` permanecem no histórico (para diagnóstico) e exibem `error_message`.

---

## AIGEN-RF-004 — Processamento interno do job (worker assíncrono)

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (worker interno, não endpoint).

**Descrição:**
Worker que processa jobs da fila. Extrai texto dos PDFs, identifica exemplos na prova de referência, monta o prompt e chama o LLM, e persiste as questões geradas.

**Critérios de aceitação:**
- **CA-1:** Worker monitora a fila de jobs `status=pending` em FIFO (`created_at ASC`). Respeita o limite de 2 simultâneos (global). Usa `SELECT ... FOR UPDATE SKIP LOCKED` no Postgres para evitar duplo-processamento.
- **CA-2:** Ao assumir um job, transiciona para `status=processing`, preenche `processing_started_at`. Se já decorreu > 5 minutos desde `processing_started_at` sem conclusão, outro worker pode retomar (idempotência por lock).
- **CA-3:** **Extração de texto:**
  - Lê o PDF da prova de referência do R2 e extrai texto via biblioteca de parsing (texto selecionável).
  - Lê o PDF do material do R2 e extrai o texto completo.
  - Se qualquer PDF não tiver texto extraível (zero caracteres após extração) → job vai para `failed` com `error_message="PDF sem texto selecionável. Use arquivos com texto nativo, não digitalizados."`. PDFs são excluídos do R2.
- **CA-4:** **Identificação de exemplos na prova de referência (com cache — ADR-0025):**
  - Computa o SHA-256 do conteúdo do arquivo de prova de referência baixado do R2.
  - Verifica se existe registro em `ai_reference_exams` com o mesmo `file_hash`. Se sim (cache hit): usa os `extracted_questions` armazenados diretamente, atualiza `last_used_at`, preenche `ai_generation_jobs.reference_exam_id`. Zero custo de reprocessamento.
  - Se não existe (cache miss): extrai da prova até **8 questões completas** (enunciado + alternativas + gabarito quando disponível) usando heurística por padrões de numeração comuns em provas de concurso (`1.`, `1)`, `QUESTÃO 01`, etc.). Se ao menos 1 questão for reconhecida, persiste em `ai_reference_exams` para reutilização futura.
  - Se nenhuma questão reconhecível for encontrada (cache miss sem resultado): usa os primeiros 3.000 caracteres da prova como contexto de estilo (sem few-shot explícito) e continua. Não persiste em `ai_reference_exams` (sem exemplos estruturados para cachear).
- **CA-5:** **Montagem do prompt e prompt caching (ADR-0026):**
  - System prompt instrui o modelo a gerar questões em **português brasileiro**, no formato exato definido por ADR-0016 (4 alternativas, 1 correta, justificativa fundamentada no material, referência opcional).
  - Inclui os exemplos extraídos/cacheados da prova de referência como demonstração de estilo e complexidade.
  - O bloco `[system prompt + exemplos da prova]` é marcado com `cache_control: { type: "ephemeral" }` na chamada à API Claude. Em jobs multi-chunk (CA-6), as chamadas subsequentes pagam ~10% do custo normal desse bloco (TTL de cache 5 min). O campo `cached_tokens` do job registra os tokens lidos do cache (via `usage.cache_read_input_tokens` da resposta).
  - Inclui o texto completo do material (ou porção relevante — CA-6) como bloco separado (não cacheado, varia por chunk).
  - Instrui: cobrir diferentes seções do material, não concentrar questões em uma única parte, variação de dificuldade, alternativas plausíveis mas inequívocas, enunciado claro e direto.
  - Solicita saída em JSON estritamente estruturado (schema OpenAPI-like para garantir parsing).
- **CA-6:** **Limite de contexto:** se o texto do material ultrapassar ~80.000 tokens estimados (~60.000 palavras), o worker divide o material em blocos proporcionais e distribui o `question_count` entre os blocos (geração em múltiplas chamadas). O campo `material_name` na resposta indica que houve divisão com sufixo `" (processado em N partes)"`. Para a grande maioria dos manuais operacionais do CBMGO (geralmente < 200 páginas de texto), uma única chamada é suficiente.
- **CA-7:** **Parseamento da resposta LLM:**
  - Valida que cada questão retornada tem: `statement`, `alternatives` com exatamente 4 strings não-vazias, `correct_index` ∈ {0,1,2,3}, `explanation` não-vazia.
  - Questões malformadas são descartadas silenciosamente (não quebram o job). `question_count_generated` reflete apenas as válidas.
  - Se zero questões válidas retornadas → job vai para `failed` com `error_message="O modelo não retornou questões válidas. Tente com um material diferente ou quantidade menor."`.
- **CA-8:** Em sucesso: grava todas as `ai_generated_questions` com `review_status=pending`, transiciona job para `status=completed`, preenche `completed_at`, `model_used`, `prompt_tokens`, `completion_tokens`. **Exclui os dois PDFs do R2** (ADR-0024). Notificação in-app para o admin (badge ou alerta na UI — canal a definir junto ao sistema de notificações).
- **CA-9:** Em qualquer exceção não recuperável: transiciona para `status=failed`, preenche `error_message` com resumo do erro (sem stack trace), exclui os PDFs do R2, registra no `audit_log` com `action=ai_generation_failed`.
- **CA-10:** Timeout: se `processing_started_at` > 5 minutos e job ainda em `processing`, um job de manutenção (parte do scheduler — ADR-0017) transiciona para `failed` com `"Tempo de processamento excedido."`.

**Erros previstos:** tratados internamente (ver CA-3, CA-7, CA-9, CA-10).

---

## AIGEN-RF-005 — Editar questão gerada antes da aprovação

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Job em `status=completed`; questão em `review_status=pending`.

**Descrição:**
Admin ajusta o conteúdo de uma questão gerada antes de aprová-la — corrigindo enunciado, alternativas, gabarito ou justificativa.

**Critérios de aceitação:**
- **CA-1:** `PATCH /admin/ai-generation/jobs/:job_id/questions/:qid` com `{ statement?, alternatives?, correct_index?, explanation?, source_reference? }`. Mesmas validações de tamanho/formato de CONT-RF-010 (regras gerais do Módulo 3).
- **CA-2:** Aplica edição no campo `question_data` (JSONB atualizado). Marca `edited=true`.
- **CA-3:** Questão já em `approved` ou `discarded` → E-2.
- **CA-4:** Nenhuma entrada em `audit_log` para edições intermediárias (a aprovação final registra o estado definitivo). Isso mantém o log limpo sem ruído de rascunho.

**Erros previstos:**
- **E-1:** Job ou questão inexistente / de outro admin → HTTP 404.
- **E-2:** Questão já aprovada ou descartada → HTTP 409, `"Esta questão já foi revisada."`.
- **E-3:** Validação de conteúdo falha → HTTP 422 com campos.

---

## AIGEN-RF-006 — Aprovar questão gerada individualmente

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Job em `status=completed`; questão em `review_status=pending`.

**Descrição:**
Admin aprova uma questão gerada, que é então inserida no banco de questões principal como `published`.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/ai-generation/jobs/:job_id/questions/:qid/approve`.
- **CA-2:** Cria um registro em `questions` com:
  - `subject_id` = o da matéria do job.
  - `statement`, `alternatives`, `correct_index`, `explanation`, `source_reference?` = dados de `question_data` (após edições do AIGEN-RF-005, se houver).
  - `status = published`, `author_id` = admin que aprovou.
  - `published_at = now()`.
  - Sem imagem (campo `image_url` não é gerado pelo LLM no MVP; admin pode adicionar depois via CONT-RF-013).
- **CA-3:** Atualiza `ai_generated_questions`: `review_status=approved`, `approved_question_id=<novo id>`, `reviewed_by`, `reviewed_at`.
- **CA-4:** Entrada no `audit_log` com `action=approve_ai_question`, `target_type=question`, `target_id=<novo question id>`, `payload_summary` com `job_id` e `generation_index` (rastreabilidade).
- **CA-5:** Questão já aprovada ou descartada → E-2.

**Erros previstos:**
- **E-1:** Job ou questão inexistente → HTTP 404.
- **E-2:** `review_status ≠ pending` → HTTP 409.
- **E-3:** Job `status ≠ completed` → HTTP 409, `"O job ainda não foi concluído."`.

---

## AIGEN-RF-007 — Descartar questão gerada individualmente

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Job em `status=completed`; questão em `review_status=pending`.

**Descrição:**
Admin rejeita uma questão gerada. A questão permanece registrada no job (para auditoria) mas não entra no banco principal.

**Critérios de aceitação:**
- **CA-1:** `POST /admin/ai-generation/jobs/:job_id/questions/:qid/discard` com `{ reason? }` (texto livre, até 500 caracteres, opcional). O motivo é registrado no `audit_log`.
- **CA-2:** Atualiza `ai_generated_questions`: `review_status=discarded`, `reviewed_by`, `reviewed_at`.
- **CA-3:** Entrada no `audit_log` com `action=discard_ai_question` e `reason` (se fornecido).
- **CA-4:** Questão já aprovada → E-2 (não é possível desfazer aprovação por aqui — questão já existe em `questions`; remoção segue CONT-RF-012).
- **CA-5:** Questão já descartada → E-2.

**Erros previstos:**
- **E-1:** Job ou questão inexistente → HTTP 404.
- **E-2:** `review_status ≠ pending` → HTTP 409.

---

## AIGEN-RF-008 — Aprovação ou descarte em lote

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Admin aprova ou descarta todas as questões ainda `pending` de um job de uma vez, agilizando o fluxo quando o resultado é de alta qualidade (aprovar tudo) ou inaceitável (descartar tudo).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/ai-generation/jobs/:job_id/approve-all` — processa todas as questões `pending` do job aplicando a lógica de AIGEN-RF-006 em cada uma, em transação única.
- **CA-2:** `POST /admin/ai-generation/jobs/:job_id/discard-all` — processa todas as questões `pending` aplicando a lógica de AIGEN-RF-007 com `reason="descarte em lote"`. O admin pode adicionar razão customizada no body: `{ reason? }`.
- **CA-3:** Se não houver questões `pending` → HTTP 409, `"Não há questões pendentes de revisão neste job."`.
- **CA-4:** Questões já aprovadas ou descartadas não são reprocessadas — a operação afeta apenas as `pending`.
- **CA-5:** Resposta: `{ processed: N, approved: N | discarded: N, errors: [] }`. Erros individuais (improvável, mas possível) são listados sem abortar o lote.
- **CA-6:** Cada aprovação/descarte gera entrada individual em `audit_log` (rastreabilidade por questão).

**Erros previstos:**
- **E-1:** Job inexistente → HTTP 404.
- **E-2:** Job `status ≠ completed` → HTTP 409.
- **E-3:** Sem questões pending → HTTP 409.

---

## AIGEN-RF-009 — Excluir job

**Prioridade:** Desejável.
**Ator:** Administrador.

**Descrição:**
Remove um job e suas questões não aprovadas do sistema. Questões já aprovadas (que viraram registros em `questions`) **não** são afetadas. Útil para limpeza do histórico.

**Critérios de aceitação:**
- **CA-1:** `DELETE /admin/ai-generation/jobs/:id`.
- **CA-2:** Job não pode estar em `processing` → E-2 (worker em andamento; cancele antes de excluir).
- **CA-3:** Questões `approved` (com `approved_question_id` preenchido) permanecem intactas em `questions` — o vínculo no `ai_generated_questions` é nulificado antes da exclusão do job.
- **CA-4:** Questões `pending` e `discarded` são excluídas junto com o job (hard-delete, pois nunca existiram no banco principal).
- **CA-5:** Se o job ainda tiver questões `pending` (revisão incompleta), exige confirmação explícita via body: `{ confirm_discard_pending: true }`. Sem isso → E-3.
- **CA-6:** Entrada no `audit_log` com `action=delete_ai_generation_job`.

**Erros previstos:**
- **E-1:** Job inexistente ou de outro admin → HTTP 404.
- **E-2:** Job em `processing` → HTTP 409, `"O job está em processamento. Aguarde a conclusão."`.
- **E-3:** Questões `pending` sem `confirm_discard_pending: true` → HTTP 409, `"Ainda há N questões pendentes de revisão. Confirme o descarte."`.

---

## Pendências deste módulo

- **AIGEN-P-01 — Notificação de conclusão do job.** Quando o job muda para `completed` ou `failed`, o admin deve ser avisado sem precisar ficar em polling ativo. Canal preferido: badge/toast in-app (a integrar com o sistema de notificações que vier a existir). Para o MVP, polling automático do frontend a cada 3 s enquanto `status ∈ {pending, processing}` é suficiente; push ou SSE pode vir depois.
- **AIGEN-P-02 — Retry de job com falha.** Um job em `failed` poderia ser "re-tentado" sem o admin ter que subir os PDFs novamente (se ainda estivessem em R2). Mas a ADR-0024 exclui os PDFs ao término — logo, retry exige novo upload. Decidir se vale guardar os PDFs por mais tempo (custo vs. comodidade) — adiado pós-MVP.
- **AIGEN-P-03 — Histórico cross-admin.** Atualmente cada admin vê apenas os próprios jobs. Se o admin quiser ver os jobs de outros admins (para gestão de custos), precisaria de um endpoint separado com filtro por `created_by`. Avaliar necessidade após uso real.
- **AIGEN-P-04 — Escolha de modelo pelo admin.** Expor ao admin a possibilidade de escolher entre `claude-sonnet-4-6` (mais rápido e barato) e `claude-opus-4-8` (mais elaborado). Adiado — começa com Sonnet fixo e avalia qualidade na prática.
