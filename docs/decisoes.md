# Decisões — BomberQuiz

> Registro de decisões arquiteturais (ADR) em formato curto. Uma entrada por decisão.

Formato:

```
## NNNN — Título curto (YYYY-MM-DD)

**Contexto:** o problema/situação.
**Decisão:** o que foi decidido.
**Consequências:** trade-offs aceitos, o que isso facilita/dificulta.
```

---

## 0001 — Documentação e idioma em português (2026-05-28)

**Contexto:** Projeto voltado ao público brasileiro (candidatos do CBMGO). Usuário se comunica em português.
**Decisão:** Toda a documentação, requisitos e comunicação com o usuário em português brasileiro. Código (identificadores) pode ser em inglês conforme convenção da linguagem escolhida.
**Consequências:** Facilita revisão pelo usuário e por colaboradores futuros do domínio. Aumenta levemente o atrito ao integrar com ferramentas/exemplos em inglês.

## 0002 — Fase atual é apenas especificação (2026-05-28)

**Contexto:** Início do projeto, sem stack definida.
**Decisão:** Antes de escrever qualquer código, concluir um levantamento mínimo de requisitos em `docs/requisitos.md`. Stack técnica (runtime, storage, frontend) só será decidida após isso.
**Consequências:** Evita retrabalho por decisões prematuras. Atrasa o primeiro código rodando.

## 0003 — Nomenclatura "Eixo Temático" para o nível 1 (2026-05-28)

**Contexto:** O conteúdo do TAP é organizado em uma hierarquia de três níveis (agrupador → matéria → pergunta). O usuário propôs "Grupo Temático" para o nível 1, perguntando se havia nome melhor.
**Decisão:** Usar **"Eixo Temático"**. Alternativas avaliadas: "Grupo Temático" (proposta original, igualmente válida), "Área de Conhecimento" (muito amplo), "Bloco" (genérico demais).
**Consequências:** "Eixo Temático" é o termo padrão em editais de concurso público brasileiro, alinhando o vocabulário do app ao vocabulário que o candidato já encontra no edital. Renomeação é trivial se mudarmos de ideia (replace_all sobre `docs/`).

## 0004 — Hierarquia de conteúdo: Eixo → Matéria → Pergunta (2026-05-28)

**Contexto:** O TAP cobra matérias agrupadas por afinidade, cada matéria geralmente associada a uma fonte oficial (manual/norma/lei), com quantidade de questões por matéria definida pelo edital.
**Decisão:** Modelar o domínio em três níveis hierárquicos, todos cadastráveis em runtime (não enumerados em código):
1. **Eixo Temático** (ex: Salvamento)
2. **Matéria** — pertence a um eixo, tem fonte oficial e peso/quantidade de questões no TAP (ex: Salvamento Terrestre → Manual Operacional de Salvamento Terrestre)
3. **Pergunta** — pertence a uma matéria, com alternativas e resposta correta.
**Consequências:** Permite acomodar mudanças anuais do edital sem deploy. O peso por matéria viabiliza um modo "simulado TAP" que respeita a distribuição real da prova. Adiciona complexidade ao schema e às telas de cadastro.

## 0005 — Administradores via whitelist de e-mails (2026-05-28)

**Contexto:** A equipe administrativa será pequena (até 5 pessoas no início). É preciso impedir autoescalada de privilégios e simplificar o controle de quem é admin.
**Decisão:** O conjunto de administradores é definido por uma **whitelist de e-mails na configuração da aplicação** (não em UI de admin). Quando um e-mail whitelist faz seu primeiro login, o sistema o promove a Administrador automaticamente. Adicionar/remover admin exige alterar a configuração e fazer deploy/reload — não há UI.
**Consequências:** Simples, seguro, sem necessidade de tela de "gerenciar administradores". Limita escalabilidade da equipe administrativa, mas isso é desejável no momento. Se no futuro a equipe crescer (>5), reavaliar para um modelo de admin-promove-admin com auditoria.

## 0006 — Cortesia de assinatura como funcionalidade de primeira classe (2026-05-28)

> **Renomeação aplicada em 2026-05-28:** "doação" → "cortesia". Cliente preferiu "cortesia" pela conotação comercial neutra; "doação" pode soar como caridade ou trazer implicações fiscais indesejadas. Schema renomeado para `courtesies`, `source=courtesy`, `courtesy_id`.

