# Contrato da API — BomberQuiz

> **Status:** planta de design. A spec OpenAPI **canônica** é gerada pelo backend a partir dos schemas Zod (`@hono/zod-openapi`, ADR-0008) — este documento **não** é o `openapi.json`. Ele consolida, num só lugar, o **inventário de endpoints** (hoje espalhado pelos RFs) e as **convenções transversais** do contrato, servindo de planta para a implementação e de checklist de conformidade da spec gerada.
>
> Fontes de verdade por trás de cada linha: os arquivos `docs/rf/*.md`. Em divergência, o RF prevalece e este doc é corrigido.

---

## 1. Convenções gerais

### Base, versionamento e formato
- **Sem prefixo de versão** (`/v1`) no MVP. Frontend (`bomberquiz-web`) e backend (`bomberquiz-api`) são deployados em conjunto e o cliente HTTP é gerado da spec a cada release (ADR-0008/0010), então mudanças de contrato são coordenadas em lockstep. Reservar `/v1` só se surgir uma API pública para terceiros.
- Caminhos relativos à `API_BASE_URL` (ver `arquitetura.md`).
- `Content-Type: application/json; charset=utf-8`, exceto uploads de imagem (`multipart/form-data` — CONT-RF-013, PART-RF-006, PROF-RF-005).
- Datas em **ISO 8601**; agregações temporais no fuso `America/Sao_Paulo` (QUIZ-RF-010, jobs). Valores monetários sempre em **centavos (inteiro)** (Módulo 6).

### Autenticação e autorização
- Sessão via **cookie httpOnly** (Better-Auth, ADR-0018). Não há token Bearer no MVP.
- Política de **sessão única** (PROF-RF-010): requisição com sessão substituída → `401 { reason: "session_replaced" }`.
- Papéis (RBAC): `client` < `partner` < `admin`. `admin` herda acessos de `partner` e `client`; `partner` herda `client` (ADR-0007).
- Convenção de namespaces:
  - `/auth/*` — fluxos de identidade (Better-Auth + extensões), público ou semi-público.
  - `/me/*` — recursos do **próprio usuário** autenticado (`author_id`/`user_id` implícito = sessão).
  - `/admin/*` — exige `role=admin`.
  - `/quizzes/*`, `/plans` — cliente autenticado (com acesso ativo onde indicado).
  - `/webhooks/*` — público, autenticado por HMAC (não por sessão).

### Paginação, ordenação e filtros
- Listagens paginadas: query `page` (1-based) e `page_size` (default **20**, máx **100**). Resposta inclui `{ items: [...], page, page_size, total }` (envelope de paginação único em todas as listas).
- Ordenação via `sort` (campo) — valores aceitos documentados por endpoint; default por endpoint.
- Filtros via query string, conforme cada RF (ex.: `status`, `q`, `subject_id`).

### Envelope de erro
- Formato único `{ error: { code, message, details?, fields?, request_id } }` — definição completa em [`arquitetura.md` § Formato padronizado de resposta de erro](arquitetura.md). Catálogo de `code`s compartilhado via geração OpenAPI.
- HTTP status permanece significativo (401/402/403/404/409/413/422/429/502). `422` carrega `fields[]` (validação Zod). `429` carrega header `Retry-After`.

### Rate limiting
- Baseline 60 req/min por IP (sem sessão) e 120 req/min por usuário; limites específicos por endpoint têm precedência (ver `arquitetura.md` § Rate limiting e os RFs). Excesso → `429 { code: "rate_limit_exceeded" }`.

### Idempotência
- `POST /webhooks/mercado-pago` é idempotente por `mp_payment_id` + estado (SUB-RF-004 CA-5).
- Submissão de resposta de quiz é imutável por posição (QUIZ-RF-002): re-submissão → `409`.

### Auditoria
- Toda ação sensível grava em `audit_log` na mesma transação (ver `arquitetura.md` § Audit log). Não é exposto como endpoint no MVP.

---

## 2. Inventário de endpoints

Legenda de **Acesso**: `público` (sem sessão) · `sessão` (qualquer papel logado) · `client/partner/admin` (papel mínimo) · `ativo` (exige acesso vigente — trial/assinatura/cortesia, QUIZ-RF-009) · `HMAC` (webhook).

### Módulo 1 — Autenticação e cadastro (`auth.md`)
Fluxos de identidade apoiados no Better-Auth (ADR-0018); as rotas abaixo são a convenção do projeto — os nomes nativos do Better-Auth são montados/alinhados a elas.

