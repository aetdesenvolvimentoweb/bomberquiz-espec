# Arquitetura — BomberQuiz

> Visão técnica do sistema. Para decisões e justificativas, ver [`decisoes.md`](decisoes.md). Para o que o sistema faz, ver [`requisitos.md`](requisitos.md).

## Visão geral

```
┌──────────────────────┐        ┌──────────────────────┐
│  bomberquiz-web      │        │  bomberquiz-api      │
│  (PWA — Vite/React)  │ HTTPS  │  (REST — Bun/Hono)   │
│  Cloudflare Pages    │ ─────▶ │  Fly.io              │
└──────────────────────┘        └──────────┬───────────┘
                                           │
              ┌────────────────────────────┼────────────────────────┐
              │                            │                        │
              ▼                            ▼                        ▼
       ┌─────────────┐            ┌────────────────┐        ┌──────────────┐
       │ Postgres    │            │ Mercado Pago   │        │ WhatsApp /   │
       │ (Neon)      │            │ (assinaturas)  │        │ E-mail/R2/   │
       │             │            │                │        │ Cloudinary   │
       └─────────────┘            └────────────────┘        └──────────────┘
```

Dois repositórios independentes no GitHub:
- **`bomberquiz-api`** — backend REST.
- **`bomberquiz-web`** — frontend PWA.

Contrato entre eles: **OpenAPI** gerado pelo backend a partir de schemas Zod. Cliente HTTP do frontend é tipado a partir dessa spec.

---

## Backend (`bomberquiz-api`)

### Princípios
- **Arquitetura hexagonal** (ports & adapters): domínio puro no centro, infra plugável na borda.
- Dependências fluem **de fora para dentro**: `http → application → domain`. `infra` implementa portas definidas em `application` ou `domain`.
- Zero acoplamento do domínio com ORM, HTTP, ou gateways externos.

### Estrutura de pastas

```
bomberquiz-api/
├── src/
│   ├── domain/                  # Entidades e regras de negócio puras
│   │   ├── user/
│   │   │   ├── user.entity.ts
│   │   │   ├── user.repository.port.ts
│   │   │   ├── phone.vo.ts              # Value object, valida via IPhoneValidator (ADR-0030)
│   │   │   ├── phone-validator.port.ts
│   │   │   └── user.errors.ts
│   │   ├── question/
│   │   ├── quiz/
│   │   ├── subscription/        # Inclui regra de acúmulo de cortesia
│   │   └── statistics/          # Fórmula de nível da pergunta
│   ├── application/             # Use cases (um arquivo por caso de uso)
│   │   ├── auth/
│   │   │   ├── register-user.usecase.ts
│   │   │   ├── verify-email.usecase.ts
│   │   │   └── login.usecase.ts
│   │   ├── quiz/
│   │   │   ├── start-quiz.usecase.ts
│   │   │   ├── submit-answer.usecase.ts
│   │   │   └── ports/           # Interfaces que infra implementa
│   │   └── subscription/
│   │       ├── donate-subscription.usecase.ts
│   │       └── ports/
│   ├── infra/                   # Adapters concretos
│   │   ├── persistence/
│   │   │   ├── drizzle/
│   │   │   │   ├── schema.ts
│   │   │   │   ├── client.ts
│   │   │   │   └── repositories/
│   │   │   └── migrations/
│   │   ├── payments/
│   │   │   └── mercadopago.adapter.ts
│   │   ├── messaging/
│   │   │   └── whatsapp.adapter.ts
│   │   ├── email/
│   │   │   └── resend.adapter.ts
│   │   ├── phone/
│   │   │   └── regex-phone-validator.adapter.ts   # implementa IPhoneValidator (ADR-0030)
│   │   └── storage/
│   │       ├── r2.adapter.ts            # imagens de questões
│   │       └── cloudinary.adapter.ts    # avatares
│   ├── http/                    # Camada de entrada HTTP
│   │   ├── app.ts               # Hono app + middleware globais
│   │   ├── routes/
│   │   │   ├── auth.routes.ts
│   │   │   ├── quiz.routes.ts
│   │   │   ├── admin.routes.ts
│   │   │   └── ...
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts
│   │   │   ├── rbac.middleware.ts
│   │   │   └── rate-limit.middleware.ts
│   │   └── schemas/             # Zod schemas para request/response
│   ├── config/
│   │   ├── env.ts               # Validação Zod do .env
│   │   └── admin-whitelist.ts
│   └── index.ts                 # Entry point
├── tests/
│   ├── unit/                    # Domain + use cases (sem I/O)
│   ├── integration/             # Use case + infra real (Neon branch ou container)
│   └── e2e/                     # HTTP black-box contra app em execução
├── drizzle.config.ts
├── package.json
└── tsconfig.json
```

