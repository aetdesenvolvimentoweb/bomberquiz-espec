# Especificação de requisitos — BomberQuiz

> Documento vivo. Atualize conforme as decisões forem tomadas.

## Visão geral

BomberQuiz é um sistema de perguntas e respostas para auxiliar no estudo e fixação dos conteúdos cobrados no **Teste de Avaliação Profissional (TAP) do Corpo de Bombeiros Militar do Estado de Goiás (CBMGO)**. O usuário realiza testes curtos, **com ou sem cronômetro**, e acompanha sua evolução por matéria para concentrar esforços onde tem mais dificuldade.

O sistema tem duas frentes:
1. **Cadastro de questões** (administradores e parceiros).
2. **Realização de testes** (clientes / usuários assinantes).

## Modelo de negócio

- **Período gratuito** de acesso (duração a definir).
- Após o período, acesso por **assinatura paga** com planos:
  - Mensal
  - Trimestral
  - Semestral
  - Anual
- **Cortesia de assinatura:** administradores podem conceder assinatura gratuita a qualquer usuário por período arbitrário (ver "Cortesia de assinatura"). É o mecanismo usado para remunerar parceiros pelo trabalho de cadastro e também para uso promocional/demonstração.
- **Parceria por contrapartida:** parceiros recebem assinatura doada em troca de cadastro de questões. _A definir: regras de equivalência (quantas questões = quanto tempo de assinatura), critérios de aprovação das questões cadastradas._

## Atores (papéis)

### Administrador
- Equipe **pequena** no início: **máximo 5 administradores**.
- Controle de acesso via **whitelist de e-mails** mantida em configuração (não autoatribuição). Quando um usuário whitelist faz login pela primeira vez, é promovido a administrador automaticamente. Ver ADR-0005.
- Cadastra perguntas e respostas (sem restrição de matéria).
- Cadastra **Eixos Temáticos** e **Matérias** (CRUD completo).
- **Concede cortesia de assinatura** a usuários (clientes ou parceiros) — período arbitrário, com categoria registrada. Ver "Cortesia de assinatura" abaixo.
- Gerencia parceiros (promove cliente a parceiro; revoga o papel).
- Acessa o painel administrativo-financeiro:
  - Estatísticas por pergunta (nível de dificuldade derivado de erros/acertos acumulados).
  - Quantidade de perguntas cadastradas por cada parceiro.
  - Assinaturas vigentes (por plano, por status), incluindo assinaturas doadas.
  - Parte financeira: receitas, fluxo de caixa.

### Cliente (Usuário)
- Faz cadastro próprio (ver "Cadastro e dados do usuário").
- Assina o aplicativo (ou usa no período gratuito, ou via assinatura doada).
- Responde questionários nas matérias do TAP.
- Acompanha a **evolução do seu desempenho por matéria**, identificando pontos fortes e fracos.
- _A definir: pode escolher matérias específicas para o teste? Pode definir tamanho do teste (nº de questões)? Cronômetro é opcional por teste ou configurável globalmente?_

### Parceiro
- Tem todos os direitos de um Cliente (responde testes, acompanha desempenho).
- **Adicionalmente** cadastra perguntas e respostas — acessa a tela de gerenciamento de perguntas.
- **Exclusão:** pode excluir **apenas as perguntas que ele próprio cadastrou**, e só enquanto `draft`. Em `pending_review`/`published`/`archived`, só admin atua. PART-RF-005.
- **Edição:** livre em `draft`; edição de pergunta já `published` devolve para `pending_review` (sai do catálogo até nova aprovação). PART-RF-003.
- Não é assinante pagante: recebe **assinatura doada** por administrador como contrapartida pelo trabalho de cadastro.
- Cadastra livremente em **qualquer matéria `active`** — sem vinculação por especialidade no MVP. PART-RF-002.

## Cadastro e dados do usuário

Todos os papéis (Administrador, Cliente, Parceiro) compartilham o mesmo cadastro base. O papel é uma **propriedade** do usuário, atribuída conforme regras descritas em "Atores".

### Dados solicitados no cadastro

