# Graph Report - .  (2026-07-15)

## Corpus Check
- 61 files · ~69,684 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 734 nodes · 1003 edges · 93 communities (64 shown, 29 thin omitted)
- Extraction: 99% EXTRACTED · 1% INFERRED · 0% AMBIGUOUS · INFERRED: 10 edges (avg confidence: 0.7)
- Token cost: 234,113 input · 0 output

## Community Hubs (Navigation)
- Documentação e Infra Core
- Entidade User — Métodos e DTOs
- Bootstrap da API + Testes E2E Auth
- Dependências Backend (package.json)
- Dependências Frontend (UI/forms)
- Use Cases Password/Logout
- DevDependencies Frontend (build/test)
- Use Cases Register/UpdateProfile
- Use Cases Consent/Deactivate/Login + Erros
- Hooks React Query — Perfil
- tsconfig.app.json (web)
- Rotas /me + Injeção de Use Cases
- Use Cases Consent/Deactivate/Delete (impl)
- Hooks React Query — Auth
- Rotas Auth + Validação HTTP
- tsconfig.json (web)
- Persistência: DB Client + User Repository
- components.json (shadcn config)
- tsconfig.node.json (web)
- Rate Limiting de Login
- Seções de Perfil (Web UI)
- Guards de Sessão (Web)
- Envelope de Erro (API Client)
- Confirm Email Change + Porta de Email
- Request Email Change + Erros
- AlertDialog (shadcn component)
- Entidade User (Domínio) + RBAC
- ResendEmailAdapter
- Validação de Telefone (Frontend)
- Delete Account + Config/Env
- Validação de Telefone (Backend)
- Card (shadcn component)
- Schemas Comuns (Zod)
- OpenAPI Schema Gerado (web)
- CI: Postgres + Testes + Deploy Fly.io
- App Root + Router (web)
- Login Use Case
- Button (shadcn component)
- Config de Testes (Docker/CI)
- Admin Whitelist
- Testes do Repositório de Usuário
- PhoneInput (componente web)
- Input (shadcn component)
- Label (shadcn component)
- VerifyEmailPage
- EmailConfirmPage
- tsconfig.json (root refs)
- Cloudinary (avatares)
- CI/Deploy Web (Cloudflare Pages)
- Checkbox (shadcn component)
- API Client (web)
- env.ts (web)
- vite.config.ts
- E2E Tests (descrição)
- Unit Tests (descrição)
- Rate Limiting (conceito)
- ADR-0001 Idioma/Documentação
- ADR-0002 Fase de Especificação
- ADR-0004 Hierarquia de Conteúdo
- ADR-0011 Arquitetura Hexagonal
- ADR-0019 Nota Fiscal Fora do MVP
- index.html (SPA entry)
- App Icon 192x192
- App Icon 512x512

## God Nodes (most connected - your core abstractions)
1. `User` - 41 edges
2. `IUserRepository` - 21 edges
3. `scripts` - 19 edges
4. `compilerOptions` - 18 edges
5. `Architecture — BomberQuiz (arquitetura.md)` - 17 edges
6. `compilerOptions` - 15 edges
7. `registerVerifiedUser()` - 15 edges
8. `API Contract — BomberQuiz (api.md)` - 15 edges
9. `postJson()` - 13 edges
10. `Requirements Specification — BomberQuiz (requisitos.md)` - 13 edges

## Surprising Connections (you probably didn't know these)
- `deploy.yml — Web CI/CD Pipeline (GitHub Actions)` --semantically_similar_to--> `Architecture — BomberQuiz (arquitetura.md)`  [INFERRED] [semantically similar]
  web/.github/workflows/deploy.yml → requisitos/docs/arquitetura.md
- `postgres-dev service` --semantically_similar_to--> `Neon Postgres (database)`  [INFERRED] [semantically similar]
  api/docker-compose.dev.yml → requisitos/docs/arquitetura.md
- `docker-compose.dev.yml — Local Dev Postgres` --conceptually_related_to--> `Architecture — BomberQuiz (arquitetura.md)`  [INFERRED]
  api/docker-compose.dev.yml → requisitos/docs/arquitetura.md
