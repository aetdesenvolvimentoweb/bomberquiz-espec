# Visão Geral do Sistema — BomberQuiz

> Documento de síntese. Para detalhes, consulte as referências cruzadas no fim de cada seção.

---

## 1. O que é o BomberQuiz

O BomberQuiz é uma plataforma web/PWA de questões e quizzes voltada para candidatos ao **Teste de Avaliação Profissional (TAP)** do Corpo de Bombeiros Militar do Estado de Goiás (CBMGO). O objetivo é oferecer uma ferramenta de estudo eficaz, gamificada e acessível pelo celular.

**Modelo de negócio:** freemium com assinaturas pagas.

| Período | Acesso |
|---------|--------|
| 7 dias após verificação de e-mail | Trial gratuito completo |
| Após expiração do trial | Somente usuários com assinatura ativa ou cortesia |
| Planos | Mensal, trimestral, semestral, anual |

O conteúdo (eixos, matérias, perguntas) é dinâmico e gerenciado via interface administrativa — nenhum conjunto de matérias está fixo no código.

---

## 2. Atores e papéis

O sistema possui um único cadastro de usuário; o papel é uma propriedade, não um tipo separado (ADR-0007).

| Ator | Permissões principais |
|------|----------------------|
| **Administrador** | Acesso total: conteúdo, usuários, planos, finanças, cortesias. Máximo 5, definidos por whitelist de e-mails no ambiente. |
| **Parceiro** | Submete perguntas para revisão; acessa dashboard de contribuições e histórico de aprovações/rejeições. |
| **Cliente** | Faz quizzes, acompanha desempenho próprio, gerencia perfil e assinatura. |

> Referência: `docs/requisitos.md` (seção Atores) · `docs/decisoes.md` (ADR-0005, ADR-0007)

---

## 3. Hierarquia de conteúdo

```
Eixo Temático
  └── Matéria
        └── Pergunta
              ├── 4 alternativas (1 correta)
              ├── Justificativa (obrigatória)
              ├── Fonte oficial (opcional, texto livre)
              └── Imagem (opcional, Cloudflare R2)
```

- Eixos e matérias podem ser **ativados/desativados** sem exclusão, acompanhando o ciclo anual do edital do TAP.
- Perguntas têm **nível de dificuldade calculado automaticamente** (job diário às 00:00) com base no histórico de acertos/erros.
- Cada matéria carrega um peso (`tap_weight`) usado para compor o simulado TAP proporcional.

> Referência: `docs/rf/content-admin.md` · `docs/rf/content-partner.md` · `docs/decisoes.md` (ADR-0003, ADR-0004, ADR-0016)

---

## 4. Módulos funcionais

O sistema tem **7 módulos** com aproximadamente **80 endpoints** na API.

### M1 — Autenticação
Registro com verificação de e-mail, login/logout, redefinição de senha por e-mail, sessão única (Netflix-style: novo login invalida sessões anteriores).

### M2 — Perfil
Atualização de dados pessoais, avatar (Cloudinary), troca de e-mail e senha com cooldown anti-takeover, consentimento LGPD versionado, desativação reversível e exclusão definitiva com anonimização (ADR-0015).

### M3 — Conteúdo (Admin)
CRUD completo de eixos, matérias e perguntas; publicação direta sem fila; aprovação/rejeição de contribuições de parceiros (motivo obrigatório na rejeição); gerenciamento de imagens (R2); ajuste automático de dificuldade via job.

### M4 — Conteúdo (Parceiro)
Submissão de perguntas (até 50 rascunhos simultâneos); edição de publicadas devolve à fila `pending_review`; exclusão restrita a rascunhos; notificação de revisão via badge no app e e-mail transacional; dashboard com estatísticas das próprias contribuições.

### M5 — Quiz
Três modos: **simulado TAP** (proporcional ao `tap_weight`, até 60 questões), **livre por matéria** e **livre por eixo**. Cronômetro opcional (1–5 min/questão, padrão 3 min). Sessão online-only, sem pause/resume. Estatísticas de desempenho por matéria, histórico de quizzes e timeline de evolução temporal. Reset total preserva histórico de sessões concluídas.