| Campo | Obrigatório? | Observações |
|---|---|---|
| Nome completo | Sim | — |
| E-mail | Sim | **Único no sistema.** Canal principal de contato e identificador para login. Deve ser **verificado** após o cadastro (envio de link/código). |
| Telefone (WhatsApp) | Sim | Canal secundário de contato, recuperação de senha, notificações. _A definir: validar via OTP no WhatsApp?_ |
| Data de nascimento | Sim | Permite ações de fidelização (saudação de aniversário) e segmentação. _A definir: idade mínima?_ |
| Sexo | Sim | Mesma justificativa de fidelização/segmentação. _Considerar opção "prefere não informar" para conformidade._ |
| Senha | Sim | Armazenada com hash adequado (a definir). Regras mínimas de força a definir. |
| Avatar | **Não** | Opcional. Sem upload, usuário recebe **avatar genérico** (placeholder do sistema). |

### Considerações

- **LGPD:** o sistema coleta dados pessoais (nome, e-mail, telefone, DOB, sexo). Será necessário: política de privacidade, termo de consentimento no cadastro, mecanismo de exclusão/portabilidade de dados.
- **Recuperação de senha:** via e-mail e/ou WhatsApp (a decidir o canal padrão).
- **Verificação de identidade:** mínimo é verificar o e-mail no cadastro. Verificação de WhatsApp pode vir em etapa posterior, especialmente se for usada para recuperação de senha.

## Cortesia de assinatura

Funcionalidade administrativa de primeira classe (não um workaround): permite a um administrador conceder assinatura gratuita a um usuário existente. Termo escolhido: "cortesia" (conotação comercial neutra; substituiu "doação" para evitar conotação de caridade e implicações fiscais inadequadas — ADR-0006). Atende dois usos com o mesmo mecanismo, diferenciados por categoria: remunerar parceiros pelo trabalho de cadastro (`parceria`) e uso promocional/demonstração (`demonstracao`).

Acumula sobre assinatura ativa existente (estende a expiração em vez de sobrepor), aparece separada de receita no painel financeiro, e é revogável pelo admin (motivo obrigatório, limite de 10/mês civil). Campos, regras e critérios de aceitação completos em [`rf/subscriptions.md`](rf/subscriptions.md) — SUB-RF-008 (conceder) e SUB-RF-010 (revogar).

## Domínio: estrutura dos conteúdos do TAP

O conteúdo do TAP é organizado em três níveis hierárquicos. Esses três níveis são todos **dados cadastráveis**, não enumerações fixas em código, porque o edital muda a cada ano.

### Nível 1 — Eixo Temático
> Termo de trabalho. Alternativas consideradas: "Grupo Temático", "Área de Conhecimento", "Bloco". Decisão registrada em `docs/decisoes.md` (ADR-0003).

Agrupamento de matérias afins. Cada eixo tem um **peso/quantidade de questões cobradas no TAP** — o edital define a prova em blocos por eixo (ex.: Legislação e Normas = 18 questões, Salvamento = 9 questões), não por matéria individual. Esse peso é usado pelo sistema na hora de **montar um quiz que simule a distribuição real da prova**. A prova do TAP tem **50 questões no total**, então a soma dos pesos de todos os eixos deve totalizar ~50 (o simulado TAP em QUIZ-RF-001 reflete essa distribuição). Exemplos:
- **Salvamento**
- **Prevenção e Combate a Incêndio**
- _(outros eixos a serem cadastrados conforme edital vigente)_

### Nível 2 — Matéria
Cada matéria pertence a **um** eixo temático e geralmente corresponde a uma **fonte oficial** (manual operacional, norma técnica, lei, protocolo). A matéria em si não carrega peso próprio no TAP — o peso é do eixo (nível 1); a matéria só organiza o conteúdo/fonte oficial dentro do eixo.

Exemplos:

