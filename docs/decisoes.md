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

## 0006 — Doação de assinatura como funcionalidade de primeira classe (2026-05-28)

**Contexto:** Parceiros recebem acesso gratuito em troca de cadastro de conteúdo. Em vez de criar um "tipo de assinatura de parceiro" ou hardcodar a contrapartida, queremos um mecanismo genérico.
**Decisão:** Existe a funcionalidade **"Doar assinatura"** disponível ao Administrador. Doação tem beneficiário, período, motivo (categoria) e admin concedente registrados. Acumula sobre assinatura paga existente. Aparece no painel financeiro **separada** de receitas.
**Consequências:** Mesmo mecanismo atende parceria, marketing, cortesia e compensação — sem código duplicado. Exige campo "motivo" obrigatório para que relatórios financeiros distingam os usos. Doações também precisam contar para métricas (ex.: "X assinaturas doadas via parceria este mês").

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

**Contexto:** Domínio razoavelmente rico (usuários, quiz, questões, assinaturas, doações, estatísticas) com múltiplas integrações externas (pagamento, WhatsApp, e-mail, storage). Queremos testabilidade e capacidade de trocar adapters sem reescrever regras de negócio.
**Decisão:** Estrutura em quatro camadas:
- `domain/` — entidades e regras de negócio puras, sem dependências externas.
- `application/` — use cases / serviços que orquestram domínio e portas (interfaces).
- `infra/` — adapters concretos (Drizzle, Mercado Pago, WhatsApp, Resend, R2).
- `http/` — Hono routes, controllers, schemas Zod, middleware (RBAC).

A direção de dependência é sempre de fora para dentro: `http → application → domain`; `infra` implementa portas definidas em `domain` ou `application`. Detalhes em `docs/arquitetura.md`.

**Consequências:** Testes de domínio sem mocks de banco/HTTP. Trocar Drizzle por outro ORM, ou Mercado Pago por outro gateway, é localizado em `infra`. Custo: mais arquivos e indireção do que uma estrutura "rotas + serviços". O porte do projeto justifica o investimento porque há integrações reais e regras de negócio (doação acumulando assinatura, fórmula de nível da pergunta) que merecem isolamento.

## 0012 — Integrações externas iniciais (2026-05-28)

**Contexto:** Sistema precisa de pagamento recorrente, mensageria (verificação/notificação), e-mail transacional e armazenamento de imagens das questões.
**Decisão (inicial — alguns provedores ainda a validar):**
- **Pagamentos:** Mercado Pago (PIX + cartão, assinaturas recorrentes, doc em PT-BR, sem custo fixo).
- **WhatsApp:** WhatsApp Cloud API (Meta) como primeira opção; Z-API como fallback se o onboarding com a Meta for inviável.
- **E-mail transacional:** Resend (free 3.000 e-mails/mês, DX excelente) ou AWS SES como fallback se custos crescerem.
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
