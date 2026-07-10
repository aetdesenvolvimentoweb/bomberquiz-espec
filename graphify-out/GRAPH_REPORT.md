# Graph Report - .  (2026-07-10)

## Corpus Check
- 5 files · ~66,660 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 903 nodes · 1130 edges · 132 communities (80 shown, 52 thin omitted)
- Extraction: 98% EXTRACTED · 2% INFERRED · 0% AMBIGUOUS · INFERRED: 25 edges (avg confidence: 0.77)
- Token cost: 103,474 input · 0 output

## Community Hubs (Navigation)
- Documentação e Módulos do Espec
- Use Cases de Perfil (Auth)
- Entidade User (Domínio)
- Forgot-Password e Validação HTTP
- Bootstrap da API + Testes Auth
- Dependências Backend (package.json)
- Dependências Frontend (UI/forms)
- DevDependencies Frontend (build/test)
- Use Cases Change-Password/Deactivate/Reset
- Use Cases Email-Change/Delete/Env
- Hooks React Query — Perfil
- tsconfig.app.json (web)
- Hooks React Query — Auth
- tsconfig.json (web)
- Use Cases Register/GetMe/UpdateProfile
- M7 Geração por IA — ADRs
- Logout, Admin Whitelist, Auth Middleware
- components.json (shadcn config)
- tsconfig.node.json (web)
- Rate Limiting de Login
- Tabelas de Assinatura/Cortesia
- Tabelas de Conteúdo (Quiz)
- Sessão Única e LGPD (ADRs)
- RFs de Conteúdo Admin/Parceiro
- Deploy: Fly.io + Cloudflare Pages
- RFs Transversais (Quiz/Perfil/Conteúdo)
- Seções de Perfil (Web UI)
- Login Use Case + Erros
- Integrações Externas (Resend/MP/R2)
- Módulo 7 — IA (RFs)
- Assinaturas — Planos e Checkout
- Guards de Sessão (Web)
- Envelope de Erro (API Client)
- AlertDialog (shadcn component)
- ResendEmailAdapter
- RFs de Papéis e Conta
- RFs de Quiz (submeter/resultado)
- Backup, Rate Limit, Jobs (ADRs)
- Estrutura da Pergunta (domínio)
- Card (shadcn component)
- Schemas Comuns (Zod)
- OpenAPI Schema Gerado (web)
- CI: Postgres + Testes
- App Root + Router (web)
- Camadas Hexagonais (backend)
- Button (shadcn component)
- Config de Testes (Docker/CI)
- Testes do Repositório de Usuário
- Access Status (Assinatura)
- Input (shadcn component)
- Label (shadcn component)
- Verify Email Page (web)
- Email Confirm Page (web)
- tsconfig.json (api)
- normalize-email.ts
- request-id.ts
- user.entity.test.ts
- Arquitetura Hexagonal (ADR-0011)
- Upload de Imagem (RFs)
- Finalização de Quiz (RFs)
- Produto BomberQuiz / TAP CBMGO
- AuthLayout (web)
- LoadingScreen (web)
- Checkbox (shadcn component)
- Cliente HTTP (web)
- utils.ts cn() (web)
- LoginPage (web)
- RegisterPage (web)
- ResendVerificationPage (web)
- ResetPasswordPage (web)
- PrivacyPage (web)
- TermsPage (web)
- ConsentRenewalPage (web)
- Política de Sessão Única (conceito)
- AUTH-RF-011 Consentimento LGPD
- CONT-RF-003 Editar Eixo
- CONT-RF-005 Listar Matérias
- CONT-RF-007 Editar Matéria
- CONT-RF-008 Arquivar Matéria
- PART-RF-001 Listar Perguntas Próprias
- PROF-RF-001 Ver Perfil
- PROF-RF-002 Editar Dados
- PROF-RF-003 Trocar Senha
- PROF-RF-005 Gerenciar Avatar
- PROF-RF-006 Reaceite de Termos
- PROF-RF-008 Reativar Conta
- QUIZ-RF-006 Listar Quizzes
- SUB-RF-005 Status da Assinatura
- SUB-RF-006 Histórico de Pagamentos
- SUB-RF-007 Lembretes de Expiração
- Módulo 5 — Quiz Cliente
- Stack Backend (conceito)
- postcss.config.js
- Ícone PWA 192x192
- Ícone PWA 512x512
- main.tsx (web)
- vite-env.d.ts
- tailwind.config.ts
- setup.ts (testes web)
- initials.test.ts
- Assinaturas RF-007
- ADR-0017 Referência
- Stack Backend
- Ícone PWA 192px
- Ícone PWA 512px