### M7 — Geração de Questões por IA
Exclusivo para admins. Admin faz upload de dois PDFs: uma prova real do TAP (referência de estilo e complexidade) e o material de estudo (fonte do conteúdo). Escolhe a matéria de destino e a quantidade de questões (1–30). O sistema processa de forma assíncrona via Claude API (ADR-0022): extrai texto dos PDFs, identifica exemplos de questões na prova de referência, monta o prompt e gera as questões em JSON estruturado. O admin revisa questão a questão — pode editar, aprovar (vai direto para `published`) ou descartar individualmente ou em lote. Aceita apenas PDFs com texto selecionável (ADR-0021). **Otimizações de custo:** exemplos extraídos da prova de referência são cacheados em banco por hash do arquivo — o mesmo PDF de referência não é reprocessado em jobs futuros (ADR-0025); o bloco `[system + exemplos]` do prompt usa `cache_control` na API Claude para reduzir custo em jobs multi-chunk (ADR-0026).

### M6 — Assinaturas
Checkout via Mercado Pago (PIX, saldo MP, cartão em até 3×); sem recorrência automática no MVP. Lembretes de expiração por e-mail (D-7, D-3, D-1, D-0). Cortesias de assinatura (parceria/demonstracao) administradas com limite de 10/mês. Cupons de desconto. Reembolso automático em até 7 dias (CDC art. 49). Painel financeiro admin com separação entre receitas de vendas e de cortesias.

> Referência: `docs/rf/` (um arquivo por módulo) · `docs/api.md` (inventário de endpoints)

---

## 5. Arquitetura e partes do sistema

O sistema é dividido em **dois repositórios independentes**, conectados por um contrato OpenAPI.

### 5.1 Backend — `bomberquiz-api`

| Camada | Conteúdo |
|--------|---------|
| **domain** | Entidades, value objects, regras de negócio puras |
| **application** | Use cases (orquestram domain + portas) |
| **infra** | Adapters: banco (Drizzle), e-mail (Resend), storage (R2), pagamentos (MP), scheduler |
| **http** | Rotas Hono, middleware, validação Zod, geração do spec OpenAPI |

**Tecnologias principais:**
- Runtime: **Bun**
- Framework HTTP: **Hono** + `@hono/zod-openapi` (gera spec OpenAPI canônica)
- ORM: **Drizzle** sobre **PostgreSQL / Neon**
- Autenticação: **Better-Auth** (fronteira com código customizado a ser definida no spike — ADR-0018)
- Scheduler in-process (ADR-0017) com 5 jobs:
  - `00:00` — recalcular dificuldade das perguntas
  - `09:00` — lembretes de expiração de assinatura
  - `a cada 5 min` — expirar quiz sessions sem atividade há 24h
  - periódico — purga de sessões expiradas
  - periódico — dump lógico de backup para R2
- Hospedagem: **Fly.io**

### 5.2 Frontend — `bomberquiz-web`

| Responsabilidade | Tecnologia |
|-----------------|-----------|
| Build e bundling | Vite + TypeScript |
| UI | React + Tailwind CSS + shadcn/ui |
| Roteamento | React Router |
| Cache de dados | TanStack Query |
| Estado global | Zustand |
| PWA | vite-plugin-pwa (service worker + manifest) |
| Cliente HTTP | Gerado automaticamente a partir do spec OpenAPI do backend |

Mobile-first, projetado para uso via celular. Hospedado em **Cloudflare Pages**.

### 5.3 Contrato entre API e Web

O backend gera a spec OpenAPI via `@hono/zod-openapi`. O frontend consome essa spec para gerar um cliente HTTP type-safe automaticamente no CI. Isso garante que qualquer quebra de contrato seja detectada em tempo de build.

> Referência: `docs/arquitetura.md` · `docs/decisoes.md` (ADR-0008 a ADR-0012)

---

## 6. Dependências externas

| Serviço | Finalidade | Status no MVP |
|---------|-----------|--------------|
| **Neon** (Postgres serverless) | Banco de dados principal; PITR nativo para backup contínuo | Ativo |
| **Fly.io** | Hospedagem e execução da API (containers Bun) | Ativo |
| **Cloudflare Pages** | Hospedagem e CDN do frontend PWA | Ativo |
| **Cloudflare R2** | Armazenamento de imagens de perguntas e dumps de backup | Ativo |
| **Cloudinary** | CDN e transformações de avatares de usuários (ADR-0013) | Ativo |
| **Mercado Pago** | Gateway de pagamentos (PIX, cartão, saldo MP) + webhooks | Ativo |
| **Resend** | E-mails transacionais (verificação, recuperação, alertas, recibos) | Ativo |
| **WhatsApp Cloud API** | Notificações por WhatsApp | Adiado pós-MVP |

