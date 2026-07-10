# Graph Report - .  (2026-07-09)

## Corpus Check
- 28 files · ~64,796 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 865 nodes · 1151 edges · 121 communities (73 shown, 48 thin omitted)
- Extraction: 98% EXTRACTED · 2% INFERRED · 0% AMBIGUOUS · INFERRED: 19 edges (avg confidence: 0.74)
- Token cost: 0 input · 123,016 output

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
- Arquitetura Hexagonal (ADR-0011)
- Upload de Imagem (RFs)
- Finalização de Quiz (RFs)
- Produto BomberQuiz / TAP CBMGO
- Checkbox (shadcn component)
- Cliente HTTP (web)
- env.ts (api)
- vite.config.ts (web)
- Testes E2E (conceito)
- Testes Unitários (conceito)
- CI/CD (GitHub Actions, conceito)
- Cliente HTTP Tipado (conceito)
- Contrato OpenAPI (conceito)
- RBAC (conceito)
- ADR-0001 Idioma PT-BR
- ADR-0002 Fase de Especificação
- Exclusão de Conta LGPD (conceito)
- Política de Sessão Única (conceito)
- AUTH-RF-011 Consentimento LGPD
- CONT-RF-001 Listar Eixos
- CONT-RF-002 Criar Eixo
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
- Ícone PWA 192x192
- Ícone PWA 512x512

## God Nodes (most connected - your core abstractions)
1. `User` - 42 edges
2. `IUserRepository` - 27 edges
3. `compilerOptions` - 18 edges
4. `scripts` - 17 edges
5. `compilerOptions` - 15 edges
6. `registerVerifiedUser()` - 15 edges
7. `IEmailPort` - 14 edges
8. `Especificação de requisitos — BomberQuiz` - 14 edges
9. `audit_log (tabela de auditoria)` - 14 edges
10. `Contrato da API — BomberQuiz` - 13 edges

## Surprising Connections (you probably didn't know these)
- `Deploy workflow (bomberquiz-web CI/CD)` --conceptually_related_to--> `Stack frontend (Vite+React+Tailwind PWA)`  [INFERRED]
  web/.github/workflows/deploy.yml → espec/docs/requisitos.md
- `index.html — SPA entry point (bomberquiz-web)` --shares_data_with--> `Stack frontend (Vite+React+Tailwind PWA)`  [INFERRED]
  web/index.html → espec/docs/requisitos.md
- `QUIZ-RF-008 — Reset de estatísticas do cliente` --semantically_similar_to--> `CONT-RF-011 — Editar pergunta`  [INFERRED] [semantically similar]
  espec/docs/rf/quiz.md → espec/docs/rf/content-admin.md
- `Módulo 1 — Autenticação e cadastro (API)` --references--> `AUTH-RF-004: Login com e-mail e senha`  [EXTRACTED]
  espec/docs/api.md → espec/docs/rf/auth.md
- `ai_generation_jobs table` --semantically_similar_to--> `Tabela quiz_sessions`  [INFERRED] [semantically similar]
  espec/docs/rf/ai-generation.md → espec/docs/arquitetura.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Camadas da arquitetura hexagonal (domain/application/infra/http)** — espec_docs_arquitetura_domain_layer, espec_docs_arquitetura_application_layer, espec_docs_arquitetura_infra_layer, espec_docs_arquitetura_http_layer [EXTRACTED 1.00]