## God Nodes (most connected - your core abstractions)
1. `User` - 42 edges
2. `IUserRepository` - 25 edges
3. `compilerOptions` - 18 edges
4. `scripts` - 17 edges
5. `compilerOptions` - 15 edges
6. `registerVerifiedUser()` - 15 edges
7. `Especificação de requisitos — BomberQuiz` - 14 edges
8. `audit_log (tabela de auditoria)` - 14 edges
9. `Contrato da API — BomberQuiz` - 13 edges
10. `postJson()` - 13 edges

## Surprising Connections (you probably didn't know these)
- `api/.github/workflows/deploy.yml` --semantically_similar_to--> `web/.github/workflows/deploy.yml`  [INFERRED] [semantically similar]
  espec/docs/tarefas.md → web/.github/workflows/deploy.yml
- `index.html — SPA entry point (bomberquiz-web)` --shares_data_with--> `Stack frontend (Vite+React+Tailwind PWA)`  [INFERRED]
  web/index.html → espec/docs/requisitos.md
- `QUIZ-RF-008 — Reset de estatísticas do cliente` --semantically_similar_to--> `CONT-RF-011 — Editar pergunta`  [INFERRED] [semantically similar]
  espec/docs/rf/quiz.md → espec/docs/rf/content-admin.md
- `web/.github/workflows/deploy.yml` --shares_data_with--> `api.bomberquiz.com.br (Fly.io)`  [EXTRACTED]
  web/.github/workflows/deploy.yml → espec/docs/runbook-dominio-cloudflare.md
- `env VITE_API_BASE_URL` --shares_data_with--> `api.bomberquiz.com.br (Fly.io)`  [EXTRACTED]
  web/.github/workflows/deploy.yml → espec/docs/runbook-dominio-cloudflare.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Migração de domínio Fase 2 (bomberquiz.com.br)** — espec_docs_runbook_dominio_cloudflare_doc, espec_docs_tarefas_adr_0028, espec_docs_runbook_dominio_cloudflare_bomberquizcombr, web__github_workflows_deploy, api_src_infra_auth_better_auth [INFERRED 0.85]
