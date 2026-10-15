# Conferência de RFs × código real (api / web / bot) — 2026-10-01

## Context
Pedido: comparar os 85 RFs de `requisitos/docs/rf/*.md` com o que realmente existe em `api/`, `web/` e `bot/`, ignorando o status declarado nos docs. Verificação feita por inventário de rotas (`api/src/http/routes/*`, `app.ts`), casos de uso (`api/src/application/*`), jobs (`api/src/index.ts`), hooks do Better-Auth (`infra/auth/better-auth.ts`), router e páginas do web (`web/src/app/router.tsx`, `web/src/pages/*`) e rotas/telas do bot.

Legenda: ✅ implementado (API + Web) · 🟡 parcial · ❌ não implementado · 🗑️ removido de api/web (ver coluna Bot)

## Resumo
Contagem atualizada em 2026-10-14, depois de parceiros (PROF-RF-011 a 013), cortesias, cupons, desativar/reativar conta (PROF-RF-007/008), detalhe de usuário (PROF-RF-014), purga de sessões (AUTH-RF-008) e retirada do saldo Mercado Pago (SUB-RF-003, ADR-0048). Em 2026-10-01 era 61 / 4 / 11 / 9.

| Situação | Qtde |
|---|---|
| ✅ Implementado | 75 |
| 🟡 Parcial | 0 |
| ❌ Não implementado | 1 |
| 🗑️ Removido (Módulo 7, coberto parcialmente pelo bot) | 9 |
| **Total** | **85** |

## Módulo 1 — Autenticação (auth.md)
| RF | Título | Situação | Evidência / observação |
|---|---|---|---|
| AUTH-RF-001 | Cadastro | ✅ | `POST /auth/register` · `/cadastro` |
| AUTH-RF-002 | Verificação de e-mail | ✅ | Rota nativa do Better-Auth · `/verificar-email` |
| AUTH-RF-003 | Reenvio de verificação | ✅ | `POST /auth/resend-verification` · `/reenviar-verificacao` |
| AUTH-RF-004 | Login | ✅ | `POST /auth/login` · `/login` |
| AUTH-RF-005 | Logout | ✅ | `POST /auth/logout` · nav-shell |
| AUTH-RF-006 | Solicitar recuperação | ✅ | `POST /auth/forgot-password` · `/esqueci-senha` |
| AUTH-RF-007 | Redefinir senha | ✅ | `POST /auth/reset-password` · `/redefinir-senha` |
| AUTH-RF-008 | Sessão e expiração | ✅ | 7 dias deslizantes, sessão única e `GET /me` ok. Job `session-purge` diário às 03:00 (CA-3, implementado em 2026-10-14) |
| AUTH-RF-009 | Rate limiting | ✅ | Bloqueio exponencial por (IP, e-mail) + limites de cadastro, recuperação e reenvio. Recuperação e reenvio: 3/h por e-mail + teto de 10/h por IP desde 2026-10-14 (ADR-0047) |
| AUTH-RF-010 | Promoção por whitelist | ✅ | Hook em `better-auth.ts` + `config/admin-whitelist.ts` |
| AUTH-RF-011 | Consentimento LGPD | ✅ | `consent_version` no cadastro · `/termos`, `/privacidade` |

## Módulo 2 — Perfil e papéis (profile.md)
| RF | Título | Situação | Evidência / observação |
|---|---|---|---|
| PROF-RF-001 | Ver perfil | ✅ | `GET /me/profile` · `/perfil` |
| PROF-RF-002 | Editar dados | ✅ | `PATCH /me/profile` |
| PROF-RF-003 | Trocar senha | ✅ | `POST /me/password` |
| PROF-RF-004 | Trocar e-mail | ✅ | `POST /me/email` + `/me/email/confirm` · `/perfil/email/confirmar` |
| PROF-RF-005 | Avatar | ❌ → ✅ 2026-10-15 | Nenhuma rota `/me/avatar` e nenhuma tela. Só existe o campo `avatarUrl`, sempre `null`. **Resolvido em 2026-10-15** (ver `tarefas.md`) |
| PROF-RF-006 | Reaceite de termos | ✅ | `POST /me/consent` · `/consentimento` |
| PROF-RF-007 | Desativar conta | ✅ | Corrigido em 2026-10-13: o login de conta `inactive` responde 403 `account_inactive` depois de conferir a senha (antes só barrava `deleted`). A migration 0027 encerra as sessões de contas que já estavam `inactive` |
| PROF-RF-008 | Reativar conta | ✅ | 2026-10-13: `POST /auth/reactivate` e tela "Sua conta está desativada" no login. 401 genérico em vez do 404 da E-1 (aditivo em `rf/profile.md`) |
| PROF-RF-009 | Excluir conta (LGPD) | ✅ | `DELETE /me` · seção "zona de perigo" do perfil |
| PROF-RF-010 | Sessão única | ✅ | `revocation_reason=session_replaced` + 401 tratado no web |
| PROF-RF-011 | Buscar/listar usuários (admin) | ✅ (2026-10-13) | `GET /admin/users` · `/painel/usuarios` |
| PROF-RF-012 | Promover a Parceiro | ✅ (2026-10-13) | `PATCH /admin/users/:id/role` com `role=partner` · `/painel/usuarios` |
| PROF-RF-013 | Revogar Parceiro | ✅ (2026-10-13) | `PATCH /admin/users/:id/role` com `role=client` · `/painel/usuarios` |
| PROF-RF-014 | Detalhe de usuário (admin) | ✅ | 2026-10-13: `GET /admin/users/:id` com o bloco `contract` (acesso, últimas 5 assinaturas, cortesias do ano) e tela `/painel/usuarios/:id` com as ações de papel e cortesia |