- `CONT-RF-017 Cálculo diário do nível de dificuldade` --semantically_similar_to--> `Module 5 — Quiz RF (quiz.md)`  [INFERRED] [semantically similar]
  requisitos/docs/rf/content-admin.md → requisitos/docs/rf/quiz.md
- `Cloudflare Domain Setup Runbook (runbook-dominio-cloudflare.md)` --references--> `deploy.yml — Web CI/CD Pipeline (GitHub Actions)`  [EXTRACTED]
  requisitos/docs/runbook-dominio-cloudflare.md → web/.github/workflows/deploy.yml

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **BomberQuiz Functional Requirement Modules (M1-M7)** — requisitos_docs_rf_auth, requisitos_docs_rf_profile, requisitos_docs_rf_content_admin, requisitos_docs_rf_content_partner, requisitos_docs_rf_quiz, requisitos_docs_rf_subscriptions, requisitos_docs_rf_ai_generation, requisitos_docs_api [EXTRACTED 0.90]
- **Module 7 AI Question Generation — spec + supporting ADRs** — requisitos_docs_rf_ai_generation, requisitos_docs_decisoes_adr_0021, requisitos_docs_decisoes_adr_0022, requisitos_docs_decisoes_adr_0023, requisitos_docs_decisoes_adr_0024, requisitos_docs_decisoes_adr_0025, requisitos_docs_decisoes_adr_0026 [EXTRACTED 0.90]
- **Core stack & repository-split decisions (ADR-0008..0012)** — requisitos_docs_decisoes_adr_0008, requisitos_docs_decisoes_adr_0009, requisitos_docs_decisoes_adr_0010, requisitos_docs_decisoes_adr_0011, requisitos_docs_decisoes_adr_0012 [EXTRACTED 0.85]

## Communities (93 total, 29 thin omitted)

### Community 0 - "Documentação e Infra Core"
Cohesion: 0.06
Nodes (62): docker-compose.dev.yml — Local Dev Postgres, bomberquiz-dev-data volume, postgres-dev service, CLAUDE.md — Project Guidance for Claude Code, API Contract — BomberQuiz (api.md), Architecture — BomberQuiz (arquitetura.md), audit_log (transversal audit table), Better-Auth (session/identity library) (+54 more)

### Community 1 - "Entidade User — Métodos e DTOs"
Cohesion: 0.07
Nodes (5): GetMeOutput, User, DrizzleUserRepository, entityToRow(), rowToEntity()

### Community 2 - "Bootstrap da API + Testes E2E Auth"
Cohesion: 0.19
Nodes (18): app, server, postRegister(), authedRequest(), extractTokenFromEmail(), loginAndGetCookie(), postJson(), randomTestIp() (+10 more)

### Community 3 - "Dependências Backend (package.json)"
Cohesion: 0.06
Nodes (34): dependencies, better-auth, drizzle-orm, hono, @hono/zod-openapi, postgres, resend, zod (+26 more)

### Community 4 - "Dependências Frontend (UI/forms)"
Cohesion: 0.06
Nodes (29): dependencies, class-variance-authority, clsx, @hookform/resolvers, lucide-react, openapi-fetch, @radix-ui/react-checkbox, @radix-ui/react-dialog (+21 more)

### Community 5 - "Use Cases Password/Logout"
Cohesion: 0.09
Nodes (17): ChangePasswordInput, ChangePasswordUseCase, ForgotPasswordUseCase, LogoutUseCase, ResetPasswordInput, ResetPasswordUseCase, WeakPasswordError, ContextVariableMap (+9 more)

### Community 6 - "DevDependencies Frontend (build/test)"
Cohesion: 0.07
Nodes (29): devDependencies, autoprefixer, jsdom, openapi-typescript, postcss, tailwindcss, @testing-library/jest-dom, @testing-library/react (+21 more)

### Community 7 - "Use Cases Register/UpdateProfile"
Cohesion: 0.17
Nodes (8): RegisterUserInput, RegisterUserUseCase, UpdateProfileInput, UpdateProfileUseCase, IPhoneValidator, PhoneType, PhoneValidationResult, Phone