### Fluxo típico de uma request

```
HTTP Request
   │
   ▼
┌──────────────────────────────────────────┐
│ http/routes/quiz.routes.ts               │
│  - valida com Zod (req body, params)     │
│  - extrai usuário autenticado            │
│  - chama use case                        │
└──────────────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────────────┐
│ application/quiz/start-quiz.usecase.ts   │
│  - orquestra regras do domínio           │
│  - usa portas (QuizRepository, etc)      │
└──────────────────────────────────────────┘
   │             │
   │             ▼
   │      ┌──────────────────────────┐
   │      │ domain/quiz/             │
   │      │  - regras puras          │
   │      └──────────────────────────┘
   ▼
┌──────────────────────────────────────────┐
│ infra/persistence/drizzle/repositories/  │
│  - implementa QuizRepository             │
│  - traduz domínio ↔ schema do banco      │
└──────────────────────────────────────────┘
```

### Autenticação e autorização

- **Better-Auth** gerencia sessões (cookie httpOnly, SameSite=Lax) e fluxos de e-mail.
- Middleware HTTP popula `c.var.user` (ou retorna 401).
- RBAC simples por papel: `admin`, `partner`, `client`. Decoração por rota: `requireRole("admin")`.
- Admins detectados na primeira autenticação contra whitelist em `config/admin-whitelist.ts` — promoção automática (ADR-0005).

### Modelo de dados (visão conceitual)

> **O schema canônico são as migrations Drizzle no repo `bomberquiz-api`** (tarefa em `docs/tarefas.md`). Esta lista é a visão conceitual consolidada a partir dos RFs em `docs/rf/*.md` — mantida sincronizada com eles, não com a implementação. Campos abaixo são indicativos, não exaustivos.

**Identidade e conta**
- `users` — id, name, email (unique, case-insensitive), `pending_email?`, phone, dob, sex, password_hash, avatar_url, email_verified_at, role (`client`/`partner`/`admin`), status (`active`/`inactive`/`deleted`), consent_version, consent_accepted_at, trial_used_at, last_login_at, deactivated_at?, deleted_at?, email_changed_at?, created_at. _Anonimização LGPD em PROF-RF-009; e-mail anonimizado via `sha256(SALT_GLOBAL + email)`._
- `sessions` — id, user_id, last_seen_at, expires_at, revocation_reason? (`session_replaced`/`logout`/`password_reset`), created_at. _Política de sessão única, PROF-RF-010._
- Tokens de **verificação de e-mail**, **recuperação de senha** e **confirmação de troca de e-mail** — geridos pelo Better-Auth onde possível (ver ADR-0018); armazenados como hash, uso único, com expiração própria (AUTH-RF-002/006, PROF-RF-004).

**Conteúdo (Eixo → Matéria → Pergunta)**
- `axes` — id, name (unique, case-insensitive), description?, tap_weight (int ≥ 0 — nº de questões do eixo na prova real, conforme edital vigente), status (`active`/`archived`), created_at, archived_at?.
- `subjects` — id, axis_id (FK), name (unique no eixo), official_source?, status, created_at, archived_at?.
- `questions` — id, subject_id (FK), statement, alternatives (4 strings), correct_index (0..3), explanation, source_reference?, image_url?, status (`draft`/`pending_review`/`published`/`archived`), author_id (FK users), reviewed_by?, reviewed_at?, rejection_reason?, difficulty_level (`unrated`/`easy`/`medium`/`hard`), difficulty_recomputed_at?, stats_reset_at?, created_at, updated_at, published_at?, archived_at?.
- `question_stats` — question_id (PK/FK), total_answers, correct_count, last_updated. _Base do `accuracy` e do job de dificuldade (CONT-RF-017)._