- **Otimizações de custo/performance do Módulo 7 (cache de exemplos + prompt caching)** — espec_docs_decisoes_adr_0025, espec_docs_decisoes_adr_0026, espec_docs_decisoes_ai_reference_exams_table, espec_docs_decisoes_ai_generation_jobs_table, espec_docs_tarefas_modulo7_ia_generation [EXTRACTED 1.00]
- **Premissa de instância única (rate limiting, scheduler, 1 máquina Fly)** — espec_docs_arquitetura_rate_limiting, espec_docs_arquitetura_scheduled_jobs, espec_docs_decisoes_adr_0027 [EXTRACTED 1.00]
- **Fluxo de geração de questões por IA (job -> worker -> aprovação)** — espec_docs_rf_ai_generation_aigen_rf_001, espec_docs_rf_ai_generation_aigen_rf_004, espec_docs_rf_ai_generation_aigen_rf_006, espec_docs_rf_ai_generation_ai_generation_jobs [EXTRACTED 1.00]
- **Otimização de custo via cache de exemplos + prompt caching** — espec_docs_rf_ai_generation_ai_reference_exams, espec_docs_decisoes_adr_0025, espec_docs_decisoes_adr_0026, espec_docs_rf_ai_generation_aigen_rf_004 [INFERRED 0.85]
- **Fluxo de submissão, revisão e aprovação de perguntas de parceiros** — espec_docs_rf_content_partner_part_rf_004, espec_docs_rf_content_admin_cont_rf_014, espec_docs_rf_content_admin_cont_rf_015, espec_docs_rf_content_admin_cont_rf_016, espec_docs_rubrica_aprovacao_doc [INFERRED 0.85]
- **Cálculo consolidado de acesso (trial+pago+cortesia) consumido por múltiplos módulos** — espec_docs_rf_subscriptions_sub_rf_011, espec_docs_rf_quiz_quiz_rf_009, espec_docs_rf_profile_prof_rf_014, espec_docs_rf_subscriptions_sub_rf_004, espec_docs_rf_subscriptions_sub_rf_008 [INFERRED 0.80]
- **Snapshot da pergunta no quiz — imunidade a edições futuras** — espec_docs_rf_quiz_quiz_rf_001, espec_docs_rf_quiz_quiz_session_questions, espec_docs_rf_quiz_quiz_rf_005, espec_docs_rf_content_admin_cont_rf_011, espec_docs_rf_quiz_quiz_rf_008 [INFERRED 0.80]

## Communities (121 total, 48 thin omitted)

### Community 0 - "Documentação e Módulos do Espec"
Cohesion: 0.06
Nodes (48): CLAUDE.md (espec guidance), bomberquiz-api (repo, backend), bomberquiz-web (repo, frontend), Contrato da API — BomberQuiz, Módulo 1 — Autenticação e cadastro (API), Módulo 2 — Perfil e papéis (API), Módulo 3 — Conteúdo admin (API), Módulo 4 — Conteúdo parceiro (API) (+40 more)

### Community 1 - "Use Cases de Perfil (Auth)"
Cohesion: 0.08
Nodes (11): AcceptConsentUseCase, ConfirmEmailChangeUseCase, DeactivateAccountUseCase, DeleteAccountUseCase, GetMeUseCase, LoginUseCase, RegisterUserUseCase, RequestEmailChangeUseCase (+3 more)

### Community 2 - "Entidade User (Domínio)"
Cohesion: 0.06
Nodes (5): GetMeOutput, User, DrizzleUserRepository, entityToRow(), rowToEntity()

### Community 3 - "Forgot-Password e Validação HTTP"
Cohesion: 0.06
Nodes (31): ForgotPasswordUseCase, zodValidationHook(), authRoutes, ForgotPasswordBodySchema, forgotPasswordUseCase, getMeUseCase, LoginBodySchema, loginUseCase (+23 more)

### Community 4 - "Bootstrap da API + Testes Auth"
Cohesion: 0.19
Nodes (18): app, server, postRegister(), authedRequest(), extractTokenFromEmail(), loginAndGetCookie(), postJson(), randomTestIp() (+10 more)

### Community 5 - "Dependências Backend (package.json)"
Cohesion: 0.06
Nodes (32): dependencies, better-auth, drizzle-orm, hono, @hono/zod-openapi, postgres, resend, zod (+24 more)

### Community 6 - "Dependências Frontend (UI/forms)"
Cohesion: 0.06
Nodes (29): dependencies, class-variance-authority, clsx, @hookform/resolvers, lucide-react, openapi-fetch, @radix-ui/react-checkbox, @radix-ui/react-dialog (+21 more)

### Community 7 - "DevDependencies Frontend (build/test)"
Cohesion: 0.07
Nodes (29): devDependencies, autoprefixer, jsdom, openapi-typescript, postcss, tailwindcss, @testing-library/jest-dom, @testing-library/react (+21 more)

### Community 8 - "Use Cases Change-Password/Deactivate/Reset"
Cohesion: 0.11
Nodes (14): ChangePasswordInput, ChangePasswordUseCase, DeactivateAccountInput, ResetPasswordInput, ResetPasswordUseCase, ConsentRenewalRequiredError, DeletionBlockedByEmailChangeError, EmailChangeCooldownError (+6 more)