| Eixo Temático | Matéria | Fonte oficial |
|---|---|---|
| Salvamento | Salvamento Terrestre | Manual Operacional de Salvamento Terrestre |
| Salvamento | Salvamento em Altura | Manual Operacional de Salvamento em Altura |
| Salvamento | Salvamento Aquático | Manual Operacional de Guarda-Vidas |
| Prevenção e Combate a Incêndio | Combate a Incêndio Urbano | Manual Operacional de Combate a Incêndio Urbano |
| Prevenção e Combate a Incêndio | Combate a Incêndio Florestal | Manual de Prevenção e Combate a Incêndios Florestais |
| Prevenção e Combate a Incêndio | Norma Técnica 01 — Procedimentos Administrativos | NT-01 CBMGO |
| Prevenção e Combate a Incêndio | Norma Técnica 11 — Saídas de Emergência | NT-11 CBMGO |

Outros eixos prováveis (a confirmar com o edital): **Suporte Básico de Vida / APH**, **Legislação**, **Atividades Técnicas / Vistorias**.

### Nível 3 — Pergunta e Respostas
Cada pergunta pertence a **uma** matéria. Estrutura mínima (a refinar nos RFs):
- Enunciado.
- Alternativas (provavelmente múltipla escolha — quantidade fixa? variável?).
- Indicação da(s) alternativa(s) correta(s).
- Metadados: autor (admin ou parceiro), data de cadastro, status (rascunho/aprovada/arquivada?), estatísticas acumuladas (erros/acertos).
- _A definir:_ comentário/justificativa da resposta? referência à página/artigo da fonte oficial? imagens?

## Cadastros do sistema

Decorrem da estrutura acima:

1. **Eixos Temáticos** — CRUD restrito a administradores.
2. **Matérias** — CRUD restrito a administradores. Cada matéria vincula-se a um eixo e armazena fonte + quantidade de questões esperada no TAP.
3. **Perguntas e Respostas** — cadastro por administradores e parceiros (regras de edição/exclusão e aprovação a definir, ver "Questões em aberto").

## Requisitos funcionais

Organizados em **7 módulos**, cada um em arquivo próprio sob `docs/rf/`. Identificadores no formato `<MODULO>-RF-NNN` (ex.: `AUTH-RF-001`). Cada RF traz: prioridade (Essencial / Desejável / Futuro), ator, pré-condições, descrição, critérios de aceitação (CA-N) e erros previstos (E-N).

| # | Módulo | Arquivo | Status |
|---|---|---|---|
| 1 | Autenticação e cadastro de usuário | [`rf/auth.md`](rf/auth.md) | ✅ Rascunho |
| 2 | Perfil e papéis | [`rf/profile.md`](rf/profile.md) | ✅ Rascunho |
| 3 | Conteúdo (admin) — eixos, matérias, perguntas | [`rf/content-admin.md`](rf/content-admin.md) | ✅ Rascunho |
| 4 | Conteúdo (parceiro) | [`rf/content-partner.md`](rf/content-partner.md) | ✅ Rascunho |
| 5 | Quiz (cliente) | [`rf/quiz.md`](rf/quiz.md) | ✅ Rascunho |
| 6 | Assinaturas e cortesias | [`rf/subscriptions.md`](rf/subscriptions.md) | ✅ Rascunho |
| 7 | Geração de questões por IA | [`rf/ai-generation.md`](rf/ai-generation.md) | ⚠️ Rascunho (retirado de produção, ver ADR-0039) |

Pendências por módulo (decisões a tomar antes da implementação) ficam no rodapé de cada arquivo, identificadas como `<MODULO>-P-NN`.

O **inventário consolidado de endpoints** (todos os módulos) e as convenções de contrato estão em [`api.md`](api.md).

## Requisitos não-funcionais

> Performance, usabilidade, plataforma alvo, offline-first, multi-usuário, etc.

- **Porte esperado:** mercado-teto de ~2.000 bombeiros ativos no CBMGO; base usual estimada em 300–500 usuários ativos (referência: último TAP teve ~300 concorrentes). Decisões de arquitetura priorizam **custo baixo** e **manutenibilidade**, não escala massiva.
- **Plataforma:** aplicação **web mobile-first PWA** — instalável, com cache offline básico de questões já carregadas. Sem app nativo no MVP.
- **Idioma:** pt-BR em toda a interface, e-mails e mensagens.
- **Segurança:** seguir OWASP Top 10. Hash de senha com argon2id ou bcrypt. CSP, rate limiting, validação de entrada (Zod) em todas as rotas.
- **LGPD:** consentimento explícito, política de privacidade acessível, mecanismo de exclusão/portabilidade de dados.
- **Acessibilidade:** componentes acessíveis (shadcn já é a11y-friendly); meta de WCAG AA.
- **Disponibilidade:** alvo 99% (não 99.9% — não justifica o custo no porte).
- **Performance:** Time-to-Interactive < 3s em 4G; First Contentful Paint < 1.5s.

