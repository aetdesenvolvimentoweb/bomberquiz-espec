# Módulo 6 — Assinaturas e cortesias

> Identificador dos RFs deste módulo: `SUB-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- Cobre **quatro frentes:** (1) cobrança de assinaturas pagas via Mercado Pago; (2) **cortesia de assinatura** como funcionalidade administrativa (ADR-0006 — renomeado de "doação"); (3) cupom de desconto no checkout (marketing inicial); (4) cálculo do `access_status` que rege o bloqueio em QUIZ-RF-009 e PROF-RF-014.
- Gateway de pagamento: **Mercado Pago** (ADR-0012).
- **Sem recorrência automática no MVP** — todo pagamento é pelo período total do plano. Cliente recebe lembrete de expiração e renova manualmente. Justificativa do negócio: TAP é cíclico (quem passa em um ano só presta de novo daqui a 3+ anos), o público é majoritariamente sazonal, recorrência automática não compensa a complexidade.
- Cortesia é **separada de receita** em todos os relatórios. Aparece com categoria obrigatória (ADR-0006).
- **Reembolso integral** dentro de **7 dias** da compra atende ao direito de arrependimento (CDC art. 49) — automatizado via UI; fora da janela, suporte manual.
- Acesso aos endpoints administrativos exige `role=admin`; aos endpoints de assinatura própria, sessão ativa do cliente; aos webhooks do Mercado Pago, validação de assinatura HMAC.
- Período gratuito (trial) de **7 dias** ao verificar e-mail. Único por usuário (não renovável).
- Toda transação financeira (sucesso ou falha) registra entrada em `audit_log` (schema em `arquitetura.md` § Audit log). Listagens paginadas conforme `api.md` § Paginação, ordenação e filtros.

## Regras gerais

| Regra | Valor |
|---|---|
| Duração do trial | **7 dias** corridos, contados a partir da verificação de e-mail (AUTH-RF-002) |
| Trial é único por usuário | `users.trial_used_at` registra; recadastro com mesmo e-mail anonimizado **não** ganha novo trial |
| Planos disponíveis | `monthly` (30d), `quarterly` (90d), `semiannual` (180d), `annual` (365d) — exatos em dias civis |
| Métodos de pagamento | **PIX**, **Saldo Mercado Pago**, **Cartão de crédito** |
| Diferencial de preço cartão | **+10% sobre o preço PIX** (precificação automática a partir do `pix_price` configurado pelo admin) |
| Parcelamento no cartão | até **3×** sem juros adicionais (custo de parcelamento já embutido no +10%) |
| Recorrência automática | **não** no MVP — todo pagamento cobre o período integral; renovação é ato manual do cliente |
| Lembretes de expiração | e-mails enviados em **D-7, D-3 e D-1** antes do `end_at` da assinatura ativa; **único e-mail final** ao expirar |
| Categorias de cortesia | `parceria`, `demonstracao` |
| Limite de cortesias por admin | **10 cortesias/mês civil** por administrador |
| Janela de cortesia | 1–365 dias |
| Janela de reembolso automático | **7 dias** corridos a partir do `paid_at` do pagamento (CDC art. 49) |
| Cupom de desconto | percentual ou valor fixo; com validade e limite de usos; aplicável no checkout |
| Status de pagamento | `pending` (PIX aguardando confirmação), `paid`, `failed`, `refunded` |
| Status de assinatura | `active`, `expired`, `revoked`, `pending_payment` |

## Estrutura de dados (resumo do domínio)

### `subscription_plans`
- `id`, `slug` (`monthly`/`quarterly`/`semiannual`/`annual`), `name`, `duration_days`, `pix_price` (centavos), `card_price` (centavos, derivado de `pix_price × 1.10` por padrão, editável), `max_installments` (default 3), `is_active`, `created_at`, `updated_at`.

### `subscriptions`
- `id`, `user_id`, `plan_id?` (null para cortesia), `source` (`paid`/`courtesy`/`trial`), `courtesy_id?`, `payment_id?`, `start_at`, `end_at`, `status`, `created_at`.

### `payments`
- `id`, `user_id`, `plan_id`, `method` (`pix`/`mp_balance`/`card`), `gross_amount` (centavos antes do cupom), `discount_amount` (centavos), `net_amount` (centavos efetivamente cobrados), `coupon_id?`, `installments`, `mp_payment_id`, `mp_status`, `status`, `paid_at?`, `refunded_at?`, `failure_reason?`, `created_at`, `updated_at`.

### `courtesies`
- `id`, `beneficiary_user_id`, `granted_by_admin_id`, `days_granted`, `start_at`, `end_at`, `category` (`parceria`/`demonstracao`), `notes?`, `revoked_at?`, `revoked_by_admin_id?`, `revocation_reason?`, `created_at`.

### `coupons`
- `id`, `code` (único case-insensitive), `discount_type` (`percent`/`fixed_cents`), `discount_value` (inteiro — % se `percent`, centavos se `fixed_cents`), `valid_from?`, `valid_until?`, `max_uses?`, `used_count`, `is_active`, `applies_to_plan_slugs?` (array, null = todos), `created_by_admin_id`, `created_at`.

### `user_access`
- Campo derivado/materializado por usuário: `access_status` (`active`/`inactive`), `active_until` (max `end_at` entre trial/subscription/courtesy atual), `source` (trial/paid/courtesy).

---

## SUB-RF-001 — Configurar e listar planos

**Prioridade:** Essencial (MVP).
**Ator:** Admin (configura); Cliente, Admin, Parceiro (lista).

**Descrição:**
Admin gerencia os planos vendidos. Qualquer usuário lê a lista de planos ativos.

**Critérios de aceitação:**
- **CA-1:** `GET /plans` é **público (sem sessão)** — um visitante não cadastrado consegue ver preços antes de criar conta (página de preços / decisão de compra). Retorna planos com `is_active=true`: `{ slug, name, duration_days, pix_price, card_price, max_installments }`. Preços em centavos para evitar float; cliente formata. Não expõe nenhum dado sensível; sujeito ao rate limit global por IP (60 req/min).
- **CA-2:** `PATCH /admin/plans/:id` (admin) altera `pix_price`, `card_price`, `is_active`, `max_installments`. Validação: `card_price ≥ pix_price`. UI sugere `card_price = round(pix_price × 1.10)` mas permite override.
  **Aditivos de implementação (2026-08-28):**
  - **`GET /admin/plans` acompanha o PATCH.** `GET /plans` (CA-1) só devolve ativos, então um plano desativado sumiria também da tela do admin — e não haveria como religá-lo pela UI. A rota do admin devolve o catálogo completo, com `id` (o PATCH endereça por id) e `is_active`.
  - **A validação `card_price ≥ pix_price` é avaliada sobre o resultado, não sobre o corpo do request.** Num PATCH parcial que manda só `pix_price` acima do `card_price` já gravado, comparar apenas os campos recebidos deixaria a inversão passar.
  - **Piso de 100 centavos** em ambos os preços: o Mercado Pago recusa cobrança abaixo de R$1,00 (mesmo piso que CA-3 de SUB-RF-015 aplica ao desconto). Sem ele, um preço de R$0,50 só falharia no checkout do cliente.
  - **`slug`, `name` e `duration_days` não são editáveis**, mesmo estando na tabela. Mudar a duração faria assinaturas já vendidas passarem a significar outro período, e o `slug` é a chave que o checkout usa.
  - **`GET /plans` passou a ordenar por `duration_days`.** Não ordenava, e o Postgres devolvia na ordem física das linhas — que um `UPDATE` altera. O primeiro reajuste feito pela UI jogou o plano mensal para o fim do catálogo público, embaralhando a tela do cliente. Defeito latente desde a fatia 2, revelado só quando passou a existir um caminho de escrita.
- **CA-3:** Os 4 slugs (`monthly`/`quarterly`/`semiannual`/`annual`) são **semeados** na primeira migração com preços de placeholder; admin ajusta antes do lançamento.
- **CA-4:** Plano não pode ser **excluído** — apenas desativado (`is_active=false`). Mantém integridade com assinaturas históricas.
- **CA-5:** Entrada em `audit_log` ao editar plano (com diff).

**Erros previstos:**
- **E-1:** Acesso de não-admin a `PATCH` → HTTP 403.
- **E-2:** `card_price < pix_price` → HTTP 422.

---

## SUB-RF-002 — Iniciar trial automático ao verificar e-mail

**Prioridade:** Essencial (MVP).
**Ator:** Sistema.
**Pré-condições:** AUTH-RF-002 conclui (e-mail verificado).

**Descrição:**
No momento em que o cliente verifica o e-mail, sistema cria uma `subscription` do tipo `trial` com 7 dias de duração. Único por usuário.

**Critérios de aceitação:**
- **CA-1:** Após AUTH-RF-002 CA-4 (e-mail verificado), gatilho cria `subscriptions` com `source=trial`, `start_at=now()`, `end_at=now() + 7 dias`, `status=active`. Marca `users.trial_used_at=now()`.
- **CA-2:** Cliente que recadastra com **novo e-mail** ganha novo trial (nova conta, novo `trial_used_at`). **Mesma conta** não ganha dois trials.
- **CA-3:** Entrada em `audit_log` opcional (volume alto).
- **CA-4:** Trial é cancelado se a conta for **excluída** antes do término — `subscriptions.status=revoked`, junto da anonimização LGPD (PROF-RF-009). **Desativação não revoga** (ajuste feito na implementação, 2026-08-26): desativar é reversível (ADR-0015) e quem reativa a conta volta com o acesso que tinha — inclusive assinatura paga, que seria perdida sem essa distinção. Conta desativada não pratica quiz de qualquer forma.
- **CA-5** (aditivo de implementação, 2026-08-26): contas criadas **antes** do gatilho existir receberam o trial por migration de dados retroativa (`0020_retroactive_trial.sql`), contado a partir da execução da migration. Sem isso, ligar QUIZ-RF-009 barraria de surpresa toda a base existente. A migration é idempotente (`trial_used_at` é a trava) e ignora contas `deleted`.

---

## SUB-RF-003 — Iniciar checkout (cliente escolhe plano, método e cupom opcional)

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.
**Pré-condições:** Sessão ativa.

**Descrição:**
Cliente escolhe plano e método de pagamento (opcionalmente aplica cupom). Sistema cria registro de `payments` em estado `pending` e devolve dados/QR code do Mercado Pago para conclusão.

**Critérios de aceitação:**
- **CA-1:** `POST /me/checkout` com `{ plan_slug, method: "pix" | "mp_balance" | "card", installments?: 1..3, coupon_code?: string }`.
- **CA-2:** Validações:
  - `plan_slug` ∈ planos `is_active=true`.
  - `method=card` aceita `installments ∈ 1..max_installments`; demais métodos ignoram esse campo (1 implícito).
  - `method != card` → `gross_amount = pix_price`; `method=card` → `gross_amount = card_price`.
- **CA-3:** **Cupom (opcional):** se `coupon_code` informado, sistema valida via SUB-RF-014 CA-3. Cupom inválido **não bloqueia** o checkout — retorna `coupon_applied=false` no payload; cupom válido reduz `net_amount` e incrementa `used_count`.
- **CA-4:** Sistema cria registro em `payments` com `status=pending`, `gross_amount`, `discount_amount`, `net_amount`, `coupon_id?`. Gera `mp_payment_id` chamando a API do Mercado Pago **com o `net_amount`**. Resposta HTTP 201, discriminada por `method`:
  - Para `pix` / `mp_balance`: `{ method: "pix", payment_id, qr_code_base64, qr_code_text, expires_at, gross_amount, discount_amount, net_amount, coupon_applied }`.
  - Para `card` — **contrato revisado em 2026-08-26 (ADR-0040)**: o request leva `card_token`, `payment_method_id`, `device_id` e `installments`, gerados no navegador pelo MercadoPago.js V2 / Card Payment Brick; a resposta é `{ method: "card", payment_id, order_status, payment_status, status_detail, installments, gross_amount, discount_amount, net_amount, coupon_applied }`. **Não existe `checkout_url`** — a especificação original descrevia Checkout Pro (fluxo hospedado com redirect), descartado em favor do Checkout Transparente sobre a mesma API de Orders do PIX. O princípio do CA-4 original é preservado: nenhum dado do cartão passa pelo backend, só o token de uso único.
  - **A assinatura NÃO nasce no checkout**, nem no cartão (que é síncrono): o único caminho que cria assinatura continua sendo o webhook (SUB-RF-004), já idempotente e validado em produção. O cliente confirma o acesso por `GET /me/subscription`, como no PIX.
  - `mp_balance` continua sem implementação (retorna 422 `method_not_supported_yet`) — é fluxo hospedado, e reabriria a complexidade de conta de teste "compradora" que a ADR-0040 evita.
- **CA-5:** Cliente que tem **assinatura `paid` ativa** pode iniciar novo checkout — o pagamento futuro **estende** a assinatura quando confirmado (CA aplicada em SUB-RF-004).
- **CA-6:** Rate limit: 5 checkouts pendentes simultâneos por cliente; o 6º falha com instrução para concluir os anteriores ou esperar expiração.

**Erros previstos:**
- **E-1:** Plano inativo → HTTP 422.
- **E-2:** Falha de comunicação com Mercado Pago → HTTP 502; `payments.status=failed` registrado.
- **E-3:** Rate limit excedido → HTTP 429.
- **E-4** (cartão, 2026-08-26): `card_token`/`payment_method_id` ausentes → HTTP 422 `card_token_required` (indica falha de integração do frontend, não erro do cliente).
- **E-5** (cartão, 2026-08-26): `installments` fora de `1..max_installments` → HTTP 422 `invalid_installments`, com `details.max_installments`.
- **Nota (cartão):** uma recusa do emissor **não** é erro HTTP — o checkout responde 201 com `order_status`/`status_detail` descrevendo a recusa, e o pagamento vira `failed` quando o webhook chega. Erro HTTP fica reservado a falha de comunicação (E-2).

> **Nota de ambiente de teste (2026-08-26):** no sandbox desta conta, o cartão de teste **Mastercard** (5031 4332 1540 6351) é recusado com `invalid_transaction_amount` para qualquer valor; o **Visa** (4235 6477 2802 5682) funciona normalmente. A mensagem é enganosa — a frase completa é "outside the valid range for *available payment methods*", e sem método disponível nenhuma faixa contém o valor. Não confundir com piso de valor real do método.

---

## SUB-RF-004 — Webhook do Mercado Pago: confirmação de pagamento

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (recebe POST do MP).

**Descrição:**
Endpoint que recebe notificações do Mercado Pago e atualiza estado interno de `payments` e `subscriptions`. Toda confirmação de pagamento sólido (`approved`) gera/estende assinatura.

**Critérios de aceitação:**
- **CA-1:** `POST /webhooks/mercado-pago` — endpoint público; **valida assinatura HMAC** do header `x-signature` contra segredo `MP_WEBHOOK_SECRET`. O manifest do HMAC usa o `data.id` da **query string** da notificação (`?data.id=...&type=...`) — é sobre esse valor que o MP assina (doc oficial valida com `req.query['data.id']`), nunca sobre o `data.id` do corpo. Falha → HTTP 401, ignora.
- **CA-8:** **Contrato de entrega do MP: autenticou, responde 200.** Notificação com assinatura válida cujo pagamento não existe localmente (ex.: simulador do painel, que envia `data.id` fictício) ou cujo tópico não é rastreado (`type` ≠ `order`) → HTTP 200 com `ignored` no corpo e log para investigação — **nunca 404/erro**: o MP trata qualquer status ≠ 200/201 como falha de entrega e reentrega a cada 15 minutos, marcando o endpoint como falho no painel.
- **CA-2:** Para evento `payment.updated` com `status=approved`: localiza `payments` por `mp_payment_id`, marca como `paid`, `paid_at=now()`. Cria/estende `subscriptions`:
  - Se cliente **não tem** assinatura ativa: `start_at=now()`, `end_at=now() + plan.duration_days`.
  - Se **tem** assinatura `paid` ativa: `start_at=now()`, `end_at=current_subscription.end_at + plan.duration_days` (renovação antecipada **acumula** dias). Cortesias vigentes também acumulam (regra ADR-0006).
  - `status=active`, `source=paid`, `plan_id=<plano>`, `payment_id=<pagamento>`.
- **CA-3:** Para evento `payment.updated` com `status=rejected`/`cancelled`: `payments.status=failed`, `failure_reason=<motivo do MP>`. Não cria assinatura.
- **CA-4:** Para evento `payment.updated` com `status=refunded`: `payments.status=refunded`, `refunded_at=now()`; localiza a `subscription` correspondente e a revoga (`status=revoked`, `end_at=now()`) **se ainda não consumida totalmente**. Auditoria registra.
- **CA-5:** Webhook é **idempotente** — receber duas vezes o mesmo evento não duplica assinatura nem registra duas vezes. Implementado via `mp_payment_id` + verificação de estado atual.
- **CA-6:** Entrada em `audit_log` com `action=mp_webhook_processed`, `payload_summary`.
- **CA-7:** Em sucesso de pagamento, sistema **envia e-mail ao cliente** confirmando pagamento, plano, novo `end_at` e link para o comprovante MP (`mp_receipt_url`).

**Erros previstos:**
- **E-1:** Assinatura HMAC inválida → HTTP 401, log de diagnóstico (sem vazar segredo). O log traz o **manifest exato**, o HMAC esperado por **fingerprint** de cada segredo configurado (`sha256` truncado — identifica o segredo sem revelá-lo) e `live_mode`/`application_id`/`user_id` do corpo. Causas, em ordem de frequência observada:
  1. **Valor de `MP_WEBHOOK_SECRET` divergente do painel** — copiado truncado/com espaço, ou regenerado no painel depois da cópia. Sintoma: fingerprint do log ≠ fingerprint do segredo do painel.
  2. **Processo com `.env` obsoleto** — `bun --watch` recarrega módulos importados, **não** o `.env`; trocar o segredo exige reiniciar `bun run dev`. Sintoma: fingerprint do log ≠ o esperado embora o arquivo já esteja correto.
  3. **Notificação emitida pela aplicação do USUÁRIO DE TESTE** (causa confirmada dos 401 em dev, 2026-08-18). O `application_id` do log identifica quem emitiu a notificação. Nesta conta: a aplicação BomberQuiz é **`1098861728853745`** (User ID `77654682`), mas as orders criadas com o Access Token de *Credenciais de teste* saem como **`4997377397405635`** (User ID `3422204690`) — que é a aplicação existente **dentro da conta do usuário de teste** (`TESTUSER…`, listado no painel em *Credenciais de teste → Dados das credenciais de teste*), não a BomberQuiz em modo teste. Essas notificações são assinadas com o segredo **daquela** aplicação; nenhum segredo da tela de Webhooks da BomberQuiz as valida. O simulador do painel, assinado pela própria BomberQuiz, responde 200 e **não prova nada** sobre o tráfego real.
  4. Segredo do modo errado do painel (teste × produtivo), **quando** a aplicação tiver segredos distintos por modo — nem toda conta tem.

  Diagnóstico: `bun run mp:webhook-replay --log <arquivo>` (ou `--file <jsonl>`) recalcula o HMAC das notificações já recebidas contra os segredos candidatos (`--secret` aceita um valor avulso do painel, sem tocar no `.env`) e diz se o defeito é do manifest ou do segredo. `MP_WEBHOOK_SECRET` aceita **vários segredos separados por vírgula**, para rotação sem janela de 401.
- **E-2:** `payments` não encontrado → HTTP 200 com `ignored: "unknown_payment"` e log para investigação (ver CA-8 — respondia 404 até 2026-08-18, o que violava o contrato de entrega do MP e fazia o simulador do painel reportar falha).

---

## SUB-RF-005 — Visualizar status da própria assinatura

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Tela "Minha assinatura" do cliente com status atual e próximos passos.

**Critérios de aceitação:**
- **CA-1:** `GET /me/subscription` retorna:
  ```json
  {
    "access_status": "active",
    "active_until": "2026-06-04",
    "source": "trial" | "paid" | "courtesy",
    "current_subscription": {
      "id": "...", "plan_name": "Mensal", "start_at": "...", "end_at": "...",
      "remaining_days": 4
    },
    "trial_used_at": "2026-05-28",
    "pending_payments": [
      { "id": "...", "plan_name": "Trimestral", "method": "pix", "amount": 8000, "expires_at": "..." }
    ],
    "refund_eligible_payments": [
      { "id": "...", "plan_name": "Mensal", "paid_at": "...", "refund_deadline": "..." }
    ]
  }
  ```
- **CA-2:** Quando `access_status=inactive`, payload inclui `cta: "subscribe"` para a UI sugerir checkout.
- **CA-3:** Quando há cortesia **e** pagamento simultâneos (acumulação ADR-0006), `current_subscription` reflete a com maior `end_at`; `source` mostra a origem dessa.
- **CA-4:** `refund_eligible_payments` lista pagamentos dentro da janela de 7 dias do CDC (ver SUB-RF-014). **Implementado em 2026-08-26** (fatia 4), junto com o `cta` de CA-2. A janela é avaliada pela mesma regra de domínio que valida o reembolso (`Payment.isRefundableAt`) — o botão oferecido é exatamente o que a API aceita. Cada item traz `{ id, plan_name, paid_at, refund_deadline }`.

---

## SUB-RF-006 — Histórico de pagamentos do cliente

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Lista dos pagamentos passados (sucesso, falha, reembolso) do próprio cliente. Atende exigência fiscal/transparência.

**Critérios de aceitação:**
- **CA-1:** `GET /me/payments` retorna `{ id, plan_name, method, gross_amount, discount_amount, net_amount, status, paid_at?, refunded_at?, failure_reason?, mp_payment_id, mp_receipt_url? }`, paginado.
- **CA-2:** Filtros: `status`, intervalo de datas.
- **CA-3:** Não retorna dados do cartão (apenas últimos 4 dígitos se MP fornecer no webhook — opcional no MVP).
- **CA-4:** `mp_receipt_url` é o link público do comprovante hospedado pelo Mercado Pago. Frontend exibe botão "Ver comprovante". **Correção de implementação (2026-08-26), medida contra a API real:**
  - **Só PIX tem link.** O payload de uma order de **cartão** paga não traz `ticket_url` nem qualquer outro campo de URL/recibo — verificado varrendo o JSON íntegro. Em cartão o campo fica `null` e o botão simplesmente não aparece. Dar recibo a quem paga com cartão exigiria gerar um comprovante próprio, o que SUB-P-05 descartou de propósito.
  - **É gravado no checkout, não no webhook.** Os campos do meio de pagamento PIX (`ticket_url`, `qr_code`, `qr_code_base64`) só vêm preenchidos na resposta de **criação** da order; uma vez paga, desaparecem. Preencher "a partir do webhook quando `status=paid`", como este CA dizia, resultaria em `mp_receipt_url` sempre nulo.
- **CA-5** (aditivo de implementação, 2026-08-26): a listagem também devolve `installments`, `created_at`, `refund_deadline` e `refundable` — os dois últimos espelham SUB-RF-005 CA-4 para que a tela de histórico desenhe o botão de reembolso sem cruzar dois endpoints. O filtro `status` aceita vários valores separados por vírgula, mesma convenção de `GET /me/questions`.

---

## SUB-RF-007 — Lembretes de expiração

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (job agendado).

**Descrição:**
Job diário avisa o cliente por e-mail quando sua assinatura está próxima do fim, incentivando renovação antes de perder acesso.

**Critérios de aceitação:**
- **CA-1:** Job roda **diariamente às 09:00** (`America/Sao_Paulo`).
- **CA-2:** Para cada cliente com `access_status=active`, avalia `remaining_days`. Em **D-7**, **D-3** e **D-1**, envia e-mail "Sua assinatura vence em N dias — renove para não perder acesso". Inclui link de checkout.
- **CA-3:** Envia **no máximo um e-mail por marco** (D-7, D-3, D-1) por assinatura — `subscription_reminders` registra qual marco já foi disparado. **Precisão da implementação (2026-08-26):** o registro é por **usuário + marco + `end_at` alvo**, não por assinatura. Cada pagamento cria uma `subscription` nova (SUB-RF-004 CA-2), então chavear por assinatura mandaria e-mails contraditórios a quem acumulou dias em duas compras. O job avalia o acesso **consolidado** (maior `end_at` entre as ativas), que é o mesmo número que SUB-RF-011 usa.
- **CA-4:** Trial recebe os mesmos lembretes (D-7 não se aplica — trial tem 7 dias). Apenas D-3 e D-1 são relevantes para trial; D-7 só dispara para planos pagos.
- **CA-5:** Cliente que renova entre o marco D-7 e D-3 (estendendo a assinatura) **reseta** os marcos com base no novo `end_at`.
- **CA-6:** **No dia da expiração** (D-0), sistema envia e-mail final "Sua assinatura expirou. Renove para voltar a acessar." — uma única vez. Não há nova insistência depois disso (TAP é cíclico — usuários voltam quando precisarem).
- **CA-7:** Falha de envio (Resend down) fica em log; job é idempotente — re-executar no mesmo dia não duplica. **Implementação:** o marco é reservado (`INSERT … ON CONFLICT DO NOTHING`) **antes** do envio — uma queda entre reserva e envio custa um lembrete, o contrário custaria e-mail duplicado. Falha em um cliente não interrompe os demais.
- **CA-8** (aditivo de implementação, 2026-08-26): o mesmo job **fecha as assinaturas vencidas** (`status=active` e `end_at < now()` → `expired`), executando essa etapa **depois** dos lembretes — expirar antes tiraria da consulta justamente quem deve receber o D-0. Até esta fatia, nada no sistema fazia essa transição: `active` só era abandonado por revogação, e o acesso só estava correto porque SUB-RF-011 compara `end_at` na leitura.
- **CA-9** (aditivo de implementação, 2026-08-26): contas `inactive`/`deleted` são filtradas na própria consulta do job, conforme a regra geral de e-mails transacionais (PROF-RF-007 CA-4, catálogo em `arquitetura.md`).

---

## SUB-RF-008 — Conceder cortesia de assinatura (admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Sessão ativa com `role=admin`.

**Descrição:**
Admin concede assinatura gratuita a um usuário existente como **cortesia** — usado tanto para remunerar parceiros pelo trabalho de cadastro quanto para uso promocional/demonstração. Funcionalidade de primeira classe (ADR-0006).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/courtesies` com `{ beneficiary_user_id, days_granted, category, notes? }`. Validação Zod:
  - `beneficiary_user_id` aponta para usuário `active` (ou `inactive` — admin pode antecipar a reativação).
  - `days_granted ∈ 1..365`.
  - `category ∈ { "parceria", "demonstracao" }`.
  - `notes` opcional, até 500 caracteres.