**Contexto:** Parceiros recebem acesso gratuito em troca de cadastro de conteúdo. O admin também precisa conceder acesso gratuito em ações promocionais (demonstração). Em vez de criar um "tipo de assinatura de parceiro" ou hardcodar a contrapartida, queremos um mecanismo genérico.
**Decisão:** Existe a funcionalidade **"Conceder cortesia"** disponível ao Administrador. Cortesia tem beneficiário, período (dias), categoria (`parceria` ou `demonstracao`) e admin concedente registrados. Acumula sobre assinatura paga existente, estendendo o `end_at`. Aparece no painel financeiro **separada** de receitas. Admin pode revogar cortesia ainda não totalmente consumida.
**Consequências:** Mesmo mecanismo atende parceria (remunerar quem cadastra conteúdo) e demonstração (marketing/divulgação) — sem código duplicado. Exige campo `category` para que relatórios financeiros distingam os usos. Cortesias contam para métricas (ex.: "X cortesias concedidas via parceria este mês") e nunca como receita. Revogação permite ajuste se a contrapartida do parceiro não se concretizar.

## 0007 — Cadastro único de usuário; papel é propriedade (2026-05-28)

**Contexto:** Administrador, Cliente e Parceiro compartilham o mesmo conjunto de dados básicos. Modelar três cadastros separados criaria duplicação.
**Decisão:** Um único cadastro de usuário com o mesmo formulário (nome, e-mail único, telefone WhatsApp, data de nascimento, sexo, senha, avatar opcional). O **papel** (admin/cliente/parceiro) é uma propriedade do usuário, atribuída por regras do sistema (whitelist para admin, autocadastro para cliente, promoção pelo admin para parceiro).
**Consequências:** Simplifica o modelo de dados e a UI de autenticação. Exige cuidado com autorização (RBAC) por endpoint/tela. Permite que um mesmo usuário ganhe ou perca papéis sem recriar conta.

## 0008 — Dois repositórios separados (API e Web) (2026-05-28)

**Contexto:** Backend e frontend podem ser deployados em plataformas diferentes (Fly.io e Cloudflare Pages) e possuem ciclos de release independentes. Há o trade-off entre repositórios separados (operacionalmente simples) e monorepo (compartilhamento de tipos).
**Decisão:** Dois repositórios no GitHub: `bomberquiz-api` e `bomberquiz-web`. O **contrato** entre eles é a especificação OpenAPI gerada pelo backend; o frontend consome via cliente HTTP type-safe (gerado da spec ou usando `openapi-fetch`/`hey-api`). Sem package compartilhado.
**Consequências:** Deploy desacoplado, mais simples para hospedagens distintas. Risco de drift de tipos é mitigado pelo cliente gerado da OpenAPI. Custo é configurar a geração e mantê-la atualizada no CI do web.

## 0009 — Stack do backend: Bun + Hono + Drizzle + Postgres no Neon (2026-05-28)

**Contexto:** Backend REST em arquitetura limpa; porte esperado é baixo (centenas a poucos milhares de usuários ativos); usuário tem familiaridade com TS/Node e está explorando Bun.
**Decisão:**
- Runtime: **Bun** (TS-first sem build, performance, testing built-in).
- Framework: **Hono** (portável entre Bun/Node/Workers, type-safe, ecossistema maduro).
- ORM: **Drizzle** (type-safe, SQL-like, leve, transparente — bom para arquitetura hexagonal).
- Banco: **PostgreSQL** no **Neon** (free tier, scale-to-zero, branching para testes de integração).
- Autenticação: **Better-Auth** (e-mail/senha, verificação de e-mail, recuperação, sessões).
- Validação: **Zod** em todas as fronteiras (request/response/env).
- Documentação: **`@hono/zod-openapi`** gera OpenAPI a partir dos schemas Zod; UI via Scalar.
- Testes: **Vitest** (unit/integration); integração contra branch Neon dedicado.
- Hospedagem: **Fly.io** (sempre-on no free tier inicial, deploy via CLI).

**Consequências:** Stack moderna, custo zero no início, alta produtividade. Bun em produção tem menos cases que Node — risco aceito dado o porte. Migração futura para Node é trivial (Hono e Drizzle são runtime-agnostic). Neon free tier (0.5 GB) atende com folga; upgrade é barato se necessário.