## Stack técnica

> Decisões consolidadas em ADRs 0008–0012. Esta seção é a visão sintética.

### Arquitetura
- **Dois serviços, dois repositórios:** `bomberquiz-api` (backend REST) e `bomberquiz-web` (frontend PWA). Contrato entre eles via OpenAPI gerado pelo backend.
- **Backend em arquitetura hexagonal** (ports & adapters) — ver `docs/arquitetura.md`.

### Backend (`bomberquiz-api`)
- Runtime: **Bun**.
- Framework HTTP: **Hono**.
- ORM: **Drizzle**.
- Banco: **PostgreSQL** no **Neon** (free tier inicial, scale-to-zero, branching para testes de integração).
- Autenticação: **Better-Auth** (e-mail + senha, verificação de e-mail, recuperação).
- Validação: **Zod** (request/response schemas).
- Documentação: **OpenAPI via `@hono/zod-openapi`** + UI **Scalar**.
- Testes: **Vitest** (unit + integration) — integração contra Neon branch ou Postgres em container.
- Hospedagem: **Fly.io** (sempre-on no free tier inicial).

### Frontend (`bomberquiz-web`)
- Bundler/framework: **Vite + React + React Router**.
- UI: **Tailwind CSS + shadcn/ui**.
- Estado servidor: **TanStack Query**. Estado UI: **Zustand**.
- PWA: **`vite-plugin-pwa`** (service worker, manifest, install prompt).
- Testes: **Vitest** + **React Testing Library** (unit/integration); **Playwright** (E2E).
- Hospedagem: **Cloudflare Pages**.

### Integrações externas
- **Pagamentos:** Mercado Pago (PIX + cartão, assinaturas recorrentes).
- **WhatsApp:** WhatsApp Cloud API (Meta) ou Z-API — _provedor a confirmar_.
- **E-mail transacional:** **Resend** (confirmado, ADR-0012). Canal de suporte do MVP também é e-mail (`SUPPORT_EMAIL`). Migração para SES localizada na porta `EmailSender` se o volume crescer.
- **Armazenamento de imagens de questões:** Cloudinary (revisado 2026-07-19 — antes Cloudflare R2, ver ADR-0012) — mesmo provedor usado para avatares.
- **Armazenamento de avatares:** Cloudinary (free tier, transformações automáticas) — ver ADR-0013.
- **Armazenamento de backups:** Cloudflare R2 (sem cobrança de egress) — inalterado. (O uso original para PDFs temporários do Módulo 7 não se aplica mais desde a retirada do pipeline em 2026-07-27, ver ADR-0039.)

### Princípios
- Clean Architecture / Hexagonal — domínio puro, infra plugável.
- KISS, DRY, YAGNI.
- TDD onde fizer sentido (domain e use cases primeiro).
- TSDoc nos contratos públicos das camadas.
- OWASP Top 10 como checklist obrigatório antes de cada release.

## Fluxos principais

