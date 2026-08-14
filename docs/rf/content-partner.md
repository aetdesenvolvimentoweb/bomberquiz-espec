# Módulo 4 — Conteúdo (parceiro)

> Identificador dos RFs deste módulo: `PART-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- Cobre o que o **Parceiro** pode fazer com perguntas — exclusivamente as **próprias**. Eixos e matérias permanecem responsabilidade do admin (Módulo 3).
- A estrutura de dados de uma pergunta (4 alternativas, justificativa obrigatória, imagem opcional, etc.) é **a mesma** definida em [`content-admin.md`](content-admin.md) e ADR-0016 — este módulo não a redefine, apenas restringe permissões.
- **Filtro de matérias:** o parceiro pode cadastrar em **qualquer matéria `active`** — sem vinculação por especialidade no MVP. A garantia de qualidade vem da revisão admin (CONT-RF-014..016).
- **Workflow do parceiro:** `draft` → `pending_review` → `published`. Rejeição volta para `draft` com `rejection_reason` preenchido (CONT-RF-016). Re-submissão é livre (sem limite de iterações).
- **Edição de pergunta já publicada:** parceiro pode editar a própria pergunta `published`, mas a edição **devolve a pergunta para `pending_review`** e ela sai do catálogo ativo até nova aprovação. Decisão consciente para preservar revisão humana (ver pendência resolvida no rodapé).
- **Exclusão pelo parceiro:** apenas perguntas próprias em `draft` (incluindo as rejeitadas que voltaram para `draft`). Em outros status, o parceiro **não exclui** — admin é quem arquiva (CONT-RF-012).
- Todo acesso a endpoint deste módulo exige `role=partner` (admin também passa, herda as permissões). Acesso de cliente → HTTP 403.
- Mesmas convenções de auditoria, Zod e mensagens genéricas dos módulos anteriores. Listagens paginadas conforme `api.md` § Paginação, ordenação e filtros.

## Regras gerais

| Regra | Valor |
|---|---|
| Endpoints do parceiro | sob `/me/questions` (implicitamente filtrado por `author_id = self`) |
| Limite de rascunhos em aberto | 50 perguntas em `draft` simultaneamente por parceiro |
| Rate limit de submissão | 30 envios para revisão (`draft → pending_review`) por hora por parceiro |
| Rate limit de upload de imagem | 60 uploads/hora por parceiro |
| Tempo de retenção de rascunhos | indefinido no MVP (não auto-purga) |

---

## PART-RF-001 — Listar próprias perguntas

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.
**Pré-condições:** Sessão ativa com `role=partner`; conta `active`.

**Descrição:**
Lista todas as perguntas cadastradas pelo parceiro, em todos os status.

**Critérios de aceitação:**
- **CA-1:** `GET /me/questions` retorna `{ id, subject_id, subject_name, axis_name, statement_preview, status, has_image, created_at, submitted_at, published_at, rejection_reason, total_answers, accuracy, difficulty_level }`, paginado.
- **CA-2:** Filtros suportados: `status` (multi-select, padrão `draft,pending_review,published`), `subject_id`, `axis_id`, `q` (busca em enunciado).
- **CA-3:** Ordenação padrão: `updated_at DESC`. Aceita `sort=published_at|created_at|accuracy`.
- **CA-4:** `accuracy` e `total_answers` só são significativos para `published` ou perguntas que já tiveram passagem por `published`. Para `draft`/`pending_review` recém-criadas, exibe zero.
- **CA-5:** Resposta **não inclui** perguntas de outros autores nem de admins. Filtro por `author_id` é implícito.

**Erros previstos:**
- **E-1:** Acesso por usuário sem papel `partner` (e sem `admin`) → HTTP 403.

---

## PART-RF-002 — Criar pergunta (rascunho)

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.

**Descrição:**
Parceiro inicia o cadastro de uma nova pergunta. Toda nova pergunta nasce como `draft`, **nunca** vai direto para `published` (diferença de admin — CONT-RF-010).

**Critérios de aceitação:**
- **CA-1:** `POST /me/questions` com `{ subject_id, statement, alternatives[4], correct_index, explanation, source_reference?, image_url? }`. Mesmas validações de CONT-RF-010 (regras gerais do Módulo 3).
- **CA-2:** `subject_id` deve apontar para matéria `active`. Qualquer matéria ativa é permitida — sem vínculo por especialidade.
- **CA-3:** Em sucesso, pergunta nasce com `status=draft`, `author_id=<parceiro>`. HTTP 201 com payload completo. Entrada em `audit_log` com `action=create_draft_question`.
- **CA-3b:** `source` é sempre forçado para `manual` no servidor para submissões de parceiro, independentemente de qualquer valor enviado no corpo da requisição (campo descrito em `content-admin.md` § Estrutura de dados).
- **CA-4:** ~~Cadastros parciais são aceitos~~ **Revisado na implementação (Módulo 4, Slice 2):** `Question` (entidade compartilhada com o admin, Módulo 3) exige incondicionalmente as 4 alternativas distintas/não-vazias e `correct_index` 0-3 já na construção — não há instância "incompleta". Afrouxar esse invariante só para rascunho de parceiro exigiria mudança mais profunda no domínio compartilhado com o admin (decisão tomada com o usuário: não vale o risco). Na prática, **todo rascunho do parceiro já nasce completo**, igual ao admin (CONT-RF-010) — a única diferença real é `status` sempre `draft` (nunca `published` direto). Submissão para revisão (PART-RF-004) não precisa revalidar completude por esse motivo.
- **CA-5:** Limite de 50 rascunhos em aberto por parceiro (regras gerais). Acima disso → E-2.

**Erros previstos:**
- **E-1:** Matéria arquivada → HTTP 422.
- **E-2:** Limite de rascunhos excedido → HTTP 429 com mensagem "Você já tem 50 rascunhos em aberto. Submeta ou descarte alguns antes de criar novos."

---

## PART-RF-003 — Editar pergunta própria

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.
**Pré-condições:** Pergunta pertence ao parceiro (`author_id = self`).

**Descrição:**
Comportamento depende do status atual:
- `draft` (inclui rejeitada) → edição livre, sem mudança de status.
- `published` → edição é aceita mas **devolve a pergunta para `pending_review`** (admin reaprovará).
- `pending_review` → bloqueado (está em mãos do admin).
- `archived` → bloqueado (somente admin reverte, CONT-RF-012).

**Critérios de aceitação:**
- **CA-1:** `PATCH /me/questions/:id` com qualquer subconjunto editável de `{ subject_id?, statement?, alternatives?, correct_index?, explanation?, source_reference?, image_url? }`.
- **CA-2:** Pergunta `draft`: aplica edição, mantém `status=draft`. Se a pergunta tinha `rejection_reason`, ele é **preservado** até o próximo envio para revisão (PART-RF-004 limpa).
- **CA-3:** Pergunta `published`: aplica edição **e** transiciona `status` para `pending_review`. `submitted_at` é atualizado para o momento da edição; `reviewed_by`, `reviewed_at`, `rejection_reason` são limpos. A pergunta **deixa de aparecer em quizzes** até nova aprovação. Frontend exibe aviso explícito antes de salvar: "Esta pergunta voltará para revisão e ficará indisponível para clientes até a nova aprovação."
- **CA-4:** Pergunta `pending_review` ou `archived` → E-2 / E-3.
- **CA-5:** Pergunta de outro autor → E-1.
- **CA-6:** Reset de estatísticas? **Não** disparado automaticamente pelo parceiro (CONT-P-04 / CONT-RF-011 CA-4 — só admin decide). Se a edição mudar o gabarito, o admin (na revisão) marca `reset_stats=true` ao aprovar via CONT-RF-011 ou CONT-RF-015.
- **CA-7:** Entrada em `audit_log` com `action=edit_own_question` e diff resumido.

**Erros previstos:**
- **E-1:** Pergunta não pertence ao parceiro → HTTP 404 (não vaza existência).
- **E-2:** Pergunta em `pending_review` → HTTP 409, "Aguarde a revisão do admin antes de editar."
- **E-3:** Pergunta `archived` → HTTP 409, "Esta pergunta foi arquivada. Solicite ao admin para desarquivar."

---

## PART-RF-004 — Enviar pergunta para revisão

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.
**Pré-condições:** Pergunta pertence ao parceiro e está em `draft`.

**Descrição:**
Transiciona a pergunta de `draft` para `pending_review`, entrando na fila de aprovação do admin (CONT-RF-014).

**Critérios de aceitação:**
- **CA-1:** `POST /me/questions/:id/submit`.
- **CA-2:** **Validação completa** dos campos obrigatórios antes de aceitar: enunciado, exatamente 4 alternativas, `correct_index`, justificativa. Faltando algo → E-2.
- **CA-3:** Em sucesso, `status=pending_review`, `submitted_at=now()`. `rejection_reason` é limpo (era de uma rodada anterior). Entrada em `audit_log` com `action=submit_question`.
- **CA-4:** Rate limit: 30 submissões/hora por parceiro (regras gerais). Excedido → HTTP 429.
- **CA-5:** Notifica os admins (canal a definir junto ao Módulo 6 / notificações — provavelmente badge no painel de pendências).

**Erros previstos:**
- **E-1:** Pergunta não pertence ao parceiro → HTTP 404.
- **E-2:** Campos obrigatórios incompletos → HTTP 422 com a lista do que falta.
- **E-3:** Pergunta não está em `draft` → HTTP 409 ("Já está em revisão" / "Já está publicada").

---

## PART-RF-005 — Excluir pergunta própria (rascunho/rejeitada)

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.
**Pré-condições:** Pergunta pertence ao parceiro e está em `draft`.

**Descrição:**
Hard-delete de rascunho ou pergunta rejeitada (que volta para `draft`). Como pergunta em `draft` por definição **nunca foi publicada**, ela tem `total_answers=0` — exclusão definitiva é segura.

**Critérios de aceitação:**
- **CA-1:** `DELETE /me/questions/:id`.
- **CA-2:** Em sucesso, pergunta é removida fisicamente; imagem no Cloudinary é deletada (best-effort). Entrada em `audit_log` com `action=delete_own_draft` registrando ID, matéria e enunciado antes da exclusão (auditoria).
- **CA-3:** UI exige confirmação explícita.
- **CA-4:** Pergunta em qualquer outro status → E-2 (parceiro nunca exclui pergunta em `pending_review`, `published` ou `archived`).

**Erros previstos:**
- **E-1:** Pergunta não pertence ao parceiro → HTTP 404.
- **E-2:** Status diferente de `draft` → HTTP 409, "Apenas rascunhos podem ser excluídos. Para outros status, peça ao admin."

---

## PART-RF-006 — Upload de imagem em pergunta própria

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.

**Descrição:**
Faz upload da imagem opcional da própria pergunta. Espelha CONT-RF-013, restrito às próprias.

**Critérios de aceitação:**
- **CA-1:** `POST /me/questions/:id/image` (multipart). Mesmas validações de CONT-RF-013 (tipo, tamanho, MIME real).
- **CA-2:** Pergunta deve pertencer ao parceiro. Caso contrário → HTTP 404.
- **CA-3:** Pergunta em `pending_review` ou `archived` → E-1 (siga a mesma regra de edição — não pode mexer enquanto admin revisa).
- **CA-4:** Em pergunta `published`, o upload **transiciona** a pergunta para `pending_review` (mesmo princípio de PART-RF-003 CA-3 — qualquer alteração de conteúdo volta para revisão). UI avisa antes de salvar.
- **CA-5:** Rate limit: 60 uploads/hora por parceiro (regras gerais).
- **CA-6:** `DELETE /me/questions/:id/image` segue as mesmas regras de status.

**Erros previstos:**
- **E-1:** Status incompatível com edição → HTTP 409.
- **E-2:** Tipo/tamanho inválido → HTTP 422 / HTTP 413.
- **E-3:** Falha no Cloudinary → HTTP 502.

---

## PART-RF-007 — Visualizar histórico de revisão da pergunta

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.

**Descrição:**
Mostra ao parceiro o histórico das revisões de uma pergunta dele — quem aprovou, quando, notas eventuais, e o motivo de cada rejeição que houve.

**Critérios de aceitação:**
- **CA-1:** `GET /me/questions/:id/review-history` retorna lista ordenada cronologicamente: cada entrada com `{ event, at, by_admin_name, notes?, rejection_reason? }`. Eventos possíveis: `submitted`, `approved`, `rejected`, `edit_after_published` (transição automática gerada por PART-RF-003 CA-3).
- **CA-2:** Origem dos dados: `audit_log` filtrado pelo `target_id` da pergunta + estado atual (`reviewed_by`, `reviewed_at`, `rejection_reason` em `questions`).
- **CA-3:** Endpoint não retorna dados de admins de forma identificável além do nome — não expõe e-mail nem IDs internos.
- **CA-4:** Acessar o histórico marca todos os eventos de revisão da pergunta como **lidos** para fins do contador `unread_review_events` em PART-RF-008. Implementado via timestamp `last_seen_review_events_at` por par (usuário, pergunta), ou estratégia equivalente.

**Erros previstos:**
- **E-1:** Pergunta não pertence ao parceiro → HTTP 404.

---

## PART-RF-008 — Painel/Dashboard do parceiro

**Prioridade:** Essencial (MVP).
**Ator:** Parceiro.

**Descrição:**
Tela inicial do parceiro com agregados motivacionais e operacionais: quantas perguntas em cada status, total de respostas geradas, acurácia média das próprias publicadas, distribuição por matéria e por nível de dificuldade.

**Critérios de aceitação:**
- **CA-1:** `GET /me/partner/dashboard` retorna:
  - `counts`: `{ draft, pending_review, published, archived }`.
  - `engagement`: `{ total_answers_received, avg_accuracy }` — somando as próprias `published`.
  - `by_subject`: array `[{ subject_id, subject_name, count_published, total_answers, avg_accuracy }]`.
  - `by_difficulty`: `{ easy, medium, hard, unrated }` — contagem de publicadas por banda (alimentada por CONT-RF-017).
  - `last_published`: `[{ id, statement_preview, published_at }]` — últimas 5.
  - `last_rejected`: `[{ id, statement_preview, rejection_reason, rejected_at }]` — últimas 5 (para o parceiro priorizar correções).
  - `unread_review_events`: inteiro — quantidade de eventos de aprovação/rejeição que o parceiro **ainda não consultou** via PART-RF-007. Alimenta o badge global no app. Decrementa quando o parceiro abre o histórico da pergunta (PART-RF-007 CA-4).
- **CA-2:** Endpoint é leve (consultas agregadas) — alvo P95 < 300ms.
- **CA-3:** Acesso restrito ao parceiro (e admin, que vê os agregados da própria conta caso também cadastre).

---

## Pendências deste módulo — resolvidas em 2026-05-28

- ✅ **PART-P-01 — Limite de rascunhos.** Decidido: **50 rascunhos** em aberto por parceiro (mantém o valor inicial). Ajustar conforme uso real depois do MVP.
- ✅ **PART-P-02 — Notificação ao parceiro.** Decidido: **badge no app + e-mail transacional**. Badge alimentado por `unread_review_events` em PART-RF-008 CA-1; e-mail enviado via Resend em cada aprovação (CONT-RF-015 CA-4) e rejeição (CONT-RF-016 CA-3).
- ✅ **PART-P-03 — Visibilidade entre parceiros.** Decidido: **parceiro só vê as próprias perguntas** (mantém PART-RF-001 CA-5). Justificativa do cliente: a organização do trabalho entre parceiros (divisão por matéria/tema, evitar duplicação) é responsabilidade do admin via processo externo, não do sistema.
- ✅ **PART-P-04 — Rubrica de aprovação.** Decidido: esboço inicial criado em [`../rubrica-aprovacao.md`](../rubrica-aprovacao.md). Documento operacional vivo, refinado com a prática.