### Community 8 - "Use Cases Consent/Deactivate/Login + Erros"
Cohesion: 0.15
Nodes (9): DeactivateAccountInput, LoginInput, LoginOutput, AccountInactiveError, ConsentRenewalRequiredError, EmailNotVerifiedError, InvalidCredentialsError, ReauthRequiredError (+1 more)

### Community 9 - "Hooks React Query — Perfil"
Cohesion: 0.10
Nodes (11): ChangePasswordFormValues, changePasswordSchema, DeleteAccountFormValues, deleteAccountSchema, passwordSchema, ReauthActionFormValues, reauthActionSchema, RequestEmailChangeFormValues (+3 more)

### Community 10 - "tsconfig.app.json (web)"
Cohesion: 0.10
Nodes (20): compilerOptions, allowImportingTsExtensions, isolatedModules, jsx, lib, module, moduleDetection, moduleResolution (+12 more)

### Community 11 - "Rotas /me + Injeção de Use Cases"
Cohesion: 0.10
Nodes (18): DeletionBlockedByEmailChangeError, acceptConsentUseCase, ChangePasswordBodySchema, changePasswordUseCase, confirmEmailChangeUseCase, deactivateAccountUseCase, DeactivateBodySchema, DeleteAccountBodySchema (+10 more)

### Community 12 - "Use Cases Consent/Deactivate/Delete (impl)"
Cohesion: 0.15
Nodes (5): AcceptConsentUseCase, DeactivateAccountUseCase, DeleteAccountUseCase, GetMeUseCase, IUserRepository

### Community 13 - "Hooks React Query — Auth"
Cohesion: 0.12
Nodes (11): useForgotPassword(), useResendVerification(), ForgotPasswordFormValues, forgotPasswordSchema, LoginFormValues, loginSchema, passwordSchema, RegisterFormValues (+3 more)

### Community 14 - "Rotas Auth + Validação HTTP"
Cohesion: 0.11
Nodes (16): InvalidPhoneError, zodValidationHook(), authRoutes, ForgotPasswordBodySchema, forgotPasswordUseCase, getMeUseCase, LoginBodySchema, loginUseCase (+8 more)

### Community 15 - "tsconfig.json (web)"
Cohesion: 0.11
Nodes (17): compilerOptions, allowImportingTsExtensions, allowSyntheticDefaultImports, baseUrl, ignoreDeprecations, lib, module, moduleDetection (+9 more)

### Community 16 - "Persistência: DB Client + User Repository"
Cohesion: 0.19
Nodes (11): DB, queryClient, UserRow, accounts, schema, sessionRevocationEnum, userRoleEnum, users (+3 more)

### Community 17 - "components.json (shadcn config)"
Cohesion: 0.13
Nodes (14): aliases, components, ui, utils, rsc, $schema, style, tailwind (+6 more)

### Community 18 - "tsconfig.node.json (web)"
Cohesion: 0.14
Nodes (13): compilerOptions, allowImportingTsExtensions, isolatedModules, lib, module, moduleDetection, moduleResolution, noEmit (+5 more)

### Community 19 - "Rate Limiting de Login"
Cohesion: 0.18
Nodes (9): checkLimit(), LOCKOUT_STEPS, LoginAttemptEntry, loginAttempts, rateLimitByIp(), rateLimitByUser(), WindowEntry, windows (+1 more)

### Community 20 - "Seções de Perfil (Web UI)"
Cohesion: 0.23
Nodes (4): ChangeEmailSection(), ChangePasswordSection(), DangerZoneSection(), PersonalInfoSection()

### Community 21 - "Guards de Sessão (Web)"
Cohesion: 0.31
Nodes (8): ConsentGate(), RequireAuth(), RequireGuest(), fetchSession(), SESSION_QUERY_KEY, SessionData, SessionUser, useSession()

### Community 22 - "Envelope de Erro (API Client)"
Cohesion: 0.27
Nodes (8): ApiError, ApiErrorField, ErrorEnvelope, ErrorEnvelopeSchema, ErrorFieldSchema, parseErrorBody(), throwIfError(), unwrap()

### Community 24 - "Request Email Change + Erros"
Cohesion: 0.22
Nodes (4): RequestEmailChangeInput, RequestEmailChangeUseCase, EmailAlreadyInUseError, EmailChangeCooldownError