- **CA-2:** Cria `courtesies` + `subscriptions` (`source=courtesy`, `courtesy_id=<id>`, `start_at=max(now(), current_end_at)`, `end_at=start_at + days_granted`, `status=active`). Acumula sobre assinatura ativa (paga ou cortesia) **estendendo** a partir do `end_at` atual.
- **CA-3:** **Limite de 10 cortesias por admin por mês civil** (regras gerais). Excedido → E-2 com mensagem clara.
- **CA-4:** Entrada em `audit_log` com `action=grant_courtesy`.
- **CA-5:** Sistema envia e-mail ao beneficiário ("Você recebeu N dias de assinatura como cortesia. Acesse agora.").
- **CA-6:** Cortesia aparece imediatamente em SUB-RF-005 (status da assinatura do cliente).

**Erros previstos:**
- **E-1:** Usuário-alvo `deleted` ou inexistente → HTTP 404.
- **E-2:** Admin excedeu limite mensal → HTTP 429 com `{ remaining_for_month: 0, resets_at: "..." }`.
- **E-3:** Validação Zod falha → HTTP 422.

---

## SUB-RF-009 — Listar cortesias concedidas (admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Histórico de cortesias para auditoria e relatório.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/courtesies` retorna `{ id, beneficiary_name, beneficiary_email, days_granted, category, notes, granted_by_admin_name, status, start_at, end_at, revoked_at? }`, paginado.
- **CA-2:** Filtros: `category`, `granted_by_admin_id`, `status` (`active`/`expired`/`revoked`), intervalo de `created_at`.
- **CA-3:** Por padrão **não exibe** cortesias cujo `revoked_at` é antigo; passar `?include_revoked=true` para auditoria.