**Quiz e desempenho**
- `quiz_sessions` — id, user_id, mode (`tap_simulation`/`free_subject`/`free_axis`), scope_id?, total_questions, timer_enabled, time_limit_seconds?, explanation_mode (`after_each`/`at_end`), status (`in_progress`/`finished`/`expired`/`abandoned`), correct_count, answered_count, started_at, finished_at?.
- `quiz_session_questions` — id, quiz_session_id (FK), question_id (FK), position (1..N), **snapshot** (statement/alternatives/correct_index/explanation/source_reference no momento do sorteio), submitted_index?, is_correct?, answered_at?.
- `user_subject_stats` (agregado materializado) — user_id, subject_id, total_answers, correct_answers, last_answered_at, stats_reset_at?. _Estatística por eixo é derivada em runtime._

**Assinaturas, cortesias e financeiro**
- `subscription_plans` — id, slug (`monthly`/`quarterly`/`semiannual`/`annual`), name, duration_days, pix_price (centavos), card_price (centavos), max_installments, is_active, created_at, updated_at.
- `subscriptions` — id, user_id, plan_id?, source (`trial`/`paid`/`courtesy`), courtesy_id?, payment_id?, start_at, end_at, status (`active`/`expired`/`revoked`/`pending_payment`), created_at.
- `payments` — id, user_id, plan_id, method (`pix`/`mp_balance`/`card`), gross_amount, discount_amount, net_amount (centavos), coupon_id?, installments, mp_payment_id, mp_status, mp_receipt_url?, status (`pending`/`paid`/`failed`/`refunded`), paid_at?, refunded_at?, failure_reason?, created_at, updated_at.
- `subscription_reminders` — id, user_id, milestone (`d7`/`d3`/`d1`/`d0`), target_end_at, sent_at. Único em (user_id, milestone, target_end_at) — é o índice que dá idempotência ao job de SUB-RF-007 e faz a renovação resetar os marcos sozinha.
- `courtesies` — id, beneficiary_user_id, granted_by_admin_id, days_granted, start_at, end_at, category (`parceria`/`demonstracao`), notes?, revoked_at?, revoked_by_admin_id?, revocation_reason?, created_at.
- `coupons` — id, code (unique, case-insensitive), discount_type (`percent`/`fixed_cents`), discount_value, valid_from?, valid_until?, max_uses?, used_count, is_active, applies_to_plan_slugs?, created_by_admin_id, created_at.
- `user_access` — projeção derivada/materializada por usuário: access_status (`active`/`inactive`), active_until, source. _Calculada em SUB-RF-011; consumida por QUIZ-RF-009 e PROF-RF-014._

