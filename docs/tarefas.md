# Tarefas — BomberQuiz

> Atualize este arquivo ao final de cada sessão de trabalho não-trivial.

## Realizadas

- [x] 2026-05-28 — Estrutura inicial do projeto: `CLAUDE.md`, `docs/requisitos.md`, `docs/tarefas.md`, `docs/decisoes.md`.
- [x] 2026-05-28 — Visão geral, modelo de negócio (gratuito + assinaturas), atores (Administrador, Cliente, Parceiro) e lista ilustrativa de matérias do TAP em `docs/requisitos.md`.
- [x] 2026-05-28 — Hierarquia de conteúdo definida: Eixo Temático → Matéria → Pergunta. Cadastros mapeados. Decisões registradas em ADR-0003 e ADR-0004.
- [x] 2026-05-28 — Cadastro de usuário (nome, e-mail único, WhatsApp, DOB, sexo, senha, avatar opcional), whitelist de administradores (máx. 5), e funcionalidade "Doar assinatura" especificadas. ADR-0005, 0006 e 0007 registrados.
- [x] 2026-05-28 — Stack técnica definida (Bun + Hono + Drizzle + Postgres/Neon + Better-Auth no backend, Vite + React + Tailwind + shadcn + PWA no frontend; Fly.io + Cloudflare Pages; Mercado Pago + WhatsApp Cloud API + Resend + R2 como integrações). Arquitetura hexagonal documentada em `docs/arquitetura.md`. ADR-0008 a ADR-0012 registrados.
- [x] 2026-05-28 — Convenção de RFs definida (6 módulos, formato `<MODULO>-RF-NNN`, arquivos separados em `docs/rf/`). Índice criado em `docs/requisitos.md`.
- [x] 2026-05-28 — Módulo 1 (Autenticação e cadastro) — rascunho com AUTH-RF-001 a AUTH-RF-011 em `docs/rf/auth.md`. 5 pendências identificadas (AUTH-P-01 a AUTH-P-05).
- [x] 2026-05-28 — Pendências do Módulo 1 resolvidas: idade mínima 18 anos; sexo (M/F/Prefere não informar); recuperação só por e-mail; sem OTP de WhatsApp; avatar como URL em Cloudinary. ADR-0013 registrado.
- [x] 2026-05-28 — Módulo 2 (Perfil e papéis) — rascunho com PROF-RF-001 a PROF-RF-013 em `docs/rf/profile.md`. Decisões: sessão única estilo Netflix (ADR-0014, ajustes em AUTH-RF-005 CA-2 e AUTH-RF-008 CA-4), dois fluxos de saída de conta — desativar reversível + excluir com anonimização LGPD (ADR-0015), admin não edita dados pessoais de outros usuários, portabilidade LGPD sob demanda via e-mail (sem endpoint no MVP). 4 pendências identificadas (PROF-P-01 a PROF-P-04).
- [x] 2026-05-28 — Pendências do Módulo 2 resolvidas: sem auto-exclusão de contas desativadas no MVP (P-01); detalhe admin de usuário com resumo financeiro via novo PROF-RF-014 (P-02); cooldown anti-takeover de 30 dias entre trocas de e-mail e 7 dias bloqueando exclusão pós-troca (P-03); hash do e-mail anonimizado com sal global em env (P-04).
- [x] 2026-05-28 — Módulo 3 (Conteúdo admin) — rascunho com CONT-RF-001 a CONT-RF-016 em `docs/rf/content-admin.md`. Decisões: estrutura mínima da pergunta no MVP — 4 alternativas fixas, 1 correta, justificativa obrigatória, fonte oficial em texto livre opcional, 1 imagem opcional via R2 (ADR-0016); admin publica direto, parceiro entra em fila `pending_review` com aprovação/rejeição (motivo obrigatório); soft-delete `archived` em eixos/matérias/perguntas preservando estatísticas; reset opcional de estatísticas em edição via flag `reset_stats`. 4 pendências identificadas (CONT-P-01 a CONT-P-04).
- [x] 2026-05-28 — Pendências do Módulo 3 resolvidas: nível de dificuldade via job diário às 00:00 com bandas unrated/easy/medium/hard (novo CONT-RF-017); sem versionamento automático de perguntas + hard-delete permitido apenas quando `total_answers=0` (CONT-RF-012 atualizado); admins permanecem todos iguais (mantém ADR-0005); reset total de estatísticas confirmado.