**Custo inicial previsto: R$ 0** — todos os provedores operam dentro de free tiers na fase inicial.

> Referência: `docs/decisoes.md` (ADR-0012, ADR-0013, ADR-0017, ADR-0020)

---

## 7. Modelo de dados (entidades principais)

| Tabela | Descrição |
|--------|-----------|
| `users` | Cadastro unificado com campo `role` (client/partner/admin) |
| `sessions` | Sessões de autenticação (sessão única por usuário) |
| `axes` | Eixos temáticos |
| `subjects` | Matérias (pertencentes a um eixo, com `tap_weight`) |
| `questions` | Perguntas com alternativas, justificativa, dificuldade calculada e snapshot para uso em quiz |
| `quiz_sessions` | Sessões de quiz com modo, status e snapshot das perguntas sorteadas |
| `subscription_plans` | Configuração de planos (admin pode alterar preços/duração) |
| `subscriptions` | Assinaturas pagas por usuário |
| `payments` | Registro de transações individuais |
| `courtesies` | Cortesias de acesso concedidas por admins |
| `coupons` | Cupons de desconto |
| `audit_log` | Log imutável de ações sensíveis (expurgo de IP após 12 meses) |
| `ai_generation_jobs` | Jobs de geração de questões por IA (status, metadados, contagem de tokens) |
| `ai_generated_questions` | Questões geradas pelo LLM aguardando revisão/aprovação do admin |
| `ai_reference_exams` | Cache de exemplos extraídos de provas de referência, indexado por hash do arquivo (ADR-0025) |

> Referência: `docs/arquitetura.md` (seção Modelo de dados)

---

## 8. Aspectos transversais

### Segurança e conformidade
- Autenticação por cookie `httpOnly` (sem tokens no JS)
- RBAC: client < partner < admin
- Rate limiting: 60 req/min por IP · 120 req/min por usuário autenticado
- Conformidade: LGPD (consentimento versionado, anonimização, portabilidade) + OWASP Top 10
- Audit log para todas as ações sensíveis (com actor, action, target, payload)

### Comunicação com o usuário
- E-mails transacionais via Resend (verificação, recuperação de senha, alertas de sessão, lembretes de assinatura, resultados de revisão, recibos de pagamento, cortesias)
- Notificações in-app via badge `unread_review_events` para parceiros
- Suporte via `SUPPORT_EMAIL` (endereço configurável por ambiente)

### Disponibilidade e backup
- Disponibilidade-alvo: 99%
- Backup contínuo via PITR do Neon + dump lógico periódico para R2 (ADR-0020)

### Ambientes
- **Local:** banco local ou Neon dev branch; URLs hardcoded `.localhost`
- **Staging:** `*.pages.dev` / `*.fly.dev` (domínios provisórios)
- **Produção:** `*.bomberquiz.com.br` (fase de lançamento público)

---

## 9. Bloqueios e pendências críticas

| Item | Tipo | Detalhes |
|------|------|---------|
| Política de Privacidade + Termos de Uso | Jurídico/lançamento | Obrigatório antes da abertura pública (beta privado pode rodar com versão preliminar) |
| CNPJ e regime tributário | Jurídico/fiscal | Pré-requisito para NFS-e e para ativação do WhatsApp Cloud API (ADR-0019) |
| Spike Better-Auth | Técnico | Validar suporte a sessão única, lockout, cooldown e anonimização antes do bootstrap da API (ADR-0018) |

---

## 10. Referências cruzadas

| Documento | Conteúdo |
|-----------|---------|
| [`docs/requisitos.md`](requisitos.md) | Especificação de requisitos funcionais por módulo |
| [`docs/arquitetura.md`](arquitetura.md) | Diagrama de camadas, modelo de dados, jobs, e-mails, ambientes |
| [`docs/decisoes.md`](decisoes.md) | ADRs 0001–0026: contexto, decisão e consequências |
| [`docs/api.md`](api.md) | Inventário de endpoints, convenções de paginação/erro/auth |
| [`docs/rf/`](rf/) | Requisitos funcionais detalhados (CAs, pendências por módulo) |
| [`docs/tarefas.md`](tarefas.md) | Histórico de trabalho realizado e próximos passos |