| Método | Rota | Acesso | RF | Descrição |
|---|---|---|---|---|
| POST | `/auth/register` | público | AUTH-RF-001 | Cadastro (cria `client`, dispara verificação) |
| GET/POST | `/auth/verify-email` | público | AUTH-RF-002 | Verifica e-mail via token (uso único, 24h) |
| POST | `/auth/resend-verification` | público | AUTH-RF-003 | Reenvia verificação (resposta sempre genérica) |
| POST | `/auth/login` | público | AUTH-RF-004 | Login; avalia promoção admin (AUTH-RF-010) |
| POST | `/auth/logout` | sessão | AUTH-RF-005 | Encerra a sessão (idempotente) |
| POST | `/auth/forgot-password` | público | AUTH-RF-006 | Solicita recuperação (token 1h) |
| POST | `/auth/reset-password` | público | AUTH-RF-007 | Redefine senha com token; invalida sessões |
| GET | `/me` | sessão | AUTH-RF-008, PROF-RF-006 | Hidrata usuário corrente; flag `requires_consent_renewal` |

### Módulo 2 — Perfil e papéis (`profile.md`)

| Método | Rota | Acesso | RF | Descrição |
|---|---|---|---|---|
| GET | `/me/profile` | sessão | PROF-RF-001 | Dados do próprio perfil |
| PATCH | `/me/profile` | sessão | PROF-RF-002 | Edita nome/telefone/DOB/sexo (parcial) |
| POST | `/me/password` | sessão | PROF-RF-003 | Troca de senha (logado) |
| POST | `/me/email` | sessão | PROF-RF-004 | Solicita troca de e-mail (confirmação por link) |
| GET/POST | `/me/email/confirm` | público | PROF-RF-004 | Confirma troca via token; alerta ao e-mail antigo |
| POST | `/me/avatar` | sessão | PROF-RF-005 | Upload de avatar (Cloudinary) — `multipart` |
| DELETE | `/me/avatar` | sessão | PROF-RF-005 | Remove avatar (volta a placeholder) |
| POST | `/me/consent` | sessão | PROF-RF-006 | Reaceite de termos quando a versão muda |
| POST | `/me/deactivate` | sessão | PROF-RF-007 | Desativa conta (reversível; reautenticação) |
| POST | `/auth/reactivate` | público | PROF-RF-008 | Reativa conta `inactive` no fluxo de login |
| DELETE | `/me` | sessão | PROF-RF-009 | Exclusão LGPD (anonimização; reautenticação) |
| GET | `/admin/users` | admin | PROF-RF-011 | Busca/lista usuários |
| GET | `/admin/users/:id` | admin | PROF-RF-014 | Detalhe de usuário + resumo financeiro |
| PATCH | `/admin/users/:id/role` | admin | PROF-RF-012/013 | Promove a parceiro / revoga parceiro |

### Módulo 3 — Conteúdo admin (`content-admin.md`)

| Método | Rota | Acesso | RF | Descrição |
|---|---|---|---|---|
| GET | `/admin/axes` | admin | CONT-RF-001 | Lista eixos |
| POST | `/admin/axes` | admin | CONT-RF-002 | Cria eixo |
| PATCH | `/admin/axes/:id` | admin | CONT-RF-003 | Edita eixo |
| POST | `/admin/axes/:id/archive` | admin | CONT-RF-004 | Arquiva/desarquiva eixo |
| GET | `/admin/subjects` | admin | CONT-RF-005 | Lista matérias |
| POST | `/admin/subjects` | admin | CONT-RF-006 | Cria matéria |
| PATCH | `/admin/subjects/:id` | admin | CONT-RF-007 | Edita matéria |
| POST | `/admin/subjects/:id/archive` | admin | CONT-RF-008 | Arquiva/desarquiva matéria |
| GET | `/admin/questions` | admin | CONT-RF-009 | Lista perguntas (filtros amplos) |
| GET | `/admin/questions/:id` | admin | CONT-RF-009 | Detalhe completo (usado pela edição) |
| POST | `/admin/questions` | admin | CONT-RF-010 | Cria pergunta (publica direto; `?as_draft=true`) |
| PATCH | `/admin/questions/:id` | admin | CONT-RF-011 | Edita pergunta (`reset_stats?`) |
| POST | `/admin/questions/:id/archive` | admin | CONT-RF-012 | Arquiva/desarquiva |
| DELETE | `/admin/questions/:id` | admin | CONT-RF-012 | Hard-delete (só se `total_answers=0`) |
| POST | `/admin/questions/:id/image` | admin | CONT-RF-013 | Upload de imagem (Cloudinary) — `multipart` |
| DELETE | `/admin/questions/:id/image` | admin | CONT-RF-013 | Remove imagem |
| GET | `/admin/questions/pending` | admin | CONT-RF-014 | Fila de revisão (FIFO) |
| POST | `/admin/questions/:id/approve` | admin | CONT-RF-015 | Aprova pergunta de parceiro |
| POST | `/admin/questions/:id/reject` | admin | CONT-RF-016 | Rejeita (motivo obrigatório) |