## 0010 — Stack do frontend: Vite + React + Tailwind + shadcn (PWA) (2026-05-28)

**Contexto:** App majoritariamente autenticado (sem necessidade de SSR para SEO de páginas internas), mobile-first, com fluxos de quiz que se beneficiam de cache offline.
**Decisão:**
- Bundler: **Vite**.
- Framework: **React** + **React Router**.
- Estilo: **Tailwind CSS** + **shadcn/ui** (componentes copiados para o repo, sem dependência runtime).
- Estado servidor: **TanStack Query**. Estado UI: **Zustand**.
- PWA: **`vite-plugin-pwa`** (service worker, manifest, install prompt, cache de assets e questões já carregadas).
- Testes: **Vitest** + **React Testing Library** (unit/integration); **Playwright** (E2E).
- Hospedagem: **Cloudflare Pages**.

**Consequências:** Build estático, deploy em qualquer CDN, sem lock-in. Sem SSR — landing page pública e SEO precisarão ser tratados via metadados estáticos ou prerender se virar prioridade. Bundle menor que Next.js. Cloudflare Pages oferece preview deploys, ótimo para revisão.

## 0011 — Arquitetura hexagonal no backend (2026-05-28)

**Contexto:** Domínio razoavelmente rico (usuários, quiz, questões, assinaturas, cortesias, estatísticas) com múltiplas integrações externas (pagamento, WhatsApp, e-mail, storage). Queremos testabilidade e capacidade de trocar adapters sem reescrever regras de negócio.
**Decisão:** Estrutura em quatro camadas:
- `domain/` — entidades e regras de negócio puras, sem dependências externas.
- `application/` — use cases / serviços que orquestram domínio e portas (interfaces).
- `infra/` — adapters concretos (Drizzle, Mercado Pago, WhatsApp, Resend, R2).
- `http/` — Hono routes, controllers, schemas Zod, middleware (RBAC).

A direção de dependência é sempre de fora para dentro: `http → application → domain`; `infra` implementa portas definidas em `domain` ou `application`. Detalhes em `docs/arquitetura.md`.

**Consequências:** Testes de domínio sem mocks de banco/HTTP. Trocar Drizzle por outro ORM, ou Mercado Pago por outro gateway, é localizado em `infra`. Custo: mais arquivos e indireção do que uma estrutura "rotas + serviços". O porte do projeto justifica o investimento porque há integrações reais e regras de negócio (cortesia acumulando assinatura, fórmula de nível da pergunta) que merecem isolamento.

## 0012 — Integrações externas iniciais (2026-05-28)

**Contexto:** Sistema precisa de pagamento recorrente, mensageria (verificação/notificação), e-mail transacional e armazenamento de imagens das questões.
**Decisão (inicial — alguns provedores ainda a validar):**
- **Pagamentos:** Mercado Pago (PIX + cartão, assinaturas recorrentes, doc em PT-BR, sem custo fixo).
- **WhatsApp:** **decisão adiada conscientemente** — no MVP, WhatsApp não é canal ativo (sem OTP, sem recuperação de senha, sem notificações). O número é apenas contato declarado armazenado para uso futuro. Quando virar canal ativo (divulgação/marketing pós-MVP), a escolha será entre Cloud API (Meta) — oficial, gratuito até 1k conversas/mês, exige CNPJ/KYC — e Z-API — não-oficial, paga, setup rápido sem KYC. Decisão depende do status jurídico do negócio na época.
- **E-mail transacional:** **Resend** confirmado (free 3.000 e-mails/mês cobre o volume estimado do MVP de 300–500 usuários × ~10 e-mails/mês; DX excelente; React Email reaproveita stack do frontend). Migração para SES é localizada na porta `EmailSender` se volume crescer.
- **Storage de imagens de questões:** Cloudflare R2 (sem cobrança de egress, free 10 GB) — conteúdo estático, sem necessidade de transformações.
- **Storage de avatares:** ver ADR-0013.

Todas acessadas via **portas** definidas em `application/` ou `domain/`, implementadas em `infra/`.

