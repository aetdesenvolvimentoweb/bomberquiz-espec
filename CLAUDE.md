# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status atual do projeto

**Fase: Implementação em andamento, avançando módulo a módulo.** Os dois repositórios já existem e têm código. `bomberquiz-api` (produção em `api.bomberquiz.com.br` via Fly.io + Neon) tem Auth, Perfil, Conteúdo Admin (eixos/matérias/perguntas/fila de revisão), Quiz e Geração de Questões por IA completos — backend e frontend. `bomberquiz-web` (produção em `app.bomberquiz.com.br` via Cloudflare Pages) tem as telas correspondentes a todos esses módulos. Módulos ainda não implementados: Conteúdo Parceiro e Assinaturas. Ver `docs/tarefas.md` para o estado detalhado por fatia (slice) de cada módulo.

Existe também `ai-bot/` (`ai-bot/api` + `ai-bot/web`, neste mesmo diretório raiz, fora de `bomberquiz-api`/`bomberquiz-web`): ferramenta **local**, rodada só na máquina do admin, que refaz o fluxo de geração de questões por IA sem depender do Fly.io — motivada por um incidente de OOM em produção com PDFs de fonte malformada (ver `docs/tarefas.md`, 2026-07-27). Autentica contra o `bomberquiz-api` de produção via login real (mesma barreira de acesso de sempre) e publica perguntas aprovadas via `POST /admin/questions`, endpoint que já existia — nenhuma mudança foi feita no `bomberquiz-api` para viabilizar isso. Não é um módulo do MVP nem parte da spec de requisitos; é tooling interno.

Ao iniciar uma sessão, leia primeiro:
- [docs/requisitos.md](docs/requisitos.md) — especificação de requisitos
- [docs/arquitetura.md](docs/arquitetura.md) — arquitetura técnica, stack e estrutura dos repositórios
- [docs/decisoes.md](docs/decisoes.md) — ADRs (decisões técnicas e justificativas)
- [docs/api.md](docs/api.md) — contrato da API: inventário de endpoints + convenções
- [docs/tarefas.md](docs/tarefas.md) — tarefas realizadas e a realizar

## Stack e topologia (resumo)

- **Dois repositórios independentes no GitHub:** `bomberquiz-api` (backend) e `bomberquiz-web` (frontend), ambos já criados e com código. Este diretório atual contém **apenas a documentação compartilhada**.
- **Backend:** Bun + Hono + Drizzle + Postgres (Neon) + Better-Auth, em arquitetura hexagonal. Hospedado no Fly.io.
- **Frontend:** Vite + React + Tailwind + shadcn/ui, PWA mobile-first. Hospedado em Cloudflare Pages.
- **Contrato entre eles:** OpenAPI gerado pelo backend (`@hono/zod-openapi`), consumido pelo frontend via cliente HTTP type-safe.
- **Integrações:** Mercado Pago (pagamentos), WhatsApp Cloud API, Resend (e-mail), Cloudflare R2 (imagens).

Detalhes completos em [docs/arquitetura.md](docs/arquitetura.md) e ADRs 0008–0012 em [docs/decisoes.md](docs/decisoes.md).

## Domínio

O BomberQuiz é um quiz das matérias cobradas no **Teste de Avaliação Profissional (TAP) do Corpo de Bombeiros Militar do Estado de Goiás (CBMGO)**. As matérias específicas serão introduzidas posteriormente — não assuma um conjunto fixo. Quando o usuário mencionar uma matéria nova, verifique se ela já está catalogada nos requisitos antes de tratá-la como existente.

## Idioma

Toda a documentação do projeto (requisitos, decisões, comentários em código quando necessários, mensagens de commit) é em **português brasileiro**. Responda ao usuário em português.

## Convenções de trabalho

- **Código de implementação já em andamento** nos repositórios `bomberquiz-api` e `bomberquiz-web` (fora deste diretório). Este diretório (`requisitos/`) continua sendo só a documentação compartilhada — não é onde o código mora.
- **Atualize `docs/tarefas.md` ao concluir trabalho não-trivial.** Marque o que foi feito e adicione novas pendências descobertas, em vez de deixá-las só no histórico da conversa.
- **Decisões com trade-off vão para `docs/decisoes.md`** (formato ADR curto: contexto, decisão, consequências). Não enterre decisões em comentários de código ou em memórias.
- **Memória vs. documentação:** fatos do projeto que qualquer futura sessão precisa saber vão em `docs/`. Memórias persistentes (preferências do usuário, feedback recorrente) ficam no sistema de memória do Claude. Não duplique.
