# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status atual do projeto

**Fase: Implementação inicial em andamento.** Os dois repositórios já existem e têm código (`bomberquiz-api` com autenticação/perfil funcionando e deployado; `bomberquiz-web` com as telas correspondentes). `bomberquiz-api` já está em produção no Fly.io (`bomberquiz-api.fly.dev` + `bomberquiz-api-staging.fly.dev`), com Neon como banco. `bomberquiz-web` ainda não foi implantado no Cloudflare Pages. Ver `docs/tarefas.md` para o estado detalhado.

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