- **Pipelines de CI/CD de deploy (api e web)** — web__github_workflows_deploy, api__github_workflows_deploy, espec_docs_tarefas_flyio, espec_docs_tarefas_cloudflare_pages [INFERRED 0.80]
- **Bugs encontrados na expansão de testes do módulo Auth (2026-07-09)** — espec_docs_tarefas_login_usecase_bug, espec_docs_tarefas_verifications_table_bug, espec_docs_tarefas_forgot_password_bug, espec_docs_tarefas_validation_422_bug, espec_docs_tarefas_password_list_bug [INFERRED 0.75]
- **Camadas da arquitetura hexagonal (domain/application/infra/http)** — espec_docs_arquitetura_domain_layer, espec_docs_arquitetura_application_layer, espec_docs_arquitetura_infra_layer, espec_docs_arquitetura_http_layer [EXTRACTED 1.00]
- **Premissa de instância única (rate limiting, scheduler, 1 máquina Fly)** — espec_docs_arquitetura_rate_limiting, espec_docs_arquitetura_scheduled_jobs, espec_docs_decisoes_adr_0027 [EXTRACTED 1.00]
- **Fluxo de geração de questões por IA (job -> worker -> aprovação)** — espec_docs_rf_ai_generation_aigen_rf_001, espec_docs_rf_ai_generation_aigen_rf_004, espec_docs_rf_ai_generation_aigen_rf_006, espec_docs_rf_ai_generation_ai_generation_jobs [EXTRACTED 1.00]
- **Otimização de custo via cache de exemplos + prompt caching** — espec_docs_rf_ai_generation_ai_reference_exams, espec_docs_decisoes_adr_0025, espec_docs_decisoes_adr_0026, espec_docs_rf_ai_generation_aigen_rf_004 [INFERRED 0.85]
- **Fluxo de submissão, revisão e aprovação de perguntas de parceiros** — espec_docs_rf_content_partner_part_rf_004, espec_docs_rf_content_admin_cont_rf_014, espec_docs_rf_content_admin_cont_rf_015, espec_docs_rf_content_admin_cont_rf_016, espec_docs_rubrica_aprovacao_doc [INFERRED 0.85]
- **Cálculo consolidado de acesso (trial+pago+cortesia) consumido por múltiplos módulos** — espec_docs_rf_subscriptions_sub_rf_011, espec_docs_rf_quiz_quiz_rf_009, espec_docs_rf_profile_prof_rf_014, espec_docs_rf_subscriptions_sub_rf_004, espec_docs_rf_subscriptions_sub_rf_008 [INFERRED 0.80]
- **Snapshot da pergunta no quiz — imunidade a edições futuras** — espec_docs_rf_quiz_quiz_rf_001, espec_docs_rf_quiz_quiz_session_questions, espec_docs_rf_quiz_quiz_rf_005, espec_docs_rf_content_admin_cont_rf_011, espec_docs_rf_quiz_quiz_rf_008 [INFERRED 0.80]

## Communities (132 total, 52 thin omitted)

### Community 0 - "Documentação e Módulos do Espec"
Cohesion: 0.06
Nodes (43): Módulo 7 — Geração de Questões por IA (API), Tabela audit_log, Tabela axes (Eixo Temático), Backup e recuperação (PITR + pg_dump), Cloudflare R2 (storage de imagens/backup), Estratégia de domínios/URLs/cookies (2 fases), Mercado Pago, Tabela question_stats (+35 more)

### Community 1 - "Use Cases de Perfil (Auth)"
Cohesion: 0.06
Nodes (5): GetMeOutput, User, DrizzleUserRepository, entityToRow(), rowToEntity()

### Community 2 - "Entidade User (Domínio)"
Cohesion: 0.06
Nodes (32): ForgotPasswordUseCase, zodValidationHook(), authRoutes, ForgotPasswordBodySchema, forgotPasswordUseCase, getMeUseCase, LoginBodySchema, loginUseCase (+24 more)

### Community 3 - "Forgot-Password e Validação HTTP"
Cohesion: 0.19
Nodes (18): app, server, postRegister(), authedRequest(), extractTokenFromEmail(), loginAndGetCookie(), postJson(), randomTestIp() (+10 more)

### Community 4 - "Bootstrap da API + Testes Auth"
Cohesion: 0.06
Nodes (32): dependencies, better-auth, drizzle-orm, hono, @hono/zod-openapi, postgres, resend, zod (+24 more)

### Community 5 - "Dependências Backend (package.json)"
Cohesion: 0.06
Nodes (29): dependencies, class-variance-authority, clsx, @hookform/resolvers, lucide-react, openapi-fetch, @radix-ui/react-checkbox, @radix-ui/react-dialog (+21 more)

### Community 6 - "Dependências Frontend (UI/forms)"
Cohesion: 0.07
Nodes (29): devDependencies, autoprefixer, jsdom, openapi-typescript, postcss, tailwindcss, @testing-library/jest-dom, @testing-library/react (+21 more)

### Community 7 - "DevDependencies Frontend (build/test)"
Cohesion: 0.13
Nodes (13): ChangePasswordInput, ChangePasswordUseCase, ResetPasswordInput, ResetPasswordUseCase, ConsentRenewalRequiredError, EmailAlreadyInUseError, EmailChangeCooldownError, ReauthRequiredError (+5 more)

### Community 8 - "Use Cases Change-Password/Deactivate/Reset"
Cohesion: 0.13
Nodes (16): DeleteAccountInput, envSchema, parsed, DeletionBlockedByEmailChangeError, DB, queryClient, UserRow, accounts (+8 more)