## Módulo 3 — Conteúdo admin (content-admin.md)
| RF | Título | Situação | Evidência / observação |
|---|---|---|---|
| CONT-RF-001..004 | Eixos (listar/criar/editar/arquivar) | ✅ | `admin-axes.routes.ts` · `/painel/eixos` |
| CONT-RF-005..008 | Matérias (listar/criar/editar/arquivar) | ✅ | `admin-subjects.routes.ts` · `/painel/materias` |
| CONT-RF-009 | Listar perguntas | ✅ | `GET /admin/questions` (+ `/:id`) · `/painel/perguntas` |
| CONT-RF-010 | Criar pergunta | ✅ | `POST /admin/questions` (`as_draft`) |
| CONT-RF-011 | Editar pergunta | ✅ | `PATCH` com `reset_stats` no formulário |
| CONT-RF-012 | Arquivar/excluir pergunta | ✅ | `/archive` + `DELETE` |
| CONT-RF-013 | Imagem da pergunta | ✅ | `POST/DELETE /admin/questions/:id/image` (Hono puro) · `image-upload-field` |
| CONT-RF-014 | Fila de revisão | ✅ | `GET /admin/questions/pending` · `/painel/revisao` |
| CONT-RF-015 | Aprovar | ✅ | `/approve` + e-mail `sendQuestionApproved` |
| CONT-RF-016 | Rejeitar | ✅ | `/reject` + `reject-question-dialog` + e-mail |
| CONT-RF-017 | Dificuldade diária | ✅ | Job `question-difficulty` (implementado hoje, ADR-0045) + selo de dificuldade nas listas |
| CONT-RF-018 | Publicadas da matéria | ✅ | `GET /admin/questions/published`, usado pelo bot (`bomberquiz-client.ts`) |

## Módulo 4 — Conteúdo parceiro (content-partner.md)
| RF | Título | Situação | Evidência / observação |
|---|---|---|---|
| PART-RF-001 | Listar próprias | ✅ | `GET /me/questions` · `/parceiro/perguntas` |
| PART-RF-002 | Criar rascunho | ✅ | `POST /me/questions` |
| PART-RF-003 | Editar própria | ✅ | `PATCH /me/questions/:id` |
| PART-RF-004 | Enviar para revisão | ✅ | `/submit` (limite de 30 por hora por usuário) |
| PART-RF-005 | Excluir própria | ✅ | `DELETE` (só `draft`) |
| PART-RF-006 | Imagem própria | ✅ | `POST/DELETE /me/questions/:id/image` |
| PART-RF-007 | Histórico de revisão | ✅ | `/review-history` · `/parceiro/perguntas/:id` |
| PART-RF-008 | Painel do parceiro | ✅ | `/me/partner/dashboard` · `/parceiro/inicio` (contador `unread_review_events`) |

## Módulo 5 — Quiz (quiz.md)
| RF | Título | Situação | Evidência / observação |
|---|---|---|---|
| QUIZ-RF-001 | Iniciar quiz | ✅ | 3 modos, cronômetro com seletor de tempo e `explanation_mode` · `/quiz/iniciar` |
| QUIZ-RF-002 | Responder | ✅ | `POST /quizzes/:id/answers` |
| QUIZ-RF-003 | Finalizar | ✅ | `POST /quizzes/:id/finish` |
| QUIZ-RF-004 | Encerramento automático | ✅ | Job `quiz-expire-and-abandon` |
| QUIZ-RF-005 | Resultado | ✅ | `/quiz/:id/resultado` |
| QUIZ-RF-006 | Histórico | ✅ | `GET /me/quizzes` · `/historico` |
| QUIZ-RF-007 | Painel de desempenho | ✅ | `GET /me/performance` · `/desempenho` |
| QUIZ-RF-008 | Zerar estatísticas | ✅ | `POST /me/performance/reset` |
| QUIZ-RF-009 | Bloqueio sem assinatura (402) | ✅ | `requireActiveAccess` + tratamento central do 402 em `app.tsx` |
| QUIZ-RF-010 | Evolução temporal | ✅ | `/me/performance/timeline` |