### Community 25 - "AlertDialog (shadcn component)"
Cohesion: 0.22
Nodes (6): AlertDialogAction, AlertDialogCancel, AlertDialogContent, AlertDialogDescription, AlertDialogOverlay, AlertDialogTitle

### Community 26 - "Entidade User (Domínio) + RBAC"
Cohesion: 0.29
Nodes (5): UserProps, UserRole, UserSex, UserStatus, ROLE_HIERARCHY

### Community 28 - "Validação de Telefone (Frontend)"
Cohesion: 0.25
Nodes (4): PhoneType, PhoneValidationResult, ADR-0008, VALID_DDDS

### Community 29 - "Delete Account + Config/Env"
Cohesion: 0.33
Nodes (4): DeleteAccountInput, envSchema, parsed, sessions

### Community 30 - "Validação de Telefone (Backend)"
Cohesion: 0.33
Nodes (4): RegexPhoneValidator, ADR-0008, VALID_DDDS, validator

### Community 31 - "Card (shadcn component)"
Cohesion: 0.29
Nodes (6): Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle

### Community 32 - "Schemas Comuns (Zod)"
Cohesion: 0.33
Nodes (4): ErrorFieldSchema, ErrorResponseSchema, PaginationQuerySchema, UserProfileSchema

### Community 33 - "OpenAPI Schema Gerado (web)"
Cohesion: 0.33
Nodes (5): components, $defs, operations, paths, webhooks

### Community 34 - "CI: Postgres + Testes + Deploy Fly.io"
Cohesion: 0.40
Nodes (5): postgres-test service, CI job (typecheck + tests), Deploy job (flyctl to Fly.io), Postgres service container (CI), Integration tests (transaction rollback)

### Community 37 - "Button (shadcn component)"
Cohesion: 0.50
Nodes (3): Button, ButtonProps, buttonVariants

### Community 38 - "Config de Testes (Docker/CI)"
Cohesion: 0.67
Nodes (3): docker-compose.test.yml (Postgres test), Deploy Workflow (GitHub Actions), Tests README (api)

## Knowledge Gaps
- **298 isolated node(s):** `ChangePasswordInput`, `DeactivateAccountInput`, `DeleteAccountInput`, `LoginInput`, `LoginOutput` (+293 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **29 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `User` connect `Entidade User — Métodos e DTOs` to `Login Use Case`, `Use Cases Consent/Deactivate/Login + Erros`, `Use Cases Consent/Deactivate/Delete (impl)`, `Hooks React Query — Auth`, `Persistência: DB Client + User Repository`, `Entidade User (Domínio) + RBAC`?**
  _High betweenness centrality (0.048) - this node is a cross-community bridge._
- **Why does `IUserRepository` connect `Use Cases Consent/Deactivate/Delete (impl)` to `Entidade User — Métodos e DTOs`, `Login Use Case`, `Use Cases Consent/Deactivate/Login + Erros`, `Persistência: DB Client + User Repository`, `Confirm Email Change + Porta de Email`, `Delete Account + Config/Env`?**
  _High betweenness centrality (0.015) - this node is a cross-community bridge._
- **Why does `auth` connect `Use Cases Password/Logout` to `Bootstrap da API + Testes E2E Auth`, `Use Cases Register/UpdateProfile`, `Use Cases Consent/Deactivate/Login + Erros`, `Rotas Auth + Validação HTTP`, `Delete Account + Config/Env`?**
  _High betweenness centrality (0.013) - this node is a cross-community bridge._
- **What connects `ChangePasswordInput`, `DeactivateAccountInput`, `DeleteAccountInput` to the rest of the system?**
  _311 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Documentação e Infra Core` be split into smaller, more focused modules?**
  _Cohesion score 0.06187202538339503 - nodes in this community are weakly interconnected._
- **Should `Entidade User — Métodos e DTOs` be split into smaller, more focused modules?**
  _Cohesion score 0.06825396825396825 - nodes in this community are weakly interconnected._
- **Should `Dependências Backend (package.json)` be split into smaller, more focused modules?**
  _Cohesion score 0.05714285714285714 - nodes in this community are weakly interconnected._