### Módulo 4 — Conteúdo parceiro (`content-partner.md`)

| Método | Rota | Acesso | RF | Descrição |
|---|---|---|---|---|
| GET | `/me/questions` | partner | PART-RF-001 | Lista as próprias perguntas |
| POST | `/me/questions` | partner | PART-RF-002 | Cria rascunho |
| PATCH | `/me/questions/:id` | partner | PART-RF-003 | Edita própria (publicada → volta a revisão) |
| POST | `/me/questions/:id/submit` | partner | PART-RF-004 | Envia para revisão (`draft → pending_review`) |
| DELETE | `/me/questions/:id` | partner | PART-RF-005 | Exclui (somente `draft`) |
| POST | `/me/questions/:id/image` | partner | PART-RF-006 | Upload de imagem própria — `multipart` |
| DELETE | `/me/questions/:id/image` | partner | PART-RF-006 | Remove imagem própria |
| GET | `/me/questions/:id/review-history` | partner | PART-RF-007 | Histórico de revisão (marca como lido) |
| GET | `/me/partner/dashboard` | partner | PART-RF-008 | Painel/agregados do parceiro |

### Módulo 5 — Quiz (`quiz.md`)

| Método | Rota | Acesso | RF | Descrição |
|---|---|---|---|---|
| POST | `/quizzes` | ativo | QUIZ-RF-001 | Inicia quiz (sorteia, snapshot) |
| POST | `/quizzes/:id/answers` | ativo | QUIZ-RF-002 | Submete resposta (revalida cronômetro) |
| POST | `/quizzes/:id/finish` | ativo | QUIZ-RF-003 | Finaliza manualmente |
| GET | `/quizzes/:id/result` | sessão | QUIZ-RF-005 | Resultado (leitura liberada sem assinatura) |
| GET | `/me/quizzes` | sessão | QUIZ-RF-006 | Histórico de quizzes |
| GET | `/me/performance` | sessão | QUIZ-RF-007 | Painel de desempenho (leitura liberada) |
| POST | `/me/performance/reset` | sessão | QUIZ-RF-008 | Zera estatísticas (reautenticação) |
| GET | `/me/performance/timeline` | sessão | QUIZ-RF-010 | Evolução mensal (`months=1..24`) |
| GET | `/axes` | sessão | QUIZ-RF-001 | Lista eixos ativos (enxuto: id/name/tap_weight) — popula o seletor de `free_axis` |
| GET | `/subjects` | sessão | QUIZ-RF-001 | Lista matérias ativas (enxuto: id/name/axis_id), filtro `axis_id?` — popula o seletor de `free_subject` |

> QUIZ-RF-004 (expiração/auto-abandono) e QUIZ-RF-009 (bloqueio `402`) são, respectivamente, **job** e **middleware** — não endpoints próprios. `GET /axes`/`GET /subjects` não são `/admin/*` (que exigem `role=admin` e servem a UI de gestão de conteúdo, Módulo 3) — são catálogo de leitura para qualquer sessão, introduzidos junto do frontend de QUIZ-RF-001.

### Módulo 6 — Assinaturas e cortesias (`subscriptions.md`)