**Consequências:** Free tiers cobrem todo o início. Trocas posteriores ficam limitadas a `infra/`. Risco: provedores podem mudar precificação — a abstração já está pronta para mitigar.

## 0013 — Avatares no Cloudinary; URL apenas no banco (2026-05-28)

**Contexto:** Avatares são imagens pequenas que precisam de transformações (crop quadrado, múltiplos tamanhos para diferentes telas, otimização WebP/AVIF automática). Diferente das imagens de questões (estáticas, sem transformação), avatares se beneficiam de um CDN de imagens com URL-based transforms.
**Decisão:**
- Banco armazena **apenas a URL pública** no campo `users.avatar_url` (nullable). Não armazena bytes.
- Upload e hospedagem em **Cloudinary** (free tier: 25 GB de storage / 25 mil transformações/mês — sobra com folga).
- Quando `avatar_url=null`, o frontend renderiza placeholder gerado client-side a partir das iniciais do nome (não consome nada externo).
- Cloudinary acessado por porta `AvatarStorage` em `application/`, implementação em `infra/storage/cloudinary.adapter.ts`.

**Consequências:** Avatares ficam responsivos e otimizados sem código de redimensionamento próprio. Adiciona um terceiro provedor de storage (R2 para questões, Cloudinary para avatares), mas com escopo claro de uso para cada. Migração para um único provedor no futuro é localizada na camada `infra/`.

## 0014 — Política de sessão única (2026-05-28)

**Contexto:** O cliente pediu comportamento "estilo Netflix" para desincentivar compartilhamento de conta (uma assinatura, um usuário ativo por vez). A modelagem original em AUTH-RF-008 CA-4 e AUTH-RF-005 CA-2 permitia múltiplas sessões simultâneas em vários dispositivos.
**Decisão:** Cada usuário tem **no máximo 1 sessão ativa simultânea**. Novo login bem-sucedido invalida automaticamente toda sessão anterior do mesmo `user_id`. O dispositivo descartado, na próxima requisição autenticada, recebe `HTTP 401 { reason: "session_replaced" }` e o frontend exibe "Você foi desconectado porque entrou em outro dispositivo". As regras alteradas estão consolidadas em PROF-RF-010; AUTH-RF-005 CA-2 e AUTH-RF-008 CA-4 foram reescritas para refletir essa política.
**Consequências:** Reduz compartilhamento informal de assinatura sem precisar de DRM ou device fingerprinting. UX tem o trade-off de quem esquece o logout em um dispositivo público — mas o aviso explícito no próximo login mitiga. Não há UI de "minhas sessões" no MVP porque, por construção, só há uma. A política de "todas as sessões ativas são invalidadas" em AUTH-RF-007 CA-3 (redefinição de senha) continua válida — apenas, em prática, "todas" = "a única".

## 0015 — Saída de conta: desativação reversível + exclusão por anonimização (2026-05-28)

**Contexto:** A LGPD (art. 18, VI) garante ao titular o direito de eliminação dos dados pessoais. Atender literalmente com hard-delete em cascata destruiria o histórico do sistema (questões cadastradas por parceiros, estatísticas, cortesias concedidas), o que prejudicaria outros usuários e os relatórios financeiros. Por outro lado, simplesmente "desativar" sem remover PII não atende o direito de eliminação.
**Decisão:** Oferecer **dois fluxos distintos** ao usuário:
- **Desativar conta** (PROF-RF-007/008): reversível, preserva 100% dos dados, encerra a sessão e bloqueia logins até reativação. Útil para quem só quer pausar.
- **Excluir conta** (PROF-RF-009): irreversível, anonimiza as PII em uma única transação — `name` → "Usuário excluído", `email` → `sha256(email)+"@deleted.local"` (preserva unicidade, não roteável, não re-identificável), `phone`/`dob`/`sex`/`avatar_url` → `null`, `password_hash` → `null`, `status` → `deleted`. **Mantém** `users.id` e as FKs apontando para ele (questões cadastradas, cortesias, estatísticas).

Portabilidade (art. 18, V) **não** é atendida via endpoint no MVP — fica como atendimento sob demanda documentado na política de privacidade. Reavaliar se houver volume.
**Consequências:** Adere à LGPD sem cascata destrutiva. O preço é (a) UI precisa diferenciar claramente os dois fluxos para o usuário não confundir desativar com excluir, (b) o sufixo `@deleted.local` é não-roteável e qualquer regra de unicidade de e-mail precisa ignorar essa faixa, (c) listagens administrativas precisam filtrar `status != 'deleted'` por padrão para não poluir a UI.