**Transversal**
- `audit_log` — schema completo na seção [Audit log](#audit-log) abaixo.

### Configuração e secrets

- `.env` validado via Zod em `config/env.ts`. App não inicia se faltar variável.
- Secrets em produção via Fly.io secrets (`fly secrets set`).
- Whitelist de admins é um array em `config/admin-whitelist.ts` versionado, **não** banco.

### Testes

- **Unit** (`tests/unit/`): domínio e use cases, com portas fakes/mocks em memória. Rápidos (<5s suite inteira).
- **Integration** (`tests/integration/`): use cases conectados a infra real (Drizzle + Neon branch dedicado por job CI; ou Postgres em Docker para local). Cada teste roda em transação revertida ao final.
- **E2E** (`tests/e2e/`): HTTP black-box contra app inicializado, banco real.

### Observabilidade

- **Erros para Sentry** (free tier, ~5k eventos/mês): integrado em 2026-08-03 no frontend (`web/src/lib/monitoring/sentry.ts`, hooks em `global-error-handler.ts`/`route-error-boundary.tsx`) e no backend (`api/src/infra/monitoring/sentry.ts`, hook em `app.onError`/scheduler/e-mail). Gated por `SENTRY_DSN`/`VITE_SENTRY_DSN` (opcionais — sem eles o SDK não inicializa); falta criar a conta/organização Sentry e configurar os DSNs como secret em produção para ativar de fato. Eventos correlacionados entre front e back pelo mesmo `request_id` do envelope de erro (ver ADR-0031). PII mascarada antes do envio (cookies, headers de auth, e-mail).
- **Telemetria de uso do PWA/quiz**: tabela própria `pwa_events` (Postgres/Neon, não um SaaS de analytics — ver ADR-0031/tarefas.md) grava eventos de instalação (`pwa_install_*`) e de período offline durante quiz em andamento (`quiz_offline_period`, com `duration_ms`), via `POST /events`. Objetivo: embasar com dados reais a decisão de QUIZ-P-01 (quiz offline com sincronização).
- Logs estruturados (pino): ainda não implementado (futuro).
- Métricas básicas via Fly.io: ainda não implementado (futuro).

### Domínios, URLs e cookies

Estratégia em duas fases para evitar custo desnecessário no início e ter migração suave para domínio próprio antes do lançamento.

**Fase 1 — Desenvolvimento e beta privado (atual):**
- Frontend: `bomberquiz.pages.dev` (default Cloudflare Pages)
- Backend: `bomberquiz-api.fly.dev` (default Fly.io)
- Cookie de sessão: `SameSite=None; Secure; HttpOnly` (cross-site exige `None`)
- CORS: backend permite explicitamente `Origin: https://bomberquiz.pages.dev`

**Fase 2 — Lançamento público:**
- Frontend: `app.bomberquiz.com.br` (CNAME para Pages)
- Backend: `api.bomberquiz.com.br` (CNAME para Fly)
- Cookie de sessão: `Domain=.bomberquiz.com.br; SameSite=Lax; Secure; HttpOnly` (mesma origem, mais seguro contra CSRF)
- CORS: backend permite `Origin: https://app.bomberquiz.com.br`
- **E-mail:** transacional (Resend) verificado em `send.bomberquiz.com.br` (SPF/DKIM/MX próprios, sem conflitar com a raiz); caixa humana de suporte (`suporte@bomberquiz.com.br`, Hostinger Business) com MX na raiz do domínio. Detalhes e justificativa em ADR-0028.

**Status (2026-07-10):** Fase 2 migrada e validada em produção — `bomberquiz.com.br` é o domínio oficial (`bomberquiz.com` é redirect 301 permanente), frontend em `app.bomberquiz.com.br`, backend em `api.bomberquiz.com.br`, e-mail transacional verificado em `send.bomberquiz.com.br`, suporte humano em `suporte@bomberquiz.com.br`. Detalhes da decisão em ADR-0028; histórico da execução em `tarefas.md` (entrada de 2026-07-10).

**Implicações arquiteturais (válidas desde a Fase 1):**
- URLs **em variáveis de ambiente**, nunca hardcoded. Variáveis chave:
  - `WEB_ORIGIN` — URL pública do frontend (compõe links em e-mails, valida CORS)
  - `API_BASE_URL` — URL pública do backend (usado pelo frontend e em redirects MP)
  - `COOKIE_DOMAIN` — vazio na Fase 1; `.bomberquiz.com.br` na Fase 2
  - `MP_WEBHOOK_URL` — `${API_BASE_URL}/webhooks/mercado-pago` (informado ao MP no painel deles)
  - `SUPPORT_EMAIL` — endereço de suporte exibido em mensagens de erro e e-mails (default `suporte@bomberquiz.com.br`). Canal de suporte do MVP é **e-mail** (WhatsApp adiado, ADR-0012). É o destino indicado em fluxos como reembolso fora da janela de 7 dias (SUB-RF-014), falha de pagamento e portabilidade LGPD sob demanda (ADR-0015).
- Links em e-mails transacionais (verificação, recuperação, expiração) usam `WEB_ORIGIN`.
- Mercado Pago aceita atualizar URL do webhook quando migrar para Fase 2 — sem reescrita. Pendente: URL ainda não configurada no painel do MP, já que a integração de pagamentos não existe (ver Backlog em `tarefas.md`).
- Custo da Fase 2: registro `bomberquiz.com.br` no Registro.br (~R$40/ano).

**Pegadinhas de execução no Cloudflare** (confirmadas na migração de 2026-07-10):
- O registro DNS do domínio da API (`api.bomberquiz.com.br` → Fly.io) precisa
  ficar **DNS-only** (proxy desligado, "nuvem cinza") na zona Cloudflare —
  com o proxy ligado, o desafio ACME do `flyctl certs add` falha na emissão
  do certificado. Proxy pode ser ativado depois, como melhoria opcional
  (WAF), só após o certificado já emitido e trocando o SSL/TLS mode da zona
  para **Full (strict)**.
- Um redirect de host (ex.: `www.` → `app.`, ou `bomberquiz.com` →
  `bomberquiz.com.br`) só funciona se existir **algum** registro DNS
  proxied para esse host na zona — o Cloudflare só intercepta requisições
  de hosts que existem na zona antes de aplicar uma Redirect Rule. Host sem
  registro nenhum cai direto em erro de conexão, sem chance de redirecionar.

---

### Rate limiting

Defesa em duas camadas, complementando os limites específicos de endpoints sensíveis já definidos nos RFs (login, checkout, upload, etc.).

| Camada | Regra | Onde aplica |
|---|---|---|
| **Global por IP (sem sessão)** | 60 req/min por IP | Endpoints públicos: `POST /auth/*`, `GET /plans`, recuperação de senha |
| **Por usuário (com sessão)** | 120 req/min por `user_id` | Todo endpoint autenticado que não tenha rate limit específico mais estrito |

**Comportamento ao exceder:** HTTP 429 com header `Retry-After` e envelope de erro padrão (`code: "rate_limit_exceeded"`, `details.retry_after_seconds`).

**Persistência:** sliding window in-memory por instância no MVP — viável porque rodaremos 1 instância no Fly.io. Quando escalar para múltiplas instâncias, migrar para Upstash Redis (free tier) ou tabela Postgres com `pg_advisory_lock`.

**Bypass:** webhooks do Mercado Pago são **isentos** desse rate limit por IP — validação por HMAC já é a defesa. Range de IPs do MP é amplo demais para casar com o limite genérico.

**Limites específicos têm precedência:** quando um endpoint tem rate limit próprio (login = 5 falhas/15min, troca de e-mail = 1/24h, etc.), ele substitui o baseline.

**Fora de escopo do MVP:** proteção contra ataques distribuídos. Cloudflare (que já hospeda o frontend) pode ser configurada como proxy + WAF do backend Fly.io quando necessário.

---

### Jobs agendados

Scheduler **in-process** no backend (ADR-0017). Cada job é um caso de uso em `application/` disparado por `infra/scheduler/`; idempotente; fuso `America/Sao_Paulo`; horários/intervalos em env.

| Job | Cadência | RF | Comportamento |
|---|---|---|---|
| Recalcular dificuldade das perguntas | diário 00:00 | CONT-RF-017 | Reclassifica `published` em `unrated`/`easy`/`medium`/`hard`. |
| Lembretes de expiração de assinatura | diário 09:00 | SUB-RF-007 | ✅ implementado (2026-08-26). E-mails D-7/D-3/D-1 e e-mail final D-0; um por marco. **Também fecha as assinaturas vencidas** (`active` → `expired`), transição que nenhuma outra parte do sistema fazia. Idempotência via `subscription_reminders`, chaveada por (usuário, marco, `end_at` alvo) — não por `subscription_id`, que duplicaria e-mail para quem acumulou dias em duas compras. |
| Expiração e auto-abandono de quiz | a cada 5 min | QUIZ-RF-004 | `expired` por cronômetro; `abandoned` após 24h sem atividade. |
| Purga de sessões inativas | diário (ex. 03:00) | AUTH-RF-008 | Remove sessões com >7 dias sem uso. |
| Dump lógico do banco | diário (ex. 04:00) | ADR-0020 | `pg_dump` → Cloudflare R2 (retenção rotativa). |

**Premissa de instância única:** com >1 instância, jobs rodariam em duplicidade — a migração será lock distribuído (`pg_advisory_lock`) ou Fly scheduled machines, localizada em `infra/scheduler/` (mesma premissa do rate limit in-memory).

---

### Backup e recuperação

Dois níveis (ADR-0020):
- **PITR do Neon** — recuperação point-in-time dentro da janela do tier (curta no free; reavaliar upgrade pago se insuficiente).
- **Dump lógico agendado** — `pg_dump` diário enviado a um bucket **privado** no Cloudflare R2 (separado das imagens de questões), retenção rotativa (ex.: 7 diários + 4 semanais). Runbook de restauração no repo `bomberquiz-api`.

O dump contém PII e dados financeiros: bucket privado, acesso restrito, tratado sob a mesma base legal LGPD dos dados originais. Perda máxima entre dumps = intervalo de execução; o PITR cobre o intervalo fino enquanto na janela.

---

### E-mails transacionais (catálogo)

Todos enviados via **Resend** (ADR-0012), templates em **React Email**, links compostos com `WEB_ORIGIN`, remetente/responder em `SUPPORT_EMAIL`. Referência consolidada (a fonte de verdade de cada regra continua sendo o RF citado). Contas `inactive`/`deleted` não recebem transacionais de rotina (PROF-RF-007 CA-4) — exceto os **alertas de segurança** e a **confirmação de exclusão**, endereçados ao titular real.

| E-mail | Gatilho | RF | Observação |
|---|---|---|---|
| Verificação de e-mail | Cadastro / reenvio | AUTH-RF-002/003 | Token 24h, uso único |
| Recuperação de senha | Solicitação de reset | AUTH-RF-006 | Token 1h, uso único |
| Senha alterada (alerta) | Reset ou troca logado | AUTH-RF-007, PROF-RF-003 | Inclui timestamp + IP |
| E-mail de acesso alterado (alerta) | Troca de e-mail confirmada | PROF-RF-004 | Enviado ao **endereço antigo** |
| Conta excluída (confirmação) | Exclusão LGPD | PROF-RF-009 | Ao **endereço original**, antes do commit |
| Pagamento confirmado | Webhook MP `approved` | SUB-RF-004 | Plano, novo `end_at`, link do comprovante MP |
| Lembrete de expiração | Job 09:00, D-7/D-3/D-1 | SUB-RF-007 | Um por marco; link de checkout |
| Assinatura expirada (final) | Job 09:00, D-0 | SUB-RF-007 | Único, sem insistência posterior |
| Cortesia concedida | Admin concede | SUB-RF-008 | "Você recebeu N dias…" |
| Cortesia revogada | Admin revoga | SUB-RF-010 | Inclui motivo |
| Reembolso solicitado | Cliente pede reembolso | SUB-RF-014 | Valor + prazo de crédito |
| Pergunta aprovada | Admin aprova | CONT-RF-015 | "aprovada com alterações" se houve edição |
| Pergunta precisa de ajustes | Admin rejeita | CONT-RF-016 | Motivo na íntegra + link de reedição |

Falha de envio (Resend indisponível ou erro de API) é detectada, tentada novamente (até 3× com backoff curto) e fica em log estruturado quando esgota as tentativas — ver ADR-0031. Jobs de e-mail agendados são idempotentes: os lembretes de assinatura (implementados em 2026-08-26) reservam o marco em `subscription_reminders` **antes** de enviar, então uma queda entre reserva e envio custa um lembrete, nunca um e-mail duplicado. O filtro de contas `inactive`/`deleted` acima é aplicado na própria consulta do job.

---

### Formato padronizado de resposta de erro

Toda resposta de erro do backend devolve um envelope único, consumido de forma tipada pelo frontend (catálogo de `code`s gerado a partir de schemas Zod).

```json
{
  "error": {
    "code": "subscription_required",
    "message": "Sua assinatura expirou. Renove para continuar.",
    "details": { "trial_used": true, "last_active_until": "2026-05-21" },
    "fields": [ { "field": "email", "code": "invalid_format", "message": "E-mail inválido" } ],
    "request_id": "req_abc123"
  }
}
```

| Campo | Quando aparece | Notas |
|---|---|---|
| `code` | sempre | Identificador estável legível por máquina (`snake_case`). Frontend faz `switch` nele. |
| `message` | sempre | Texto pt-BR para exibir ao usuário. Genérico quando segurança exige ("E-mail ou senha incorretos"). |
| `details` | opcional | Contexto estruturado (ex.: `retry_after_seconds`, `remaining_days`, `missing_subjects`). |
| `fields` | apenas em HTTP 422 | Lista de erros de validação Zod, um por campo. Cada item tem `field`, `code`, `message`. |
| `request_id` | sempre | UUID por request, correlaciona com logs estruturados; frontend pode exibir para suporte. |

**Regras operacionais:**
- HTTP status (401/402/403/404/409/413/422/429/502) **continua significativo** — o envelope é o corpo, não substitui o código.
- Headers especiais (`Retry-After` em 429, `WWW-Authenticate` em 401) **mantêm-se em uso**, não migram para o envelope.
- Catálogo de `code`s vive em `@/shared/error-codes.ts` (compartilhado entre backend e frontend via geração OpenAPI).
- Mensagens **não vazam estado interno** (ex.: "e-mail já existe" → genérico "Não foi possível criar a conta"). Códigos específicos podem existir desde que não sejam expostos quando comprometem segurança.

---

### Audit log

Tabela transversal referenciada por todos os módulos para auditoria de ações sensíveis (promoção de papel, exclusão de conta, CRUD de conteúdo, aprovação/rejeição de pergunta, cortesias, cupons, reembolsos, webhooks MP).

**Schema da tabela `audit_log`:**

| Coluna | Tipo | Notas |
|---|---|---|
| `id` | uuid PK | |
| `actor_id` | uuid FK `users.id`, nullable | Null para ações do sistema (jobs, webhooks) |
| `actor_role_at_time` | text | `admin`/`partner`/`client`/`system` — snapshot do papel no momento da ação |
| `action` | text | Identificador da ação (`promote_partner`, `grant_courtesy`, `approve_question`, `mp_webhook_processed`, etc.) |
| `target_type` | text, nullable | `user`/`axis`/`subject`/`question`/`courtesy`/`payment`/`coupon`/etc. |
| `target_id` | uuid, nullable | FK lógica (sem constraint, preserva histórico se alvo for anonimizado/excluído) |
| `payload_summary` | jsonb | Diff de edição, motivo de revogação/rejeição, snapshot mínimo |
| `ip_address` | inet, nullable | Quando vem de request HTTP |
| `user_agent` | text, nullable | Idem |
| `created_at` | timestamptz, default `now()` | |

**Índices:**
- `(created_at DESC)` — listagens recentes.
- `(actor_id, created_at DESC)` — auditoria por ator.
- `(target_type, target_id, created_at DESC)` — histórico de um recurso específico.
- `(action, created_at DESC)` — relatórios por tipo de ação.

**Regras operacionais:**
- **Escrita síncrona, mesma transação** da ação principal — garante atomicidade.
- **Sem PII em `payload_summary`** — apenas IDs (referências). Textos livres como motivo de rejeição/revogação são permitidos por serem do autor da ação, não snapshot de terceiros.
- **Retenção indefinida no MVP** — volume baixo justifica não purgar. Reavaliar partitioning mensal quando crescer.
- **Entradas referentes a usuários anonimizados (PROF-RF-009)** permanecem — o `target_id` continua apontando para o `users.id`, que existe com PII zerada.
- **`ip_address`/`user_agent` são PII** e sobrevivem à anonimização da conta (ficam no log, não na tabela `users`). Por isso: (a) a **política de privacidade** deve declarar a coleta de IP para fins de segurança/auditoria e a base legal (legítimo interesse); (b) define-se um **expurgo desses dois campos após 12 meses** (job de manutenção, ADR-0017) — `created_at`, `actor_id`, `action` e `target_*` permanecem para a trilha de auditoria, mas IP e user-agent são zerados. Esse prazo é o único ponto em que a "retenção indefinida" do log tem exceção.

---

## Frontend (`bomberquiz-web`)

### Estrutura de pastas

```
bomberquiz-web/
├── src/
│   ├── app/                     # Setup React, providers, router
│   ├── pages/                   # Páginas por rota
│   │   ├── auth/                # Login, signup, verify-email, forgot-password
│   │   ├── quiz/                # Lista, sessão, resultado
│   │   ├── profile/             # Perfil, estatísticas
│   │   ├── admin/               # Área admin (questões, cortesias, financeiro)
│   │   └── partner/             # Área parceiro (suas questões)
│   ├── features/                # Slices funcionais reutilizáveis
│   │   ├── quiz/                # Componentes, hooks, queries de quiz
│   │   ├── subscription/
│   │   └── ...
│   ├── components/              # UI compartilhada (shadcn em ./ui)
│   │   └── ui/
│   ├── lib/                     # api client (gerado de OpenAPI), utils, hooks gerais
│   ├── stores/                  # Zustand stores
│   └── main.tsx
├── public/
│   ├── manifest.webmanifest
│   └── icons/                   # Ícones PWA
├── tests/
│   ├── unit/                    # Componentes e hooks isolados
│   └── e2e/                     # Playwright
├── vite.config.ts               # + vite-plugin-pwa
├── tailwind.config.ts
└── package.json
```

### Comportamento PWA

- Service worker (gerado por `vite-plugin-pwa`) cacheia assets estáticos e respostas idempotentes da API.
- Quiz em andamento pode continuar offline se as questões já foram baixadas (resolução do score sincroniza ao reconectar).
- Install prompt em iOS/Android.

### Cliente HTTP

- Gerado a partir do `openapi.json` do backend (ex.: `openapi-fetch` + `openapi-typescript`).
- Atualização do cliente é manual por ora (`bun run openapi:generate` contra a API local) — ver ADR-0037 sobre a aposentadoria do ambiente de staging.
- **Planta do contrato:** [`api.md`](api.md) consolida o inventário de endpoints e as convenções transversais que a spec gerada deve refletir.

---

## Operação

### Ambientes
- **Local:** Bun roda backend; Vite roda frontend; Postgres em Docker (ou branch dev no Neon).
- **Produção:** branch `main` → deploy automático após CI verde.
- Sem ambiente de staging — aposentado em 2026-08-03 (ADR-0037), nunca chegou a ser exercitado de fato.

### CI/CD (GitHub Actions)
- `bomberquiz-api`: **implementado em 2026-07-07** (`.github/workflows/deploy.yml`) — hoje roda `typecheck` → `deploy` (push em `main` → app produção via `flyctl deploy`); `lint` → unit → integration ficam como próximo passo, pendentes até essas ferramentas existirem no repo (ver `tarefas.md`).
- `bomberquiz-web`: lint → typecheck → unit → build → e2e (opcional) → deploy. Ainda não implementado.

### Custos esperados (free tiers no início)
- Fly.io: $0 (1 VM pequena, só `bomberquiz-api` — staging aposentado, ADR-0037; ver ADR-0027 sobre reduzir do padrão HA de 2 máquinas/app do Fly).
- Neon: $0 (0.5 GB).
- Cloudflare Pages: $0 (sites ilimitados).
- Cloudflare R2: $0 (até 10 GB).
- Cloudinary: $0 (free tier — 25 GB / 25k transformações/mês).
- Resend: $0 (até 3k e-mails/mês).
- Sentry: $0 (free tier, ~5k eventos/mês — dimensionado contra os 300–2.000 usuários esperados, ver ADR-0031).
- Mercado Pago: % por transação (sem custo fixo).
- WhatsApp Cloud API: $0 (até 1k conversas/mês na camada inicial).

Total mensal inicial estimado: **R$ 0** (custos só aparecem quando há receita de assinaturas pagando taxa do gateway).

### Ferramenta interna: `ai-bot` (fora do MVP)

Substitui operacionalmente o pipeline de geração de questões por IA retirado de produção em 2026-07-27 (Módulo 7, ver ADR-0039 em `decisoes.md`). **Não é módulo do MVP nem parte da spec de requisitos** — é tooling interno, documentado aqui só como referência técnica.

- Vive em `bot/` (`bot/api` + `bot/web`), repositório git **separado** de `bomberquiz-api`/`bomberquiz-web`/`requisitos` — fora do escopo deste diretório de documentação.
- Roda **apenas na máquina local do admin**; nunca é implantado (sem Fly.io/Cloudflare). CORS restrito a `localhost`.
- Mesmo stack do backend/frontend principal (Bun + Hono no `api`, Vite + React no `web`), mais `@anthropic-ai/sdk` e `unpdf` (extração de texto de PDF) no lado `api`.
- **Auth:** nenhum sistema próprio — autentica como admin contra o `bomberquiz-api` de produção via login real (a mesma barreira de acesso já existente), mantendo a sessão só em memória.
- **Publicação:** questões aprovadas são enviadas via `POST /admin/questions`, endpoint que já existia — nenhuma mudança foi necessária em `bomberquiz-api` para viabilizar isso. Perguntas publicadas por essa via recebem `source="ai_bot"` (ver `rf/content-admin.md` § Estrutura de dados).