- _A definir (esboço inicial):_
  - **Cadastro do usuário:** usuário preenche dados → aceita termos (LGPD) → recebe e-mail de verificação → confirma → entra como Cliente.
  - **Login de administrador:** primeiro login de e-mail presente na whitelist promove o usuário automaticamente a Administrador.
  - **Promover Cliente a Parceiro:** administrador localiza cliente → atribui papel de parceiro → opcionalmente define matérias permitidas (ver questão em aberto).
  - **Doar assinatura:** admin busca usuário → informa período e motivo → confirma → assinatura ativa/estendida.
  - **Montar e realizar quiz:** cliente escolhe o escopo (TAP completo simulado, eixo específico, matéria específica, ou mix livre) → define cronômetro on/off → sistema sorteia questões respeitando o peso/quantidade de cada matéria quando o escopo for "TAP completo" → cliente responde → vê resultado → estatísticas atualizadas por matéria e por eixo.
  - **Cadastrar Eixo Temático:** admin cria/edita/remove eixo.
  - **Cadastrar Matéria:** admin cria matéria vinculada a um eixo + fonte oficial + quantidade de questões esperada no TAP.
  - **Cadastrar pergunta:** parceiro/admin escolhe matéria → cria enunciado + alternativas + resposta correta + metadados.
  - **Assinar (pago):** cliente escolhe plano → paga via gateway → assinatura ativa.

## Restrições e premissas

- Público-alvo é brasileiro (CBMGO), portanto pt-BR em toda a interface.
- **LGPD aplicável** — sistema coleta dados pessoais identificáveis. Necessário política de privacidade, consentimento explícito no cadastro, e mecanismo de exclusão a pedido do titular.
- E-mail é identificador único de usuário (não há login por CPF, username, etc.).
- Whitelist de administradores fica em **configuração da aplicação**, não cadastrada via UI por outro admin (evita autoescalada de privilégios). Máximo de 5 entradas no início.
- _Demais restrições a definir._

## Questões em aberto

### Resolvidas (mantidas aqui para referência histórica até virarem RF)
- ✅ Política de **exclusão** de questões pelo parceiro → só as próprias.
- ✅ Múltiplos administradores → sim, máximo 5 via whitelist (ADR-0005).
- ✅ Mecânica para parceiro receber assinatura → cortesia de assinatura (funcionalidade administrativa).
- ✅ Tipo de aplicação → PWA web mobile-first (ADR-0010).
- ✅ Forma de armazenamento → Postgres no Neon, via Drizzle (ADR-0009).
- ✅ Gateway de pagamento → Mercado Pago (ADR-0012).
- ✅ Política de saída de conta → dois fluxos: desativar (reversível) e excluir (anonimização irreversível das PII, preservando FKs). ADR-0015 / PROF-RF-007..009.
- ✅ Política de sessões simultâneas → máximo 1 sessão ativa por usuário; novo login encerra a anterior. ADR-0014 / PROF-RF-010.
- ✅ Edição de dados de outros usuários pelo admin → não permitida (admin só altera papel). PROF-RF-002 / PROF-RF-012 / PROF-RF-013.
- ✅ Estrutura da pergunta no MVP → 4 alternativas fixas, 1 única correta, justificativa obrigatória, fonte oficial em texto livre opcional, 1 imagem opcional. ADR-0016 / CONT-RF-010.
- ✅ Workflow de aprovação de perguntas → admin publica direto; parceiro envia para fila (`pending_review`); admin aprova ou rejeita com motivo obrigatório. ADR-0016 / CONT-RF-014..016.
- ✅ Reset de estatísticas em edição de pergunta → admin escolhe via flag `reset_stats` no PATCH; UI sugere `true` quando gabarito muda. Respostas antigas são preservadas (`stats_reset_at`). CONT-RF-011 CA-4.
- ✅ Fórmula de nível de dificuldade da pergunta → bandas `unrated` (< 30 respostas) / `easy` (≥ 70%) / `medium` (40–70%) / `hard` (< 40%), recalculadas por job diário às 00:00 (`America/Sao_Paulo`). CONT-RF-017.
- ✅ Versionamento de perguntas após atualização do manual → sem mecanismo automático no MVP; admin edita ou arquiva. Hard-delete só com `total_answers=0`. CONT-RF-012.
- ✅ Hierarquia entre administradores → todos iguais (mantém ADR-0005).
- ✅ Filtro de matéria do parceiro → cadastro livre em qualquer matéria `active`, sem vinculação por especialidade no MVP. PART-RF-002.
- ✅ Política de edição de pergunta pelo parceiro → livre em `draft`; edição em `published` devolve para `pending_review` (sai do catálogo até nova aprovação). PART-RF-003.
- ✅ Política de exclusão de pergunta pelo parceiro → hard-delete apenas em `draft` (próprias). Em `pending_review`/`published`/`archived`, só admin atua. PART-RF-005.
- ✅ Notificação ao parceiro de aprovação/rejeição → badge no app (`unread_review_events` em PART-RF-008) + e-mail transacional via Resend. CONT-RF-015 CA-4 / CONT-RF-016 CA-3.
- ✅ Critério humano de aprovação → rubrica operacional inicial em [`rubrica-aprovacao.md`](rubrica-aprovacao.md).
- ✅ Tipos de quiz no MVP → 3 modos: simulado TAP completo (respeitando `tap_weight`), livre por matéria, livre por eixo. "Pontos fracos" e filtro por dificuldade adiados (QUIZ-P-04/05). Cronômetro opcional com tempo total. Justificativa configurável (após cada questão / só no final). QUIZ-RF-001..005.
- ✅ Reset de estatísticas pelo cliente → sim, total, com reautenticação e confirmação; preserva histórico de quizzes. QUIZ-RF-008.
- ✅ Comportamento de quiz abandonado vs. finalizado/expirado → assimetria intencional: `finished`/`expired` contam não-respondidas como erro; `abandoned` preserva apenas o que foi efetivamente respondido. Auto-abandono após 24h sem atividade. QUIZ-RF-001 CA-6 / QUIZ-RF-004 CA-4.
- ✅ Evolução temporal do desempenho próprio → endpoint mensal (`/me/performance/timeline`), sem leaderboard entre clientes. QUIZ-RF-010.
- ✅ Duração do período gratuito → **7 dias** corridos a partir da verificação de e-mail. Único por usuário. SUB-RF-002.
- ✅ Meios de cobrança no MVP → PIX, saldo Mercado Pago (mesmo preço base) e cartão de crédito (preço +10%, parcelável em até 3×). Sem recorrência automática (cliente renova manualmente após e-mail de lembrete D-7/D-3/D-1). SUB-RF-001, SUB-RF-003, SUB-RF-007.
- ✅ Cortesia de assinatura: revogação e limites → admin pode revogar cortesia não consumida (com motivo obrigatório); limite de **10 cortesias/mês civil por admin**. SUB-RF-008/010. Termo "doação" foi renomeado para "cortesia" em toda a documentação.
- ✅ Equivalência "questões cadastradas ↔ tempo de assinatura" → **sem regra automática no MVP** — admin observa o trabalho e doa manualmente. SUB-P-02.