### Community 9 - "Use Cases Email-Change/Delete/Env"
Cohesion: 0.14
Nodes (7): AcceptConsentUseCase, DeactivateAccountInput, DeactivateAccountUseCase, GetMeUseCase, UpdateProfileUseCase, UserNotFoundError, IUserRepository

### Community 10 - "Hooks React Query — Perfil"
Cohesion: 0.10
Nodes (11): ChangePasswordFormValues, changePasswordSchema, DeleteAccountFormValues, deleteAccountSchema, passwordSchema, ReauthActionFormValues, reauthActionSchema, RequestEmailChangeFormValues (+3 more)

### Community 11 - "tsconfig.app.json (web)"
Cohesion: 0.10
Nodes (20): compilerOptions, allowImportingTsExtensions, isolatedModules, jsx, lib, module, moduleDetection, moduleResolution (+12 more)

### Community 12 - "Hooks React Query — Auth"
Cohesion: 0.12
Nodes (11): useForgotPassword(), useResendVerification(), ForgotPasswordFormValues, forgotPasswordSchema, LoginFormValues, loginSchema, passwordSchema, RegisterFormValues (+3 more)

### Community 13 - "tsconfig.json (web)"
Cohesion: 0.11
Nodes (17): compilerOptions, allowImportingTsExtensions, allowSyntheticDefaultImports, baseUrl, ignoreDeprecations, lib, module, moduleDetection (+9 more)

### Community 14 - "Use Cases Register/GetMe/UpdateProfile"
Cohesion: 0.14
Nodes (18): CONT-RF-004 — Arquivar/desarquivar eixo, Módulo 3 — Conteúdo (admin), Módulo 4 — Conteúdo (parceiro), ADR-0014 — Política de sessão única, ADR-0015 — Anonimização LGPD, Módulo 2 — Perfil e papéis, PROF-RF-007 — Desativar conta, PROF-RF-010 — Política de sessão única (+10 more)

### Community 15 - "M7 Geração por IA — ADRs"
Cohesion: 0.12
Nodes (11): LogoutUseCase, ContextVariableMap, hono, requireAuth, emailAdapter, ADR-0005, ADR-0018, Bug: links de e-mail com paths em inglês causavam 404 (+3 more)

### Community 16 - "Logout, Admin Whitelist, Auth Middleware"
Cohesion: 0.13
Nodes (14): aliases, components, ui, utils, rsc, $schema, style, tailwind (+6 more)

### Community 17 - "components.json (shadcn config)"
Cohesion: 0.14
Nodes (13): compilerOptions, allowImportingTsExtensions, isolatedModules, lib, module, moduleDetection, moduleResolution, noEmit (+5 more)

### Community 18 - "tsconfig.node.json (web)"
Cohesion: 0.18
Nodes (9): checkLimit(), LOCKOUT_STEPS, LoginAttemptEntry, loginAttempts, rateLimitByIp(), rateLimitByUser(), WindowEntry, windows (+1 more)

### Community 19 - "Rate Limiting de Login"
Cohesion: 0.18
Nodes (3): ConfirmEmailChangeUseCase, DeleteAccountUseCase, IEmailPort

### Community 20 - "Tabelas de Assinatura/Cortesia"
Cohesion: 0.23
Nodes (12): Especificação de requisitos — BomberQuiz, BomberQuiz (quiz TAP CBMGO), Cortesia de assinatura, Eixo Temático (nível 1), Matéria (nível 2), Administrador (papel), Cliente (papel), Parceiro (papel) (+4 more)

### Community 21 - "Tabelas de Conteúdo (Quiz)"
Cohesion: 0.24
Nodes (12): audit_log (tabela de auditoria), CONT-RF-006 — Criar matéria, PROF-RF-004 — Trocar e-mail, PROF-RF-009 — Excluir conta (LGPD), QUIZ-RF-001 — Iniciar um quiz, quiz_sessions (tabela), courtesies (tabela), SUB-RF-008 — Conceder cortesia de assinatura (+4 more)