## Módulo 6 — Assinaturas (subscriptions.md)
| RF | Título | Situação | Evidência / observação |
|---|---|---|---|
| SUB-RF-001 | Planos (público + admin) | ✅ | `GET /plans`, `GET/PATCH /admin/plans` · `/planos`, `/painel/planos` |
| SUB-RF-002 | Trial automático | ✅ | `StartTrialUseCase` disparado no hook de verificação do Better-Auth |
| SUB-RF-003 | Checkout | ✅ | PIX e cartão funcionam, com cupom desde 2026-10-13. Saldo Mercado Pago retirado da especificação em 2026-10-14 (ADR-0048) |
| SUB-RF-004 | Webhook do Mercado Pago | ✅ | `POST /webhooks/mercado-pago` |
| SUB-RF-005 | Minha assinatura | ✅ | `GET /me/subscription` · `/assinatura` |
| SUB-RF-006 | Histórico de pagamentos | ✅ | `GET /me/payments` · `/assinatura/pagamentos` |
| SUB-RF-007 | Lembretes de expiração | ✅ | Job `subscription-expire-and-remind` (D-7, D-3, D-1 e D-0) |
| SUB-RF-008 | Conceder cortesia | ✅ | Implementado em 2026-10-13: `POST /admin/courtesies` · `/painel/cortesias` |
| SUB-RF-009 | Listar cortesias | ✅ | Implementado em 2026-10-13: `GET /admin/courtesies` · `/painel/cortesias` |
| SUB-RF-010 | Revogar cortesia | ✅ | Implementado em 2026-10-13: `POST /admin/courtesies/:id/revoke` · `/painel/cortesias` |
| SUB-RF-011 | `access_status` | ✅ | Porta `IAccessPolicy` + adapter Drizzle. O cache opcional do CA-4 não foi feito |
| SUB-RF-012 | Painel financeiro | ✅ | `/admin/financial/overview` + `/admin/payments` · `/painel/movimentacoes` (com CSV). Cortesias e `discounts_applied_cents` entregues em 2026-10-13 |
| SUB-RF-013 | Cupons (admin) | ✅ | Implementado em 2026-10-13: `POST/GET /admin/coupons`, `PATCH /admin/coupons/:id` · `/painel/cupons` |
| SUB-RF-014 | Reembolso | ✅ | `POST /me/payments/:id/refund` · botão no histórico de pagamentos |
| SUB-RF-015 | Aplicar cupom | ✅ | Implementado em 2026-10-13: `coupon_code` no checkout + prévia `POST /me/checkout/coupon-preview` · campo "Cupom de desconto" em `/planos` |

## Módulo 7 — Geração por IA (ai-generation.md)
Retirado de produção em 2026-07-27 (ADR-0039). Nenhum resquício em `api/src` nem em `web/src`. O bot é uma ferramenta local e **síncrona**: não tem jobs, fila nem persistência.
| RF | Título | API/Web | Bot |
|---|---|---|---|
| AIGEN-RF-001 | Criar job | 🗑️ | Substituído por `POST /generate` (síncrono, sem job) |
| AIGEN-RF-002 | Status/polling | 🗑️ | Não se aplica (não há job) |
| AIGEN-RF-003 | Histórico de jobs | 🗑️ | ❌ Sem equivalente |
| AIGEN-RF-004 | Processamento em segundo plano | 🗑️ | Substituído por processamento local (divisão em partes, duplicatas, recorte de norma) |
| AIGEN-RF-005 | Editar gerada | 🗑️ | ✅ `question-edit-dialog` |
| AIGEN-RF-006 | Aprovar individual | 🗑️ | ✅ Publica via `POST /admin/questions` (com opção de rascunho) |
| AIGEN-RF-007 | Descartar individual | 🗑️ | ✅ Remove localmente |
| AIGEN-RF-008 | Lote | 🗑️ | 🟡 Só "aprovar todas"; **não há descarte em lote** |
| AIGEN-RF-009 | Excluir job | 🗑️ | Não se aplica |

## Divergências encontradas na documentação
1. `requisitos/CLAUDE.md` diz que **painel financeiro e admin de planos** estão fora do MVP, mas os dois foram implementados (API e web).
2. `api.md` lista como se existissem `/me/avatar`, `/auth/reactivate`, `/admin/users*`, `/admin/courtesies*` e `/admin/coupons*`, sem avisar que não foram implementados. `/admin/financial/overview` também aparece duas vezes.
3. PROF-RF-007/008 constam como entregues no bootstrap do web (tarefas.md, 2026-07-07), mas a reativação ficou "fora de escopo por falta de endpoint" e não voltou a ser registrada como pendência. **Isso é um bug funcional, não só uma lacuna de documentação.** (Corrigido em 2026-10-13.)
4. AUTH-RF-009 CA-3 pede o limite por e-mail; o código limitava por IP. (Corrigido em 2026-10-14: por e-mail + teto por IP, ADR-0047.)

## Próximo passo (após aprovação)
Só se o usuário quiser, registrar essas pendências e divergências em `requisitos/docs/tarefas.md` (seção "A realizar") e marcar as rotas não implementadas em `requisitos/docs/api.md`. Nenhuma alteração de código está prevista nesta conferência.