## A realizar — Próximos passos

- [x] ~~Definir o tipo de aplicação~~ — PWA web mobile-first (ADR-0010).
- [x] ~~Definir gateway de pagamento~~ — Mercado Pago (ADR-0012).
- [x] ~~Definir forma de armazenamento~~ — Postgres no Neon, via Drizzle (ADR-0009).
- [ ] Esboçar fluxos principais detalhados (realizar teste, cadastrar questão, assinar, parceiro ganha assinatura).
- [ ] **Escrever os módulos de RF restantes:**
  - [x] Módulo 1 — Autenticação e cadastro (`docs/rf/auth.md`).
  - [x] Módulo 2 — Perfil e papéis (`docs/rf/profile.md`).
  - [x] Módulo 3 — Conteúdo admin (`docs/rf/content-admin.md`).
  - [ ] Módulo 4 — Conteúdo parceiro (`docs/rf/content-partner.md`).
  - [ ] Módulo 5 — Quiz (`docs/rf/quiz.md`).
  - [ ] Módulo 6 — Assinaturas e doações (`docs/rf/subscriptions.md`).
- [x] ~~Resolver pendências do Módulo 1 (AUTH-P-01 a AUTH-P-05).~~ Concluído.
- [ ] Detalhar modelo de dados (schema Drizzle) a partir do esboço em `docs/arquitetura.md`.
- [ ] Definir contrato OpenAPI inicial (endpoints de auth, quiz, admin).
- [ ] Definir mecanismo de geração do cliente HTTP no frontend a partir da spec OpenAPI.
- [ ] Resolver questões em aberto na seção final de `docs/requisitos.md`:
  - [x] ~~Política de exclusão de questões pelo parceiro~~ — definida: só as próprias.
  - [x] ~~Múltiplos administradores~~ — definido: máx. 5 via whitelist (ADR-0005).
  - [ ] Política de **edição** de questões pelo parceiro.
  - [ ] Janela de tempo para travar edição/exclusão após N respostas.
  - [ ] Critério de aprovação de questões cadastradas por parceiros.
  - [ ] Equivalência "questões cadastradas ↔ tempo de assinatura".
  - [ ] Duração do período gratuito.
  - [ ] Fórmula de "nível" da pergunta a partir de erros/acertos.
  - [ ] Filtro de matéria do parceiro (especialidade vs. livre).
  - [ ] Hierarquia entre administradores (super-admin?).
  - [ ] Estrutura da pergunta (nº de alternativas, única vs. múltiplas corretas, suporte a imagem).
  - [ ] Comentário/justificativa e referência à fonte na pergunta.
  - [ ] Versionamento da pergunta quando o manual de origem é atualizado.
  - [ ] Tipos de quiz suportados (simulado TAP completo, livre por matéria, "pontos fracos", cronometrado).
  - [ ] Política de reset de estatísticas (do cliente, da pergunta editada).
  - [x] ~~Cadastro de usuário: idade mínima.~~ 18 anos.
  - [x] ~~Cadastro de usuário: verificação obrigatória de WhatsApp (OTP) ou só de e-mail.~~ Só e-mail no MVP.
  - [x] ~~Cadastro de usuário: canal padrão de recuperação de senha.~~ E-mail.
  - [x] ~~Cadastro de usuário: opções do campo "sexo".~~ Masculino / Feminino / Prefere não informar.
  - [ ] Doação de assinatura: revogação e limites por admin.
- [ ] Listar as matérias **efetivas** do TAP (a partir do edital vigente) e cadastrá-las.
- [ ] Criar os dois repositórios no GitHub (`bomberquiz-api`, `bomberquiz-web`).
- [ ] Bootstrap do `bomberquiz-api` (estrutura de pastas, Bun init, Hono hello-world, Drizzle config, env Zod).
- [ ] Bootstrap do `bomberquiz-web` (Vite + React, Tailwind, shadcn init, vite-plugin-pwa).
- [ ] Configurar deploy: Fly.io (api) + Cloudflare Pages (web) + Neon (banco).
- [ ] Validar provedor de WhatsApp (Cloud API vs Z-API) e e-mail (Resend vs SES).
- [ ] Redigir política de privacidade e termo de consentimento LGPD a serem aceitos no cadastro.
- [ ] Especificar fluxo técnico de verificação de e-mail (envio, expiração de token, reenvio).

## Backlog (sem prioridade definida)

- _Vazio._