### Community 22 - "Sessão Única e LGPD (ADRs)"
Cohesion: 0.26
Nodes (12): CONT-RF-009 — Listar perguntas, CONT-RF-011 — Editar pergunta, CONT-RF-012 — Arquivar/desarquivar/excluir pergunta, CONT-RF-014 — Fila de revisão do admin, CONT-RF-015 — Aprovar pergunta de parceiro, CONT-RF-016 — Rejeitar pergunta de parceiro, PART-RF-003 — Editar pergunta própria, PART-RF-004 — Enviar pergunta para revisão (+4 more)

### Community 23 - "RFs de Conteúdo Admin/Parceiro"
Cohesion: 0.17
Nodes (11): ADR-0018 — fronteira Better-Auth × custom, com spike obrigatório, Better-Auth, Repositório bomberquiz-api, Bun, Drizzle ORM, Bug: esqueci-senha chamava endpoint inexistente do Better-Auth, Hono, Achado: rateLimitByUser é código morto (+3 more)

### Community 24 - "Deploy: Fly.io + Cloudflare Pages"
Cohesion: 0.23
Nodes (4): ChangeEmailSection(), ChangePasswordSection(), DangerZoneSection(), PersonalInfoSection()

### Community 25 - "RFs Transversais (Quiz/Perfil/Conteúdo)"
Cohesion: 0.24
Nodes (7): RegisterUserInput, UpdateProfileInput, UserProps, UserRole, UserSex, UserStatus, ROLE_HIERARCHY

### Community 26 - "Seções de Perfil (Web UI)"
Cohesion: 0.18
Nodes (11): CLAUDE.md (espec guidance), bomberquiz-api (repo, backend), bomberquiz-web (repo, frontend), Contrato da API — BomberQuiz, Módulo 2 — Perfil e papéis (API), Módulo 3 — Conteúdo admin (API), Módulo 4 — Conteúdo parceiro (API), Módulo 5 — Quiz (API) (+3 more)

### Community 27 - "Login Use Case + Erros"
Cohesion: 0.24
Nodes (11): Módulo 1 — Autenticação e cadastro (API), Better-Auth, Tabela sessions, ADR-0014 Política de sessão única, ADR-0018 Fronteira Better-Auth × autenticação customizada, Módulo 1 — Autenticação e cadastro (RF), AUTH-RF-003: Reenvio de verificação, AUTH-RF-005: Logout (+3 more)

### Community 28 - "Integrações Externas (Resend/MP/R2)"
Cohesion: 0.18
Nodes (11): Admin whitelist (config/admin-whitelist.ts), Tabela coupons, Tabela courtesies, Tabela payments, Tabela subscriptions, Tabela user_access, ADR-0005 Administradores via whitelist de e-mails, ADR-0006 Cortesia de assinatura (+3 more)

### Community 29 - "Módulo 7 — IA (RFs)"
Cohesion: 0.20
Nodes (10): api/.env.example, app.bomberquiz.com.br (Cloudflare Pages), EMAIL_FROM, send.bomberquiz.com.br (Resend), Divergência resolvida: www. vs app. como origem canônica, www.bomberquiz.com.br (redirect 301), Zona Cloudflare bomberquiz.com.br, ADR-0028 — domínio .com.br oficial, .com redirect, e-mail isolado em send. (+2 more)

### Community 30 - "Assinaturas — Planos e Checkout"
Cohesion: 0.20
Nodes (10): bomberquiz-api (repo backend), bomberquiz-web (repo frontend PWA), Cloudflare Pages (hospedagem frontend), Fly.io (hospedagem backend), Postgres no Neon, Comportamento PWA (service worker), ADR-0008 Dois repositórios separados, ADR-0009 Stack do backend (+2 more)

### Community 31 - "Guards de Sessão (Web)"
Cohesion: 0.27
Nodes (10): coupons (tabela), Mercado Pago (gateway de pagamento), payments (tabela), SUB-RF-001 — Configurar e listar planos, SUB-RF-002 — Iniciar trial automático, SUB-RF-003 — Iniciar checkout, SUB-RF-004 — Webhook Mercado Pago, SUB-RF-013 — Gerenciar cupons de desconto (+2 more)