## 0016 — Estrutura mínima da pergunta no MVP (2026-05-28)

**Contexto:** A pergunta é a unidade central do conteúdo. Há trade-offs reais entre flexibilidade (nº variável de alternativas, múltiplas corretas, múltiplas imagens, versionamento) e simplicidade de modelagem/UI/correção. O TAP segue o padrão de prova de concurso brasileiro: 4 alternativas, 1 correta, sem múltipla escolha estendida.
**Decisão:** Para o MVP, a pergunta tem:
- **Exatamente 4 alternativas** (`alternatives[4]`), **1 única correta** (`correct_index ∈ {0,1,2,3}`).
- **Justificativa obrigatória** (`explanation`), exibida **após** o usuário responder. Reforça aprendizado e força o autor a fundamentar.
- **Referência à fonte** (texto livre, opcional) — campo `source_reference` tipo "NT-01 CBMGO, art. 5º, §2º". Sem modelagem hierárquica.
- **1 imagem opcional** por pergunta (R2, ≤ 2 MB, jpg/png/webp). Cobre diagramas/equipamentos/plantas.
- **Workflow diferente para admin e parceiro:** admin publica direto (`status=published`); parceiro envia para fila (`pending_review`); admin aprova ou rejeita. Detalhes em CONT-RF-014..016 e no Módulo 4.
- **Soft-delete** via `status=archived` — perguntas nunca são hard-deletadas (preserva estatísticas e respostas históricas).
- **Reset de estatísticas opcional** ao editar — flag `reset_stats` no PATCH; respostas antigas continuam armazenadas mas deixam de contar (`stats_reset_at`). Quem edita decide. UI sugere `true` se gabarito mudou.

**Consequências:** Modelagem simples (sem JSON-schema dinâmico para alternativas, sem versionamento de pergunta, sem múltipla correta). Espelha o TAP real. Limita uso para perguntas V/F ou de "marque todas" — provavelmente OK no domínio bombeiro/concurso, mas se o cliente decidir cobrir simulado de "marque todas as corretas", esse ADR é revisto. A divisão admin/parceiro de workflow exige campos `reviewed_by`, `reviewed_at`, `rejection_reason` no schema desde o início.

## 0017 — Jobs agendados via scheduler in-process (2026-05-29)

**Contexto:** Vários requisitos dependem de execução periódica: recálculo diário de dificuldade às 00:00 (CONT-RF-017), lembretes de expiração às 09:00 (SUB-RF-007), expiração/auto-abandono de quiz a cada ~5 min (QUIZ-RF-004) e purga de sessões inativas (AUTH-RF-008). O Fly.io não tem cron nativo e, no início, rodamos **1 instância sempre-on**. Alternativas: scheduler dentro do processo Bun/Hono; Fly scheduled machines (efêmeras); cron externo chamando endpoints protegidos.
**Decisão:** Usar um **scheduler in-process** no backend (lib leve de cron rodando no mesmo processo Bun). Cada job é um caso de uso em `application/` disparado pelo scheduler em `infra/scheduler/`, reutilizando a mesma lógica de domínio (não há duplicação de regra). Todos os jobs operam no fuso `America/Sao_Paulo` e devem ser **idempotentes** (já exigido pelos RFs). Horários/intervalos ficam em configuração (env), não hardcoded.
**Consequências:** Zero infra e custo extra; viável por haver 1 instância. Trade-off aceito: se a instância estiver fora do ar no horário do job, ele só roda quando voltar (mitigado pela idempotência e por jobs que varrem por estado, não por "disparo único"). **Quando escalar para múltiplas instâncias**, isso vira problema (jobs rodariam N vezes) — a migração será para um lock distribuído (`pg_advisory_lock` no próprio Postgres) ou Fly scheduled machines, localizada em `infra/scheduler/`. A escolha casa com a do rate limit in-memory (mesma premissa de instância única).

## 0018 — Fronteira Better-Auth × autenticação customizada (2026-05-29)