### Community 9 - "Use Cases Email-Change/Delete/Env"
Cohesion: 0.15
Nodes (16): DeleteAccountInput, RequestEmailChangeInput, envSchema, parsed, DB, queryClient, UserRow, accounts (+8 more)

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
Cohesion: 0.19
Nodes (8): RegisterUserInput, UpdateProfileInput, UserProps, UserRole, UserSex, UserStatus, EmailAlreadyInUseError, ROLE_HIERARCHY

### Community 15 - "M7 Geração por IA — ADRs"
Cohesion: 0.17
Nodes (16): Tabela quiz_sessions, ADR-0021 M7 Geração por IA: escopo e limitação a PDFs selecionáveis, ADR-0022 Modelo LLM: Claude (Anthropic), ADR-0023 Processamento de jobs assíncrono in-process, ADR-0024 PDFs temporários em R2 excluídos após job, ADR-0025 Cache de exemplos de prova de referência, ADR-0026 Prompt caching via cache_control, Tabela ai_generation_jobs (+8 more)

### Community 16 - "Logout, Admin Whitelist, Auth Middleware"
Cohesion: 0.14
Nodes (9): LogoutUseCase, ADMIN_WHITELIST, ADR-0005, ContextVariableMap, hono, requireAuth, emailAdapter, ADR-0005 (+1 more)

### Community 17 - "components.json (shadcn config)"
Cohesion: 0.13
Nodes (14): aliases, components, ui, utils, rsc, $schema, style, tailwind (+6 more)

### Community 18 - "tsconfig.node.json (web)"
Cohesion: 0.14
Nodes (13): compilerOptions, allowImportingTsExtensions, isolatedModules, lib, module, moduleDetection, moduleResolution, noEmit (+5 more)

### Community 19 - "Rate Limiting de Login"
Cohesion: 0.18
Nodes (9): checkLimit(), LOCKOUT_STEPS, LoginAttemptEntry, loginAttempts, rateLimitByIp(), rateLimitByUser(), WindowEntry, windows (+1 more)

### Community 20 - "Tabelas de Assinatura/Cortesia"
Cohesion: 0.21
Nodes (13): Admin whitelist (config/admin-whitelist.ts), Tabela coupons, Tabela courtesies, Tabela payments, Tabela subscriptions, Tabela user_access, ADR-0005 Administradores via whitelist de e-mails, ADR-0006 Cortesia de assinatura (+5 more)

### Community 21 - "Tabelas de Conteúdo (Quiz)"
Cohesion: 0.18
Nodes (13): Tabela axes (Eixo Temático), Tabela question_stats, Tabela questions, Tabela subjects (Matéria), Tabela user_subject_stats, Tabela users, ADR-0003 Nomenclatura "Eixo Temático", ADR-0004 Hierarquia Eixo → Matéria → Pergunta (+5 more)

### Community 22 - "Sessão Única e LGPD (ADRs)"
Cohesion: 0.17
Nodes (13): Arquitetura — BomberQuiz (documento), Decisões — BomberQuiz (ADRs), Módulo 3 — Conteúdo (admin), ADR-0014 — Política de sessão única, ADR-0015 — Anonimização LGPD, Módulo 2 — Perfil e papéis, PROF-RF-010 — Política de sessão única, Módulo 5 — Quiz (cliente) (+5 more)

### Community 23 - "RFs de Conteúdo Admin/Parceiro"
Cohesion: 0.23
Nodes (13): CONT-RF-009 — Listar perguntas, CONT-RF-011 — Editar pergunta, CONT-RF-012 — Arquivar/desarquivar/excluir pergunta, CONT-RF-014 — Fila de revisão do admin, CONT-RF-015 — Aprovar pergunta de parceiro, CONT-RF-016 — Rejeitar pergunta de parceiro, PART-RF-003 — Editar pergunta própria, PART-RF-004 — Enviar pergunta para revisão (+5 more)

### Community 24 - "Deploy: Fly.io + Cloudflare Pages"
Cohesion: 0.23
Nodes (12): bomberquiz-api (repo backend), bomberquiz-web (repo frontend PWA), Cloudflare Pages (hospedagem frontend), Fly.io (hospedagem backend), Postgres no Neon, Comportamento PWA (service worker), ADR-0008 Dois repositórios separados, ADR-0009 Stack do backend (+4 more)