### Community 32 - "Envelope de Erro (API Client)"
Cohesion: 0.20
Nodes (10): API_BASE_URL, api.bomberquiz.com.br (Fly.io), Achado: cache negativo de DNS local atrasou validação de api., Step: Build (produção), Step: Build (staging), Job ci, secrets CLOUDFLARE_API_TOKEN / CLOUDFLARE_ACCOUNT_ID, Step: Deploy to Cloudflare Pages (+2 more)

### Community 33 - "AlertDialog (shadcn component)"
Cohesion: 0.31
Nodes (8): ConsentGate(), RequireAuth(), RequireGuest(), fetchSession(), SESSION_QUERY_KEY, SessionData, SessionUser, useSession()

### Community 34 - "ResendEmailAdapter"
Cohesion: 0.27
Nodes (8): ApiError, ApiErrorField, ErrorEnvelope, ErrorEnvelopeSchema, ErrorFieldSchema, parseErrorBody(), throwIfError(), unwrap()

### Community 35 - "RFs de Papéis e Conta"
Cohesion: 0.36
Nodes (9): api/.github/workflows/deploy.yml, Arquitetura — BomberQuiz (documento), Decisões — BomberQuiz (ADRs), Runbook — Configurar Cloudflare para bomberquiz.com.br, Webhook Mercado Pago (pendente), Bug: VITE_API_BASE_URL hardcoded no workflow do GitHub Actions, docs/tarefas.md, Mercado Pago (+1 more)

### Community 36 - "RFs de Quiz (submeter/resultado)"
Cohesion: 0.22
Nodes (5): LoginInput, LoginOutput, AccountInactiveError, EmailNotVerifiedError, InvalidCredentialsError

### Community 37 - "Backup, Rate Limit, Jobs (ADRs)"
Cohesion: 0.22
Nodes (6): AlertDialogAction, AlertDialogCancel, AlertDialogContent, AlertDialogDescription, AlertDialogOverlay, AlertDialogTitle

### Community 40 - "Schemas Comuns (Zod)"
Cohesion: 0.25
Nodes (8): CONT-RF-017 — Cálculo diário do nível de dificuldade, QUIZ-RF-002 — Submeter resposta a uma questão, QUIZ-RF-005 — Visualizar resultado de quiz, QUIZ-RF-007 — Painel de desempenho, QUIZ-RF-008 — Reset de estatísticas do cliente, QUIZ-RF-010 — Evolução temporal do desempenho, quiz_session_questions (tabela, snapshot), user_subject_stats (tabela agregada)

### Community 41 - "OpenAPI Schema Gerado (web)"
Cohesion: 0.33
Nodes (7): Cloudinary (avatares), Tabela user_subject_stats, Tabela users, ADR-0007 Cadastro único; papel é propriedade, ADR-0013 Avatares no Cloudinary, AUTH-RF-001: Cadastro de novo usuário, AUTH-RF-002: Verificação de e-mail

### Community 42 - "CI: Postgres + Testes"
Cohesion: 0.29
Nodes (7): ADR-0016 — Estrutura mínima da pergunta, CONT-RF-010 — Criar pergunta (admin), Eixo Temático (entidade de domínio), Matéria (entidade de domínio), Pergunta (entidade de domínio), PART-RF-002 — Criar pergunta (rascunho), Checklist objetivo de aprovação/rejeição

### Community 43 - "App Root + Router (web)"
Cohesion: 0.29
Nodes (6): Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle

### Community 44 - "Camadas Hexagonais (backend)"
Cohesion: 0.33
Nodes (4): ErrorFieldSchema, ErrorResponseSchema, PaginationQuerySchema, UserProfileSchema

### Community 45 - "Button (shadcn component)"
Cohesion: 0.33
Nodes (5): components, $defs, operations, paths, webhooks

### Community 46 - "Config de Testes (Docker/CI)"
Cohesion: 0.40
Nodes (5): postgres-test service, CI job (typecheck + tests), Deploy job (flyctl to Fly.io), Postgres service container (CI), Integration tests (transaction rollback)