**Contexto:** O Módulo 1 (AUTH) e o Módulo 2 (PROF) especificam comportamentos de identidade que vão além do enxoval padrão de uma lib de auth: sessão única (PROF-RF-010), lockout exponencial por (IP, e-mail) (AUTH-RF-009), troca de e-mail com `pending_email` + alerta ao endereço antigo + cooldown anti-takeover de 30/7 dias (PROF-RF-004/009), versionamento de consentimento com reaceite (AUTH-RF-011/PROF-RF-006), anonimização LGPD (PROF-RF-009) e promoção a admin por whitelist (AUTH-RF-010). Era preciso decidir o que o Better-Auth resolve e o que é lógica própria, antes de assumir cobertura total.
**Decisão:** Better-Auth é a **base** de identidade. Mapeamento:
- **Better-Auth nativo:** e-mail/senha (argon2id via config), verificação de e-mail, recuperação de senha (`resetPasswordTokenExpiresIn=1h`, `revokeSessionsOnPasswordReset=true`), `changeEmail` com verificação, gestão e revogação de sessão, campos extras de usuário (`phone`, `dob`, `sex`, `status`, `role`, `consent_version`, etc.) via `additionalFields`.
- **Custom (hooks/use cases próprios sobre o Better-Auth):** (a) **sessão única** — hook pós-`signIn` que revoga as demais sessões do `user_id` marcando `revocation_reason=session_replaced`; (b) **lockout exponencial** — o rate limit nativo é de janela fixa; a escalada 1m→24h por (IP, e-mail) é lógica própria; (c) **cooldown de troca de e-mail (30d) e bloqueio de exclusão (7d)** — campos e checagens próprios em volta do `changeEmail`; (d) **alerta ao e-mail antigo** na troca; (e) **versionamento/reaceite de consentimento**; (f) **anonimização LGPD** (mantém o `users.id`, zera PII); (g) **promoção por whitelist** no login.
- **Spike obrigatório no bootstrap do `bomberquiz-api`:** validar que os hooks do Better-Auth (`before/after` de signIn/signUp, `databaseHooks`) expõem os pontos de extensão acima; caso algum não seja viável, esse subfluxo migra para implementação própria sobre as tabelas do Better-Auth (que são acessíveis via Drizzle).
**Consequências:** Aproveita o que a lib faz bem (tokens, hashing, sessão, verificação) sem reescrever, e isola o que é específico do domínio. Risco residual: acoplamento a APIs internas do Better-Auth nos hooks — mitigado por manter a lógica de domínio (cooldown, anonimização, lockout) em `application/`, com o Better-Auth atrás de uma porta de identidade sempre que possível. Confirmação fica para o spike, não para a especificação.

## 0019 — Nota fiscal (NFS-e) fora do escopo do MVP (2026-05-29)

**Contexto:** As assinaturas são pagas (Mercado Pago). O comprovante do MP (`mp_receipt_url`, SUB-RF-006) é prova de pagamento, **não** nota fiscal de serviço eletrônica (NFS-e). Emissão de NFS-e depende de CNPJ e do regime tributário (ex.: MEI tem regras próprias), ainda não fechados.
**Decisão:** **Não** emitir NFS-e automaticamente pelo sistema no MVP. Eventual emissão é processo **manual/externo**, conforme o regime tributário da pessoa jurídica quando este for definido. O sistema apenas mantém o histórico de pagamentos e o link de comprovante do MP. Integração com emissor (NFE.io, eNotas, etc.) fica como evolução pós-MVP, atrás de uma porta dedicada se vier.
**Consequências:** Remove dependência jurídica/tributária do caminho crítico de implementação. **Pendência de negócio/jurídica explícita** registrada em `docs/tarefas.md` como bloqueio a reavaliar antes (ou logo após) o lançamento público, junto da definição de CNPJ/regime. Risco: se a obrigação fiscal existir desde o primeiro pagamento, haverá emissão manual durante a janela inicial.

## 0021 — M7 Geração por IA: escopo, público e limitação a PDFs selecionáveis (2026-06-17)