### Em aberto
- Todos os módulos funcionais do MVP estão cobertos; pendências por módulo estão nos rodapés de `rf/*.md`. As pendências do Módulo 7 (AIGEN-P-01 a AIGEN-P-04, no rodapé de `rf/ai-generation.md`) ficaram órfãs em 2026-07-27 — o pipeline a que se referem foi retirado de produção nessa data (ver `decisoes.md` § ADR-0039) e não roda mais. Os demais módulos (1 a 5) têm suas pendências resolvidas (ver "Resolvidas" acima). **Módulo 6 (Assinaturas) é hoje o único módulo com lacuna real de implementação** — a especificação está madura e sem pendências abertas (`rf/subscriptions.md`), falta apenas o código.
- **Pendências não-funcionais/de negócio, bloqueiam o lançamento público** (detalhe de cada item em `docs/tarefas.md`):
  - 🚩 **Política de Privacidade + Termos de Uso** — ainda não redigidos. Beta privado pode rodar com versão preliminar; versão final exige revisão jurídica antes da abertura pública (sistema já tem `consent_version` e reaceite, PROF-RF-006).
  - 🚩 **CNPJ e regime tributário** (ADR-0019) — decisão em aberto; bloqueia também a emissão de NFS-e (manual vs. integração com emissor, fora do MVP) e a configuração do WhatsApp Cloud API (exige CNPJ/KYC).
  - **Webhook do Mercado Pago** — não configurado no painel do MP; pendente porque a integração de pagamentos (Módulo 6) ainda não existe.
  - **Provedor de WhatsApp** (Cloud API vs. Z-API) — decisão adiada conscientemente desde ADR-0012; WhatsApp não é canal ativo no MVP.