### Community 48 - "Access Status (Assinatura)"
Cohesion: 0.40
Nodes (5): ADR-0021 M7 Geração por IA: escopo e limitação a PDFs selecionáveis, ADR-0022 Modelo LLM: Claude (Anthropic), ADR-0026 Prompt caching via cache_control, Claude (Anthropic) API — geração de questões, Módulo 7 — Geração de Questões por IA (RF)

### Community 49 - "Input (shadcn component)"
Cohesion: 0.40
Nodes (5): Achado: app.bomberquiz.com sem registro DNS na zona .com, bomberquiz.com (segunda zona, redirect), Domínio bomberquiz.com.br, COOKIE_DOMAIN, Hostinger Business Email

### Community 50 - "Label (shadcn component)"
Cohesion: 0.40
Nodes (5): ADR-0025 — cache de exemplos de prova de referência (ai_reference_exams), ADR-0026 — prompt caching ephemeral para reduzir custo de tokens, Módulo 7 — Geração de Questões por IA, Tabela ai_reference_exams, Claude API (geração de questões)

### Community 52 - "Email Confirm Page (web)"
Cohesion: 0.67
Nodes (4): application/ (use cases), domain/ (regras de negócio puras), http/ (Hono routes, middleware), infra/ (adapters concretos)

### Community 53 - "tsconfig.json (api)"
Cohesion: 0.50
Nodes (4): ADR-0020 — backup PITR Neon + dump lógico para R2, ADR-0027 — staging/prod compartilham banco Neon; 1 máquina Fly por app, Fly.io, Neon (Postgres)

### Community 54 - "normalize-email.ts"
Cohesion: 0.50
Nodes (3): Button, ButtonProps, buttonVariants

### Community 55 - "request-id.ts"
Cohesion: 0.67
Nodes (3): docker-compose.test.yml (Postgres test), Deploy Workflow (GitHub Actions), Tests README (api)

### Community 58 - "Upload de Imagem (RFs)"
Cohesion: 0.67
Nodes (3): QUIZ-RF-009 — Bloqueio de acesso por falta de assinatura, SUB-RF-011 — Cálculo de access_status, user_access (tabela derivada)

## Ambiguous Edges - Review These
- `better-auth.ts` → `Achado: troca de senha logado não dispara alerta de e-mail`  [AMBIGUOUS]
  espec/docs/tarefas.md · relation: conceptually_related_to

## Knowledge Gaps
- **368 isolated node(s):** `ChangePasswordInput`, `DeactivateAccountInput`, `DeleteAccountInput`, `LoginInput`, `LoginOutput` (+363 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **52 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `better-auth.ts` and `Achado: troca de senha logado não dispara alerta de e-mail`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **Why does `User` connect `Use Cases de Perfil (Auth)` to `Estrutura da Pergunta (domínio)`, `DevDependencies Frontend (build/test)`, `Use Cases Change-Password/Deactivate/Reset`, `Use Cases Email-Change/Delete/Env`, `Hooks React Query — Auth`, `RFs Transversais (Quiz/Perfil/Conteúdo)`?**
  _High betweenness centrality (0.036) - this node is a cross-community bridge._
- **Why does `WEB_ORIGIN` connect `M7 Geração por IA — ADRs` to `Forgot-Password e Validação HTTP`?**
  _High betweenness centrality (0.028) - this node is a cross-community bridge._
- **Why does `Repositório bomberquiz-api` connect `RFs de Conteúdo Admin/Parceiro` to `tsconfig.json (api)`, `Módulo 7 — IA (RFs)`?**
  _High betweenness centrality (0.021) - this node is a cross-community bridge._
- **What connects `ChangePasswordInput`, `DeactivateAccountInput`, `DeleteAccountInput` to the rest of the system?**
  _392 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Documentação e Módulos do Espec` be split into smaller, more focused modules?**
  _Cohesion score 0.06090808416389812 - nodes in this community are weakly interconnected._
- **Should `Use Cases de Perfil (Auth)` be split into smaller, more focused modules?**
  _Cohesion score 0.06401137980085349 - nodes in this community are weakly interconnected._