**Contexto:** O Módulo 7 introduz geração automática de questões via LLM a partir de documentos PDF. Duas decisões imediatas: (a) quem pode usar; (b) quais formatos de PDF são aceitos.
**Decisão:**
- **Exclusivo para admins.** Admins têm papel editorial final (publicam direto, sem fila) e são o público natural para curadoria de conteúdo em volume. Abrir para parceiros adicionaria custo de API sem revisão adicional — o workflow do parceiro já exige revisão de conteúdo; adicionar geração de IA tornaria a rastreabilidade de responsabilidade editorial nebulosa.
- **Apenas PDFs com texto selecionável** (nativos/digitais). PDFs digitalizados/escaneados exigem OCR — tecnologia não presente no stack atual e que adiciona complexidade e custo não justificados para o MVP. Se o PDF não produzir texto extraível, o job falha com mensagem clara orientando o admin a usar o arquivo correto.
**Consequências:** Escopo claro e controlado. Limitação prática documentada — admins precisam ter a versão digital dos documentos (manuais operacionais, normas técnicas, regulamentos), o que é a norma para documentos oficiais do CBMGO distribuídos digitalmente. Suporte a OCR fica como evolução futura explícita.

## 0022 — Modelo LLM para geração de questões: Claude (Anthropic) (2026-06-17)

**Contexto:** O M7 precisa de um LLM para gerar questões estruturadas em português a partir de texto técnico. Opções avaliadas: Claude (Anthropic), GPT-4o (OpenAI), Gemini (Google).
**Decisão:** **Claude** (modelo `claude-sonnet-4-6` como padrão), via API da Anthropic. Justificativas: (a) Sonnet 4.6 tem janela de contexto de 200k tokens — suficiente para manuais completos sem chunking na maioria dos casos; (b) qualidade superior em português técnico e em seguir instruções de formato estruturado (JSON estrito); (c) consistência no domínio de educação/concurso; (d) o stack Claude Code já pressupõe familiaridade com a API Anthropic. Sonnet é o default pelo equilíbrio custo/qualidade; `claude-opus-4-8` é alternativa para casos que exijam mais elaboração (AIGEN-P-04).
**Consequências:** Nova dependência externa: `ANTHROPIC_API_KEY` em `.env`. Custo estimado de R$ 0,50–2,00 por job de 20 questões (Sonnet). Rate limiting de jobs (10/admin/dia, 2 simultâneos globais) controla gasto. Adicionar a chave ao checklist de bootstrap do `bomberquiz-api` e ao runbook de ambientes.

## 0023 — Processamento de jobs de geração: assíncrono in-process (2026-06-17)

**Contexto:** Uma chamada ao LLM com um PDF de 100 páginas pode levar 30–90 segundos. Manter a conexão HTTP aberta durante esse tempo é impraticável (timeouts de load balancer, UX ruim). Opções: SSE no mesmo request; webhook de callback; polling; job assíncrono persistido no banco.
**Decisão:** **Job assíncrono persistido no banco**, com polling pelo frontend. Fluxo: `POST /admin/ai-generation/jobs` retorna HTTP 202 imediatamente; o worker (mesmo scheduler in-process de ADR-0017) assume o job da fila; o frontend faz poll em `GET /admin/ai-generation/jobs/:id` a cada 3 s. Mesma arquitetura dos outros jobs do scheduler — sem nova infra.
**Consequências:** Simplicidade operacional (reusa o scheduler já planejado). O polling é aceitável para uso admin (não é user-facing em tempo real). Se o Fly.io restartar durante o processamento, o job fica em `processing` até o timeout de 5 min, quando é marcado como `failed` pelo job de manutenção (idempotente). Alternativas (SSE, WebSocket) ficam como evolução se polling mostrar problema de UX.

## 0024 — PDFs temporários em R2: excluídos após conclusão do job (2026-06-17)

**Contexto:** Os PDFs enviados pelo admin contêm material potencialmente sigiloso ou sob direitos autorais. Não há necessidade de retê-los após o processamento — o texto já foi extraído e as questões geradas. Mantê-los aumenta superfície de exposição e custo de storage.
**Decisão:** Ambos os PDFs (`reference_pdf` e `material_pdf`) são **excluídos do R2 imediatamente** após o job atingir `status=completed` ou `status=failed` (seja por conclusão normal, por erro ou por timeout). Não há "retry com o mesmo PDF" — novo job exige novo upload (AIGEN-P-02).
**Consequências:** Custo de storage ~zero (arquivos ficam no R2 por minutos a no máximo 5 min de timeout). Conformidade com princípio de minimização de dados (LGPD art. 6º, III). O `material_name` (nome original do arquivo) é preservado nos metadados do job para referência histórica, mas o conteúdo não.