### Community 25 - "RFs Transversais (Quiz/Perfil/Conteúdo)"
Cohesion: 0.24
Nodes (12): audit_log (tabela de auditoria), CONT-RF-006 — Criar matéria, PROF-RF-004 — Trocar e-mail, PROF-RF-009 — Excluir conta (LGPD), QUIZ-RF-001 — Iniciar um quiz, quiz_sessions (tabela), courtesies (tabela), SUB-RF-008 — Conceder cortesia de assinatura (+4 more)

### Community 26 - "Seções de Perfil (Web UI)"
Cohesion: 0.23
Nodes (4): ChangeEmailSection(), ChangePasswordSection(), DangerZoneSection(), PersonalInfoSection()

### Community 27 - "Login Use Case + Erros"
Cohesion: 0.18
Nodes (6): LoginInput, LoginOutput, AccountInactiveError, EmailNotVerifiedError, InvalidCredentialsError, UserNotFoundError

### Community 28 - "Integrações Externas (Resend/MP/R2)"
Cohesion: 0.31
Nodes (11): Cloudflare R2 (storage de imagens/backup), Estratégia de domínios/URLs/cookies (2 fases), Mercado Pago, Resend (e-mail transacional), SUPPORT_EMAIL / canal de suporte, Catálogo de e-mails transacionais, WhatsApp Cloud API, ADR-0012 Integrações externas iniciais (+3 more)

### Community 29 - "Módulo 7 — IA (RFs)"
Cohesion: 0.31
Nodes (10): Módulo 7 — Geração de Questões por IA (API), Tabela audit_log, ai_generated_questions table, AIGEN-RF-002: Consultar status (polling), AIGEN-RF-003: Listar histórico de jobs, AIGEN-RF-005: Editar questão gerada, AIGEN-RF-006: Aprovar questão gerada, AIGEN-RF-007: Descartar questão gerada (+2 more)

### Community 30 - "Assinaturas — Planos e Checkout"
Cohesion: 0.27
Nodes (10): coupons (tabela), Mercado Pago (gateway de pagamento), payments (tabela), SUB-RF-001 — Configurar e listar planos, SUB-RF-002 — Iniciar trial automático, SUB-RF-003 — Iniciar checkout, SUB-RF-004 — Webhook Mercado Pago, SUB-RF-013 — Gerenciar cupons de desconto (+2 more)

### Community 31 - "Guards de Sessão (Web)"
Cohesion: 0.31
Nodes (8): ConsentGate(), RequireAuth(), RequireGuest(), fetchSession(), SESSION_QUERY_KEY, SessionData, SessionUser, useSession()

### Community 32 - "Envelope de Erro (API Client)"
Cohesion: 0.27
Nodes (8): ApiError, ApiErrorField, ErrorEnvelope, ErrorEnvelopeSchema, ErrorFieldSchema, parseErrorBody(), throwIfError(), unwrap()

### Community 33 - "AlertDialog (shadcn component)"
Cohesion: 0.22
Nodes (6): AlertDialogAction, AlertDialogCancel, AlertDialogContent, AlertDialogDescription, AlertDialogOverlay, AlertDialogTitle

### Community 35 - "RFs de Papéis e Conta"
Cohesion: 0.29
Nodes (8): CONT-RF-004 — Arquivar/desarquivar eixo, Módulo 4 — Conteúdo (parceiro), PROF-RF-007 — Desativar conta, PROF-RF-011 — Buscar/listar usuários (admin), PROF-RF-012 — Promover Cliente a Parceiro, PROF-RF-013 — Revogar papel de Parceiro, PROF-RF-014 — Detalhe de usuário (admin), Módulo 6 — Assinaturas e cortesias

### Community 36 - "RFs de Quiz (submeter/resultado)"
Cohesion: 0.25
Nodes (8): CONT-RF-017 — Cálculo diário do nível de dificuldade, QUIZ-RF-002 — Submeter resposta a uma questão, QUIZ-RF-005 — Visualizar resultado de quiz, QUIZ-RF-007 — Painel de desempenho, QUIZ-RF-008 — Reset de estatísticas do cliente, QUIZ-RF-010 — Evolução temporal do desempenho, quiz_session_questions (tabela, snapshot), user_subject_stats (tabela agregada)