---

## SUB-RF-010 — Revogar cortesia

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Cortesia existe com `status=active` (não totalmente consumida — `end_at > now()`).

**Descrição:**
Admin encerra antecipadamente uma cortesia ainda não totalmente consumida. Útil em correção de erro, término de parceria ou reavaliação periódica (admin observa se o parceiro segue atendendo as expectativas e revoga se necessário).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/courtesies/:id/revoke` com `{ reason }` (10–500 caracteres, obrigatório).
- **CA-2:** Em sucesso, marca `courtesies.revoked_at=now()`, `revoked_by_admin_id=<admin>`, `revocation_reason=<reason>`. A `subscriptions` correspondente vira `status=revoked`, `end_at=now()` (encerra imediatamente).
- **CA-3:** **Não afeta** assinatura paga vigente do mesmo cliente — se cliente tem `paid` + `courtesy`, a revogação da cortesia encerra só a parte da cortesia; o cliente continua com a assinatura paga.
- **CA-4:** Se a cortesia já estava totalmente consumida (`end_at < now()`), a revogação é inútil — retorna E-1.
- **CA-5:** Entrada em `audit_log` com `action=revoke_courtesy`.
- **CA-6:** Sistema envia e-mail ao beneficiário ("Sua cortesia de N dias foi encerrada. Motivo: X.").

**Erros previstos:**
- **E-1:** Cortesia já consumida ou já revogada → HTTP 409.
- **E-2:** Motivo ausente ou abaixo do mínimo → HTTP 422.

---

## SUB-RF-011 — Cálculo de `access_status`

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (consultado por QUIZ-RF-009 e PROF-RF-014).

**Descrição:**
Função/serviço que, dado um `user_id`, devolve o status de acesso consolidado considerando trial, assinaturas pagas e cortesias vigentes.

**Critérios de aceitação:**
- **CA-1:** Lógica: considera todas as `subscriptions` do usuário com `status=active`. `active_until = max(end_at)`. Se `active_until ≥ now()` → `access_status=active`; senão → `inactive`.
- **CA-2:** Retorna também `source` da subscription com maior `end_at` (`trial`/`paid`/`courtesy`).
- **CA-3:** Implementado como porta `AccessPolicy` em `application/` (arquitetura hexagonal, ADR-0011). Adapter em `infra/` lê do Drizzle.
- **CA-4:** Resultado pode ser **cacheado** em memória por curto período (≤ 60s) para reduzir carga em endpoints quentes — invalida ao receber webhook (SUB-RF-004) ou ao conceder/revogar cortesia (SUB-RF-008/010).

---

## SUB-RF-012 — Painel financeiro (admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Visão consolidada de receitas, cortesias e métricas-chave do negócio.

**Critérios de aceitação:**
- **CA-1:** `GET /admin/financial/overview` retorna:
  ```json
  {
    "period": { "from": "2026-05-01", "to": "2026-05-31" },
    "revenue": {
      "total_cents": 1500000,
      "by_plan": [ { "plan_slug": "monthly", "count": 12, "total_cents": 360000 } ],
      "by_method": [ { "method": "pix", "count": 25, "total_cents": 750000 } ],
      "discounts_applied_cents": 120000,
      "refunded_cents": 30000
    },
    "courtesies": {
      "total_days_granted": 360,
      "active_count": 8,
      "by_category": [ { "category": "parceria", "count": 6, "days": 270 } ]
    },
    "users": {
      "in_trial": 18,
      "active_paid": 42,
      "active_courtesy": 8,
      "lapsed_last_30d": 5
    },
    "mrr_proxy_cents": 1200000
  }
  ```
- **CA-2:** Parâmetros opcionais: `from`, `to` (datas ISO; default = mês corrente).
- **CA-3:** Receitas **não incluem** cortesias (regra firme ADR-0006). `discounts_applied_cents` mostra o total de cupons consumidos no período; `refunded_cents` mostra o que foi devolvido a clientes.
- **CA-4:** Endpoint **não detalha** transações individuais — só agregados. Lista detalhada vem de SUB-RF-009 (cortesias) e endpoint separado de pagamentos (futuro).
- **CA-5:** Acesso restrito a `admin`. Em equipe com >1 admin, todos veem o mesmo painel.

**Erros previstos:**
- **E-1:** Acesso de não-admin → HTTP 403.

---

## SUB-RF-013 — Gerenciar cupons de desconto (admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.

**Descrição:**
Admin cria, edita e desativa cupons de desconto aplicáveis no checkout. Ferramenta de marketing — particularmente útil para a fase inicial do app (boca-a-boca em um público geograficamente delimitado, CBMGO/Goiás).

**Critérios de aceitação:**
- **CA-1:** `POST /admin/coupons` com `{ code, discount_type, discount_value, valid_from?, valid_until?, max_uses?, applies_to_plan_slugs?: string[] }`.
- **CA-2:** Validações:
  - `code` ∈ 4–24 caracteres alfanuméricos (case-insensitive), único em todo o sistema.
  - `discount_type ∈ { "percent", "fixed_cents" }`.
  - Se `percent`: `discount_value ∈ 1..100`. Se `fixed_cents`: `discount_value > 0`.
  - `valid_from < valid_until` se ambos informados.
  - `applies_to_plan_slugs`: subconjunto de slugs ativos; null = aplica a todos.
- **CA-3:** Em sucesso, cupom nasce com `is_active=true`, `used_count=0`, `created_by_admin_id=<admin>`. HTTP 201 com payload. Entrada em `audit_log`.
- **CA-4:** `GET /admin/coupons` lista cupons (com filtros `is_active`, `valid_at` para validade vigente). Retorna `{ id, code, discount_type, discount_value, valid_from, valid_until, max_uses, used_count, is_active, applies_to_plan_slugs, created_at }`.
- **CA-5:** `PATCH /admin/coupons/:id` permite editar **apenas** `valid_until`, `max_uses`, `is_active`. **Não permite** alterar `code`, `discount_type` ou `discount_value` — quem precisa de outro percentual cria novo cupom (integridade de pagamentos passados que referenciam o cupom original).
- **CA-6:** Cupom não pode ser **excluído** — apenas desativado (`is_active=false`). Histórico de uso fica preservado.
- **CA-7:** Aplicação do cupom no checkout é avaliada em SUB-RF-003 CA-3, com regras detalhadas em SUB-RF-014.

**Erros previstos:**
- **E-1:** Acesso de não-admin → HTTP 403.
- **E-2:** Código duplicado → HTTP 409.
- **E-3:** Validação Zod falha → HTTP 422.

---

## SUB-RF-014 — Solicitar reembolso (cliente, dentro de 7 dias)

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.
**Pré-condições:** Pagamento próprio com `status=paid` e dentro da janela de 7 dias do `paid_at`.

**Descrição:**
Atende ao **direito de arrependimento** previsto no CDC (art. 49) — contratos firmados fora de estabelecimento comercial podem ser desfeitos em até 7 dias sem necessidade de justificativa. Fluxo automatizado para o cliente; fora da janela, suporte manual.

**Critérios de aceitação:**
- **CA-1:** `POST /me/payments/:payment_id/refund` com payload opcional `{ reason? }` (texto livre, até 500 caracteres — opcional porque CDC não exige justificativa).
- **CA-2:** Validações:
  - Pagamento pertence ao cliente autenticado; senão → E-1.
  - `payments.status = paid`; senão → E-2.
  - `now() ≤ paid_at + 7 dias`; senão → E-3 ("Janela de 7 dias expirada — entre em contato com o suporte").
  - Pagamento ainda não foi reembolsado; senão → E-2.
- **CA-3:** Em sucesso, sistema chama a API do Mercado Pago para processar o reembolso **total**, marca `payments.status=refunded`, `refunded_at=now()` e **revoga a `subscription` correspondente na mesma transação**. Caso a API do MP falhe, o pagamento fica `paid` (nada muda localmente) e retorna HTTP 502 com mensagem orientando contato com suporte.
  **Duas correções de implementação (2026-08-26), ambas medidas contra a API real:**
  - **Endpoint.** Não é `POST /v1/payments/:id/refunds` (API legada): usamos a API de **Orders**, e o id `PAY…` que uma order carrega **não existe** na API legada — ela responde `404 resource not found`. O endpoint correto é **`POST /v1/orders/:id/refund`**, sem corpo (devolução total, que é o que o CDC exige). Responde 201 com a order já em `status: "refunded"`; um segundo pedido responde 409 `order_already_refunded`.
  - **Quem revoga a assinatura.** A versão anterior deste CA delegava a revogação ao "webhook posterior do MP". **Isso deixaria o cliente reembolsado e ainda com acesso:** a guarda de idempotência do webhook (SUB-RF-004 CA-5) só age sobre pagamento ainda `paid`, então a notificação chegaria depois de já termos gravado `refunded` e não faria nada. A revogação acontece aqui, junto da gravação. O caminho do webhook continua valendo para reembolso iniciado **fora do app** (painel do MP), onde o pagamento ainda está `paid` quando a notificação chega.
- **CA-4:** Em sucesso, sistema envia e-mail de confirmação ("Reembolso solicitado. O valor de R$X será creditado em até Y dias úteis na forma de pagamento original.").
- **CA-5:** Entrada em `audit_log` com `action=refund_requested`, `reason?`.
- **CA-6:** Aplicação do cupom (se houve): cupom **não é restituído** ao cliente — `coupons.used_count` permanece consumido (uma utilização efetiva já ocorreu). Política simples; reavaliar se virar gargalo de marketing.

**Erros previstos:**
- **E-1:** Pagamento não pertence ao cliente → HTTP 404.
- **E-2:** Pagamento não está `paid` ou já foi reembolsado → HTTP 409.
- **E-3:** Janela de 7 dias expirada → HTTP 409.
- **E-4:** Falha no Mercado Pago → HTTP 502.

---

## SUB-RF-015 — Aplicar cupom no checkout (regra de validação)

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (executado durante SUB-RF-003).

**Descrição:**
Regras de validação e cálculo do desconto quando um cliente informa `coupon_code` no checkout. Detalhamento técnico do CA-3 de SUB-RF-003.

**Critérios de aceitação:**
- **CA-1:** Validações do cupom (todas devem passar):
  - `coupons.is_active = true`.
  - `now() ∈ [valid_from, valid_until]` (ignora bordas null).
  - `used_count < max_uses` (ignora se `max_uses` null).
  - `applies_to_plan_slugs` inclui o `plan_slug` do checkout, ou é null.
- **CA-2:** Cliente pode usar o mesmo cupom **uma vez por cadastro** — duplicidade rastreada via `payments.coupon_id + payments.user_id` (paga ou pendente conta como usado). Tentativa de reuso → cupom inválido (mesmo retorno de qualquer falha de validação).
- **CA-3:** Cálculo do desconto:
  - `percent`: `discount_amount = round(gross_amount × discount_value / 100)`.
  - `fixed_cents`: `discount_amount = min(discount_value, gross_amount)` (não passa do total).
  - `net_amount = gross_amount - discount_amount`, com piso de **100 centavos** (R$1,00) — pagamentos abaixo desse piso são rejeitados pelo Mercado Pago.
- **CA-4:** Se qualquer validação falha, `coupon_applied=false` no payload de SUB-RF-003 CA-4, sem mensagem específica (não vaza se o cupom existe ou não, motivos diferentes têm a mesma resposta). UI mostra "Cupom inválido ou expirado".
- **CA-5:** Em sucesso, no momento de confirmação do pagamento (SUB-RF-004 CA-2), incrementa `coupons.used_count` em 1 (transação atômica com a confirmação do pagamento).
- **CA-6:** Se o pagamento falha (rejected/cancelled), o `used_count` **não é incrementado** (cupom volta a estar disponível).

---

## Pendências deste módulo — resolvidas em 2026-05-28

- ✅ **SUB-P-01 — Recorrência automática (cartão).** Decidido **descartar definitivamente do MVP** (não apenas adiar). Argumento de negócio: TAP é cíclico, quem passa só presta de novo daqui a 3+ anos; público é sazonal; recorrência automática não compensa a complexidade nem o atrito psicológico de cobrança contínua. Após expiração, envia-se **um único e-mail final** (SUB-RF-007 CA-6) e o sistema espera o cliente voltar quando precisar.
- ✅ **SUB-P-02 — Equivalência "questões cadastradas ↔ dias de cortesia" para parceiros.** Decidido: **sem regra automática**. Admin observa o trabalho do parceiro, concede cortesia por tempo limitado, e reavalia periodicamente — se o parceiro não atende as expectativas, admin revoga a cortesia (SUB-RF-010). Modelo flexível e ajustado ao porte da equipe.
- ✅ **SUB-P-03 — Política de reembolso.** Decidido: **reembolso integral em até 7 dias automatizado via UI** (CDC art. 49). Implementado em SUB-RF-014. Fora da janela, apenas suporte manual via contato.
- ✅ **SUB-P-04 — Cupom de desconto.** Decidido **incluir no MVP**. Justificativa: fase de aquisição inicial em público geograficamente delimitado (Goiás) precisa de ferramenta de marketing/boca-a-boca. Implementado em SUB-RF-013 (gestão) e SUB-RF-015 (aplicação).
- ✅ **SUB-P-05 — Comprovante de pagamento.** Decidido: **link para o comprovante oficial do Mercado Pago** via campo `mp_receipt_url` no histórico de pagamentos (SUB-RF-006 CA-4). Zero código adicional; comprovante oficial já atende aspecto fiscal.
- ✅ **SUB-P-06 — Categorias de cortesia.** Decidido: **2 categorias** — `parceria` (remunerar parceiro de cadastro) e `demonstracao` (uso promocional, marketing, divulgação). Categoria `outro` removida — admin escolhe a mais próxima das duas e usa `notes` para detalhe.
- ✅ **Renomeação "doação" → "cortesia".** Cliente preferiu "cortesia" — conotação mais comercial e neutra; "doação" pode soar como caridade ou ter implicações fiscais inadequadas. Aplicado em toda a documentação e schema (tabela `courtesies`, campo `source=courtesy`, etc.). ADR-0006 atualizado.

## Pendências residuais (resolução futura)

- (Nenhuma identificada neste módulo. Eventuais ajustes finos no fluxo de Mercado Pago surgirão na implementação.)