## 0025 — Cache de exemplos de prova de referência entre jobs (2026-06-17)

**Contexto:** O mesmo PDF de prova de referência (ex.: prova real do TAP 2024) tende a ser reutilizado em múltiplos jobs de geração — um para cada matéria do edital. A cada job, o worker re-baixava o arquivo do R2, re-extraía as questões-exemplo e as incluía no prompt: trabalho idêntico pago múltiplas vezes em tempo de processamento e em tokens LLM.
**Decisão:** Tabela `ai_reference_exams` armazena as questões-exemplo extraídas indexadas por SHA-256 do conteúdo do PDF. O worker computa o hash ao baixar o arquivo do R2 e faz lookup antes de iniciar a extração. Cache hit → reutiliza os exemplos diretamente, atualiza `last_used_at`. Cache miss → extrai, persiste e reutiliza nas próximas ocorrências. Os PDFs continuam excluídos do R2 conforme ADR-0024 — apenas o texto extraído (estrutura leve em JSONB) fica no banco. Campo `reference_exam_id` em `ai_generation_jobs` mantém rastreabilidade de qual cache foi usado.
**Consequências:** Elimina custo de reprocessamento para provas reutilizadas sem mudança na UX. Dados armazenados são fragmentos de texto extraído (não o PDF), sem novo impacto de LGPD além do já tratado. Purga de entradas com `last_used_at` muito antigo pode ser adicionada ao job de manutenção se necessário.

## 0026 — Prompt caching via `cache_control` na API Claude (2026-06-17)

**Contexto:** Em jobs com material extenso, o worker divide o material em múltiplos chunks e faz várias chamadas ao LLM (AIGEN-RF-004 CA-6). Em cada chamada, o bloco `[system prompt + exemplos da prova]` é idêntico e representa percentual significativo dos tokens de entrada. A API Claude suporta `cache_control: { type: "ephemeral" }` em blocos de mensagem, permitindo reutilizar o prefixo já computado pelo modelo.
**Decisão:** Marcar o bloco `[system prompt + exemplos da prova de referência]` com `cache_control: { type: "ephemeral" }` em todas as chamadas ao LLM feitas pelo worker M7. Tokens em cache custam ~10% do preço normal de tokens de entrada após o primeiro hit. TTL do cache é de 5 minutos — suficiente para cobrir as chamadas de um único job multi-chunk processado em sequência. O campo `cached_tokens` em `ai_generation_jobs` registra quantos tokens foram lidos do cache (via `usage.cache_read_input_tokens` na resposta da API).
**Consequências:** Redução de custo proporcional ao número de chunks e ao tamanho relativo do cabeçalho. Jobs de único chunk beneficiam-se apenas em cenários de burst (múltiplos jobs processando cabeçalho semelhante em < 5 min). Implementação mínima — apenas o campo `cache_control` no request; sem mudança de lógica de negócio.

## 0020 — Backup e recuperação do Postgres: PITR Neon + dump lógico (2026-05-29)

**Contexto:** O produto guarda PII e histórico financeiro (pagamentos, cortesias, auditoria). Não havia política de backup. O Neon free tier tem janela curta de history (PITR) e nenhum backup independente do provedor.
**Decisão:** Dois níveis: (1) **PITR do Neon** para recuperação fina dentro da janela do tier; (2) **dump lógico agendado** (`pg_dump`) rodando como job (ADR-0017) e enviado ao **Cloudflare R2** (bucket separado das imagens), com retenção rotativa (ex.: 7 diários + 4 semanais). Restauração documentada como runbook no repo do backend. Reavaliar upgrade do Neon pago se a janela de PITR se mostrar insuficiente.
**Consequências:** Backup independente do provedor a custo ~zero (R2 free tier). Trade-off: o dump lógico não é point-in-time entre execuções (perda máxima = intervalo entre dumps); o PITR do Neon cobre o intervalo fino enquanto estiver na janela. O dump conterá PII — o bucket R2 deve ser privado e o objeto tratado conforme a LGPD (acesso restrito, mesma base legal dos dados originais).