### Community 37 - "Backup, Rate Limit, Jobs (ADRs)"
Cohesion: 0.29
Nodes (7): Backup e recuperação (PITR + pg_dump), Rate limiting (duas camadas), Jobs agendados (scheduler in-process), ADR-0017 Jobs agendados via scheduler in-process, ADR-0020 Backup e recuperação do Postgres, AUTH-RF-009: Rate limiting de autenticação, Código morto: rateLimitByUser não usado em nenhuma rota

### Community 38 - "Estrutura da Pergunta (domínio)"
Cohesion: 0.29
Nodes (7): ADR-0016 — Estrutura mínima da pergunta, CONT-RF-010 — Criar pergunta (admin), Eixo Temático (entidade de domínio), Matéria (entidade de domínio), Pergunta (entidade de domínio), PART-RF-002 — Criar pergunta (rascunho), Checklist objetivo de aprovação/rejeição

### Community 39 - "Card (shadcn component)"
Cohesion: 0.29
Nodes (6): Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle

### Community 40 - "Schemas Comuns (Zod)"
Cohesion: 0.33
Nodes (4): ErrorFieldSchema, ErrorResponseSchema, PaginationQuerySchema, UserProfileSchema

### Community 41 - "OpenAPI Schema Gerado (web)"
Cohesion: 0.33
Nodes (5): components, $defs, operations, paths, webhooks

### Community 42 - "CI: Postgres + Testes"
Cohesion: 0.40
Nodes (5): postgres-test service, CI job (typecheck + tests), Deploy job (flyctl to Fly.io), Postgres service container (CI), Integration tests (transaction rollback)

### Community 44 - "Camadas Hexagonais (backend)"
Cohesion: 0.67
Nodes (4): application/ (use cases), domain/ (regras de negócio puras), http/ (Hono routes, middleware), infra/ (adapters concretos)

### Community 45 - "Button (shadcn component)"
Cohesion: 0.50
Nodes (3): Button, ButtonProps, buttonVariants

### Community 46 - "Config de Testes (Docker/CI)"
Cohesion: 0.67
Nodes (3): docker-compose.test.yml (Postgres test), Deploy Workflow (GitHub Actions), Tests README (api)

### Community 48 - "Access Status (Assinatura)"
Cohesion: 0.67
Nodes (3): QUIZ-RF-009 — Bloqueio de acesso por falta de assinatura, SUB-RF-011 — Cálculo de access_status, user_access (tabela derivada)

## Ambiguous Edges - Review These
- `ADR-0018 Fronteira Better-Auth × autenticação customizada` → `Bug: tabela verifications sempre vazia (aberto)`  [AMBIGUOUS]
  espec/docs/tarefas.md · relation: conceptually_related_to

## Knowledge Gaps
- **348 isolated node(s):** `ChangePasswordInput`, `DeactivateAccountInput`, `DeleteAccountInput`, `LoginInput`, `LoginOutput` (+343 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **48 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `ADR-0018 Fronteira Better-Auth × autenticação customizada` and `Bug: tabela verifications sempre vazia (aberto)`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **Why does `User` connect `Entidade User (Domínio)` to `Use Cases de Perfil (Auth)`, `Hooks React Query — Auth`, `Use Cases Register/GetMe/UpdateProfile`, `Use Cases Email-Change/Delete/Env`?**
  _High betweenness centrality (0.032) - this node is a cross-community bridge._
- **Why does `Visão Geral do Sistema — BomberQuiz` connect `Sessão Única e LGPD (ADRs)` to `RFs de Papéis e Conta`, `Estrutura da Pergunta (domínio)`, `M7 Geração por IA — ADRs`?**
  _High betweenness centrality (0.026) - this node is a cross-community bridge._
- **Why does `Módulo 7 — Geração de Questões por IA` connect `M7 Geração por IA — ADRs` to `Sessão Única e LGPD (ADRs)`?**
  _High betweenness centrality (0.025) - this node is a cross-community bridge._
- **What connects `ChangePasswordInput`, `DeactivateAccountInput`, `DeleteAccountInput` to the rest of the system?**
  _354 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Documentação e Módulos do Espec` be split into smaller, more focused modules?**
  _Cohesion score 0.05585106382978723 - nodes in this community are weakly interconnected._
- **Should `Use Cases de Perfil (Auth)` be split into smaller, more focused modules?**
  _Cohesion score 0.07682926829268293 - nodes in this community are weakly interconnected._