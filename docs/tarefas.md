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
- [x] 2026-05-28 — Módulo 4 (Conteúdo parceiro) — rascunho com PART-RF-001 a PART-RF-008 em `docs/rf/content-partner.md`. Decisões: parceiro cadastra livre em qualquer matéria ativa (sem especialidade); edição de própria publicada devolve a pergunta para `pending_review`; exclusão restrita a `draft`; parceiro vê estatísticas e histórico de revisão das próprias; painel/dashboard com agregados motivacionais. 4 pendências identificadas (PART-P-01 a PART-P-04).
- [x] 2026-05-28 — Pendências do Módulo 4 resolvidas: limite de 50 rascunhos confirmado (P-01); notificação ao parceiro via badge no app (`unread_review_events`) + e-mail transacional Resend (P-02), formalizada em CONT-RF-015/016 e PART-RF-007/008; parceiro só vê as próprias perguntas (P-03), divisão de trabalho fica externa ao sistema; rubrica operacional inicial criada em `docs/rubrica-aprovacao.md` (P-04).
- [x] 2026-05-28 — Módulo 5 (Quiz cliente) — rascunho com QUIZ-RF-001 a QUIZ-RF-009 em `docs/rf/quiz.md`. Decisões: 3 modos no MVP (simulado TAP respeitando `tap_weight`, livre por matéria, livre por eixo); cronômetro opcional com tempo total (3 min/questão default, ajustável 1–5 min); justificativa configurável (após cada / só no final); quiz online-only e efêmero (sem pause/resume); snapshot da pergunta no momento do sorteio para imunidade a edições futuras; reset total de estatísticas pelo cliente preservando histórico de quizzes; bloqueio HTTP 402 quando sem assinatura/trial ativos. 7 pendências identificadas (QUIZ-P-01 a QUIZ-P-07).
- [x] 2026-05-28 — Pendências do Módulo 5 resolvidas: assimetria explícita de estatísticas (finished/expired contam não-respondidas como erro; abandoned preserva só o que foi respondido) + auto-abandono após 24h sem atividade (QUIZ-P-02); reset seletivo descartado, total mantido (QUIZ-P-03); UI de cronômetro com toggle/switch principal formalizada em QUIZ-RF-001 (QUIZ-P-06); leaderboard descartado em favor de novo QUIZ-RF-010 — evolução temporal própria do usuário (QUIZ-P-07). P-01/04/05 mantidas conscientemente como pós-MVP.
- [x] 2026-05-28 — Módulo 6 (Assinaturas e cortesias) — rascunho com SUB-RF-001 a SUB-RF-015 em `docs/rf/subscriptions.md`. Decisões: trial automático de 7 dias após verificação de e-mail (único por usuário); 4 planos (mensal/trimestral/semestral/anual) configuráveis pelo admin; cobrança via PIX, saldo MP e cartão (cartão +10%, parcelável em até 3×); sem recorrência automática no MVP — TAP é cíclico, lembretes D-7/D-3/D-1 + e-mail final no D-0; cortesia (renomeado de "doação") como funcionalidade de primeira classe com 2 categorias (parceria/demonstracao), limite de 10/mês por admin e revogação com motivo obrigatório; cupom de desconto incluído no MVP (SUB-RF-013/015) para marketing inicial; reembolso automático em 7 dias atendendo CDC art. 49 (SUB-RF-014); comprovante via link MP; painel financeiro separa receitas de cortesias (ADR-0006); cálculo de `access_status` consolida trial+paga+cortesia (consumido por QUIZ-RF-009 e PROF-RF-014). Todas as 6 pendências iniciais resolvidas.
- [x] 2026-05-28 — Renomeação "doação" → "cortesia" propagada em toda a documentação (subscriptions.md, profile.md, requisitos.md, decisoes.md, arquitetura.md). Schema renomeado: tabela `courtesies`, `source=courtesy`, `courtesy_id`. ADR-0006 atualizado.
- [x] 2026-05-28 — Revisão pré-implementação dos 8 itens identificados: (1) schema do `audit_log` documentado em `arquitetura.md`; (2) formato padronizado de resposta de erro `{ error: { code, message, details?, fields?, request_id } }`; (3) rate limit baseline 60 req/min por IP + 120 req/min por user; (4) estratégia de domínios em duas fases (provisórios `pages.dev`/`fly.dev` → próprios `*.bomberquiz.com.br` no lançamento) com config-driven URLs; (5) Resend confirmado como provedor de e-mail; (6) WhatsApp adiado conscientemente (sem canal ativo no MVP); (7) Política de privacidade + Termos como bloqueio explícito de lançamento (não da implementação); (8) matérias do TAP cadastráveis em runtime — ativar/desativar via CONT-RF-008 absorve o ciclo anual do edital (labels de UI: "Desativar matéria"/"Reativar matéria").

