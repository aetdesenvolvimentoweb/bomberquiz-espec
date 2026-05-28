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

### Modelo de dados (esboço inicial)

> Detalhamento e migrations vão no repo do backend. Esta é a visão conceitual.

- `users` — id, name, email (unique), phone, dob, sex, password_hash, avatar_url, email_verified_at, role, created_at.
- `axes` — id, name (unique). _Eixos Temáticos._
- `subjects` — id, axis_id, name, source, expected_question_count, created_at.
- `questions` — id, subject_id, statement, options (jsonb), correct_options, justification, source_reference, image_url, status, author_id, created_at.
- `question_stats` — question_id, correct_count, wrong_count, last_updated.
- `quizzes` — id, user_id, scope (enum), config (jsonb), started_at, finished_at, score.
- `quiz_questions` — quiz_id, question_id, user_answer, correct, time_spent_ms.
- `subscriptions` — id, user_id, plan, status, started_at, expires_at.
- `subscription_donations` — id, beneficiary_id, granted_by_id, period_days, reason_category, reason_note, granted_at.
- `payments` — id, user_id, subscription_id, gateway, gateway_id, amount, status, paid_at.

### Configuração e secrets

- `.env` validado via Zod em `config/env.ts`. App não inicia se faltar variável.
- Secrets em produção via Fly.io secrets (`fly secrets set`).
- Whitelist de admins é um array em `config/admin-whitelist.ts` versionado, **não** banco.

### Testes

- **Unit** (`tests/unit/`): domínio e use cases, com portas fakes/mocks em memória. Rápidos (<5s suite inteira).
- **Integration** (`tests/integration/`): use cases conectados a infra real (Drizzle + Neon branch dedicado por job CI; ou Postgres em Docker para local). Cada teste roda em transação revertida ao final.
- **E2E** (`tests/e2e/`): HTTP black-box contra app inicializado, banco real.

### Observabilidade (futuro)

- Logs estruturados (pino).
- Erros para Sentry (free tier).
- Métricas básicas via Fly.io.

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
- Atualização do cliente é uma etapa no CI: baixar a spec do backend deployado em staging e regerar tipos.

---

## Operação

### Ambientes
- **Local:** Bun roda backend; Vite roda frontend; Postgres em Docker (ou branch dev no Neon).
- **Staging:** branch `staging` em ambos os repos → deploy automático (Fly.io app `bomberquiz-api-staging`; Cloudflare Pages preview).
- **Produção:** branch `main` → deploy automático após CI verde.

### CI/CD (GitHub Actions)
- `bomberquiz-api`: lint → typecheck → unit → integration (contra branch Neon de teste) → deploy.
- `bomberquiz-web`: lint → typecheck → unit → build → e2e (opcional) → deploy.

### Custos esperados (free tiers no início)
- Fly.io: $0 (3 VMs pequenas).
- Neon: $0 (0.5 GB).
- Cloudflare Pages: $0 (sites ilimitados).
- Cloudflare R2: $0 (até 10 GB).
- Cloudinary: $0 (free tier — 25 GB / 25k transformações/mês).
- Resend: $0 (até 3k e-mails/mês).
- Mercado Pago: % por transação (sem custo fixo).
- WhatsApp Cloud API: $0 (até 1k conversas/mês na camada inicial).

Total mensal inicial estimado: **R$ 0** (custos só aparecem quando há receita de assinaturas pagando taxa do gateway).