| Método | Rota | Acesso | RF | Descrição |
|---|---|---|---|---|
| GET | `/plans` | público | SUB-RF-001 | Lista planos ativos (preços pré-cadastro), ordenados por `duration_days` |
| GET | `/admin/plans` | admin | SUB-RF-001 | Catálogo completo, **inclui desativados** (o público não). Traz `id` e `is_active`. Sem paginação — são 4 planos fixos. |
| PATCH | `/admin/plans/:id` | admin | SUB-RF-001 | Edita `pix_price`, `card_price`, `max_installments`, `is_active` — e **só** esses (slug/nome/duração são imutáveis). 404 `plan_not_found`; 422 `card_price_below_pix_price` e para preço abaixo de 100 centavos (piso do MP). Grava `audit_log` com `changes` **e** `previous`. Não retroage: compras feitas e cobranças PIX pendentes mantêm o valor da época. |
| POST | `/me/checkout` | sessão | SUB-RF-003 | Inicia checkout. PIX e **cartão** implementados; saldo MP responde 422. Resposta discriminada por `method`: PIX devolve QR code + `expires_at`; cartão devolve o desfecho da autorização (`order_status`/`payment_status`/`status_detail`) e recebe `card_token`/`payment_method_id`/`device_id`/`installments` do Brick (ADR-0040). Cupom ainda não aceito. |
| POST | `/webhooks/mercado-pago` | HMAC | SUB-RF-004 | Confirmação de pagamento (idempotente) |
| GET | `/me/subscription` | sessão | SUB-RF-005 | Status da própria assinatura. Inclui `refund_eligible_payments` e, quando `access_status=inactive`, `cta: "subscribe"`. |
| GET | `/me/payments` | sessão | SUB-RF-006 | Histórico paginado. Filtros `status` (CSV), `created_from`, `created_to`. Cada item traz `refundable`/`refund_deadline` e `mp_receipt_url` (**só PIX** — cartão não tem comprovante no MP). |
| POST | `/me/payments/:id/refund` | sessão | SUB-RF-014 | Reembolso total (CDC 7 dias), `reason` opcional. 404 para pagamento inexistente **ou** de outro cliente; 409 `payment_not_refundable`/`refund_window_expired`; 502 sem alterar estado local. Revoga a assinatura correspondente. |
| POST | `/admin/courtesies` | admin | SUB-RF-008 | Concede cortesia |
| GET | `/admin/courtesies` | admin | SUB-RF-009 | Lista cortesias |
| POST | `/admin/courtesies/:id/revoke` | admin | SUB-RF-010 | Revoga cortesia (motivo obrigatório) |
| GET | `/admin/financial/overview` | admin | SUB-RF-012 | Painel financeiro (agregados) |
| POST | `/admin/coupons` | admin | SUB-RF-013 | Cria cupom |
| GET | `/admin/coupons` | admin | SUB-RF-013 | Lista cupons |
| PATCH | `/admin/coupons/:id` | admin | SUB-RF-013 | Edita cupom (campos limitados) |

> SUB-RF-002 (trial ao verificar e-mail), SUB-RF-007 (lembretes), SUB-RF-011 (`access_status`) e SUB-RF-015 (aplicação de cupom) são **gatilhos/jobs/serviços internos**, não endpoints.

### Módulo 7 — Geração de Questões por IA (`ai-generation.md`)

> ⚠️ **Retirado de produção em 2026-07-27.** As rotas `/admin/ai-generation/*` descritas em `ai-generation.md` (AIGEN-RF-001 a 009) foram implementadas e depois removidas por completo de `bomberquiz-api` no mesmo dia, após um incidente de OOM em produção — ver [`decisoes.md`](decisoes.md) § ADR-0039. Nenhuma dessas rotas existe hoje; não há checklist de conformidade a manter aqui. Geração de questões por IA hoje acontece via `ai-bot`, ferramenta local fora da spec de requisitos (ver `../CLAUDE.md`), que publica através de `POST /admin/questions` já existente.

---

## 3. Convenções de implementação (para a geração da spec)

- Cada rota é um `createRoute` (`@hono/zod-openapi`) com `request`/`responses` tipados por Zod; o catálogo de `code`s de erro nasce desses schemas.
- Tags da spec = nome do módulo (Auth, Perfil, Conteúdo Admin, Conteúdo Parceiro, Quiz, Assinaturas). Facilita a UI Scalar e a navegação do cliente gerado.
- `operationId` estável e legível (ex.: `startQuiz`, `submitAnswer`, `grantCourtesy`) — vira o nome do método no cliente type-safe do frontend.
- Schemas de paginação e do envelope de erro são **componentes reutilizáveis** (definidos uma vez, referenciados em todas as rotas).
- A spec é baixada manualmente da API local para regerar o cliente (`bun run openapi:generate`) — arquitetura.md § Cliente HTTP. Sem staging (ADR-0037), não há mais um backend deployado intermediário pra servir isso automaticamente.