## A realizar — Próximos passos

- [x] ~~Definir o tipo de aplicação~~ — PWA web mobile-first (ADR-0010).
- [x] ~~Definir gateway de pagamento~~ — Mercado Pago (ADR-0012).
- [x] ~~Definir forma de armazenamento~~ — Postgres no Neon, via Drizzle (ADR-0009).
- [x] ~~Esboçar fluxos principais detalhados~~ — coberto pelos CAs de cada RF nos arquivos `docs/rf/*.md`.
- [ ] **Escrever os módulos de RF restantes:**
  - [x] Módulo 1 — Autenticação e cadastro (`docs/rf/auth.md`).
  - [x] Módulo 2 — Perfil e papéis (`docs/rf/profile.md`).
  - [x] Módulo 3 — Conteúdo admin (`docs/rf/content-admin.md`).
  - [x] Módulo 4 — Conteúdo parceiro (`docs/rf/content-partner.md`).
  - [x] Módulo 5 — Quiz (`docs/rf/quiz.md`).
  - [x] Módulo 6 — Assinaturas e cortesias (`docs/rf/subscriptions.md`).
- [x] ~~Resolver pendências do Módulo 1 (AUTH-P-01 a AUTH-P-05).~~ Concluído.
- [ ] Detalhar modelo de dados (schema Drizzle) a partir do esboço em `docs/arquitetura.md`.
- [ ] Definir contrato OpenAPI inicial (endpoints de auth, quiz, admin).
- [ ] Definir mecanismo de geração do cliente HTTP no frontend a partir da spec OpenAPI.
- [x] ~~Resolver questões em aberto na seção final de `docs/requisitos.md`~~ — todas as questões que estavam na lista original foram resolvidas durante a redação dos Módulos 1–6. Pendências específicas residuais ficam nos rodapés dos arquivos `rf/*.md` (códigos `<MOD>-P-NN`).
- [x] ~~Listar as matérias efetivas do TAP~~ — **não-aplicável como tarefa prévia**. O cadastro acontece naturalmente via UI quando o admin estiver no ar (CONT-RF-002/006), e o ciclo do edital (matérias entram/saem a cada ano) é absorvido pela mecânica de ativar/desativar matéria (CONT-RF-008 CA-5).
- [ ] Criar os dois repositórios no GitHub (`bomberquiz-api`, `bomberquiz-web`).
- [ ] Bootstrap do `bomberquiz-api` (estrutura de pastas, Bun init, Hono hello-world, Drizzle config, env Zod).
- [ ] Bootstrap do `bomberquiz-web` (Vite + React, Tailwind, shadcn init, vite-plugin-pwa).
- [ ] Configurar deploy: Fly.io (api) + Cloudflare Pages (web) + Neon (banco).
- [x] ~~Validar provedor de e-mail~~ — **Resend** confirmado (ADR-0012 atualizado).
- [ ] Validar provedor de WhatsApp (Cloud API vs Z-API) — adiado conscientemente; WhatsApp não é canal ativo no MVP (só contato declarado). Reavaliar quando virar canal ativo.
- [ ] **🚩 Bloqueio de lançamento público:** Redigir Política de Privacidade + Termos de Uso (LGPD). Beta privado pode rodar com versão preliminar; contratar advogado especializado para revisão antes da abertura pública. Versionar como `docs/legal/privacidade-vN.md` e `docs/legal/termos-vN.md` (sistema já tem `consent_version` e reaceite em PROF-RF-006).
- [ ] Especificar fluxo técnico de verificação de e-mail (envio, expiração de token, reenvio).

## Backlog (sem prioridade definida)

- _Vazio._
