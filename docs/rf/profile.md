# Módulo 2 — Perfil e papéis

> Identificador dos RFs deste módulo: `PROF-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- Cobre o ciclo de vida da conta **após** o cadastro inicial (Módulo 1): self-service de dados, saída do sistema, sessões e gestão administrativa de papéis.
- Regras de validação de campos pessoais (nome, telefone, DOB, sexo) são **as mesmas de [`auth.md`](auth.md) AUTH-RF-001**; este módulo não as redefine.
- Toda ação sensível (troca de senha, troca de e-mail, exclusão de conta) **dispara e-mail de aviso** ao endereço cadastrado, no mesmo padrão de AUTH-RF-007 CA-4.
- Toda ação administrativa que altere papel ou status de outro usuário registra entrada em `audit_log` (mesmo padrão de AUTH-RF-010 CA-2).
- Admin **não edita** dados pessoais de outros usuários — só altera papel. Ver ADR sobre escopo administrativo no rodapé deste arquivo.
- **Política de sessão única:** máximo 1 sessão ativa por usuário (ver ADR-0014 e PROF-RF-009).

## Regras gerais (aplicam-se a vários RFs)

| Regra | Valor |
|---|---|
| Token de confirmação de troca de e-mail | 24 horas, uso único, hash no banco |
| Tamanho máximo de avatar | 2 MB |
| Tipos aceitos de avatar | `image/jpeg`, `image/png`, `image/webp` |
| Sufixo de e-mail anonimizado | `@deleted.local` (não roteável, apenas marcador) |
| Reautenticação para ações destrutivas | obrigatória nas últimas 5 minutos (re-informar senha) |
| Rate limit de troca de e-mail | 1 solicitação a cada 24h por usuário |
| Status possíveis de uma conta | `active`, `inactive` (desativada), `deleted` (anonimizada) |

---

## PROF-RF-001 — Visualizar próprio perfil

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado (qualquer papel).
**Pré-condições:** Sessão ativa.

**Descrição:**
Retorna ao usuário todos os dados do próprio cadastro para exibição em tela de perfil.

**Critérios de aceitação:**
- **CA-1:** Endpoint `GET /me/profile` (ou equivalente Better-Auth) devolve: `name`, `email`, `phone`, `dob`, `sex`, `avatar_url`, `role`, `status`, `consent_version`, `created_at`, `last_login_at`.
- **CA-2:** Resposta **nunca** inclui senha, hashes, tokens ou flags internas (`audit_*`, `deleted_at`).
- **CA-3:** Conta com `status=inactive` ou `deleted` **não acessa** este endpoint (já estaria sem sessão).

---

## PROF-RF-002 — Editar dados pessoais

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** Sessão ativa; conta `active`.

**Descrição:**
Permite ao usuário atualizar nome, telefone, data de nascimento e sexo. **Não** altera e-mail nem senha (fluxos próprios em PROF-RF-003 e PROF-RF-004).

**Critérios de aceitação:**
- **CA-1:** Validações idênticas às de AUTH-RF-001 (formato, idade ≥ 18, opções de sexo).
- **CA-2:** Edição é parcial — campos omitidos no payload permanecem inalterados.
- **CA-3:** Em sucesso, resposta retorna o perfil atualizado (mesmo payload de PROF-RF-001).

**Erros previstos:**
- **E-1:** Validação Zod falha → HTTP 422 com lista de campos inválidos.

---

## PROF-RF-003 — Trocar senha (logado)

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** Sessão ativa; conta `active`.

**Descrição:**
Usuário logado redefine a própria senha informando a senha atual + nova + confirmação.

**Critérios de aceitação:**
- **CA-1:** Senha atual é verificada antes de qualquer escrita. Falha → E-1.
- **CA-2:** Nova senha valida contra as regras gerais de [`auth.md`](auth.md) (≥10, complexidade, lista de "piores").
- **CA-3:** Nova senha não pode ser igual à atual.
- **CA-4:** Em sucesso, sistema envia e-mail de notificação "Sua senha foi alterada" com timestamp e IP (mesma mecânica de AUTH-RF-007 CA-4).
- **CA-5:** A sessão atual permanece válida. Como a política é de **sessão única** (PROF-RF-009), não há outras sessões a invalidar.

**Erros previstos:**
- **E-1:** Senha atual incorreta → HTTP 401, mensagem genérica "Senha atual incorreta".
- **E-2:** Nova senha igual à atual → HTTP 422, "A nova senha deve ser diferente da atual".
- **E-3:** Nova senha falha nas regras → HTTP 422.

---

## PROF-RF-004 — Trocar e-mail

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** Sessão ativa; conta `active`; e-mail atual já verificado.

**Descrição:**
Usuário solicita troca do e-mail de login. O novo e-mail precisa ser **confirmado por link** antes de substituir o antigo.

**Critérios de aceitação:**
- **CA-1:** Usuário informa novo e-mail. Sistema valida formato e unicidade (case-insensitive). Falha → E-1 / E-2.
- **CA-2:** Sistema gera token de confirmação (32+ bytes, hash, expiração de 24h) e envia link **para o novo e-mail**.
- **CA-3:** O e-mail antigo **continua valendo** para login até a confirmação. A troca pendente fica em campo separado (`pending_email`) com expiração junto com o token.
- **CA-4:** Ao clicar no link, `email` é substituído pelo `pending_email`, `email_verified_at` é atualizado, `pending_email` é limpo, token é invalidado.
- **CA-5:** Em sucesso, sistema envia e-mail de aviso **ao endereço antigo** ("Seu e-mail de acesso foi alterado para X. Se não foi você, recupere a conta agora.") — alerta de segurança.
- **CA-6:** Rate limit: 1 solicitação por usuário a cada 24h (regras gerais).

**Erros previstos:**
- **E-1:** Formato de e-mail inválido → HTTP 422.
- **E-2:** Novo e-mail já está em uso por outra conta → HTTP 409 com mensagem genérica "Não foi possível usar esse e-mail" (não confirma existência).
- **E-3:** Token inválido / expirado → HTTP 400, mensagem unificada "Link inválido ou expirado".
- **E-4:** Rate limit excedido → HTTP 429.

---

## PROF-RF-005 — Gerenciar avatar

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** Sessão ativa; conta `active`.

**Descrição:**
Usuário envia ou remove a própria imagem de avatar. Upload vai para **Cloudinary** (ADR-0013); o banco grava apenas a URL pública em `users.avatar_url`.

**Critérios de aceitação:**
- **CA-1:** Upload aceita apenas `image/jpeg`, `image/png`, `image/webp` (regras gerais); tamanho ≤ 2 MB. Backend valida MIME real (não confiar apenas no header).
- **CA-2:** Upload é feito via porta `AvatarStorage` (ADR-0013), implementada por adapter Cloudinary. Em sucesso, `users.avatar_url` recebe a URL devolvida pelo Cloudinary.
- **CA-3:** Remoção (`DELETE /me/avatar`) zera `avatar_url`. Frontend volta a renderizar placeholder gerado client-side a partir das iniciais.
- **CA-4:** Substituição (novo upload sobre avatar existente) tenta deletar a imagem antiga no Cloudinary (best-effort; falha não bloqueia o sucesso da operação — apenas log).

**Erros previstos:**
- **E-1:** Tipo de arquivo inválido → HTTP 422.
- **E-2:** Tamanho excedido → HTTP 413.
- **E-3:** Falha no Cloudinary → HTTP 502 com retry sugerido.

---

## PROF-RF-006 — Reaceite de termos quando a versão muda

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** AUTH-RF-011 (consentimento original gravado).

**Descrição:**
Implementa o gatilho prometido em AUTH-RF-011 CA-3. Ao detectar que `consent_version` do usuário é diferente da versão corrente, o sistema bloqueia o uso até que ele reaceite os novos termos.

**Critérios de aceitação:**
- **CA-1:** Em todo `GET /me`, o backend devolve um flag `requires_consent_renewal: boolean` quando a versão diverge.
- **CA-2:** O frontend, ao ver `requires_consent_renewal=true`, redireciona para tela de reaceite que mostra o resumo das mudanças e os links públicos para `/termos` e `/privacidade`. Toda outra navegação fica bloqueada.
- **CA-3:** Em aceite, sistema atualiza `consent_version` para a versão corrente e registra timestamp em `consent_accepted_at`.
- **CA-4:** Em recusa (ou logout sem aceitar), o usuário é deslogado. Na próxima tentativa de login, a tela de reaceite reaparece.
- **CA-5:** Acesso só é liberado após o reaceite.

---

## PROF-RF-007 — Desativar conta (reversível)

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** Sessão ativa; conta `active`.

**Descrição:**
Usuário pausa o acesso à própria conta. Dados são **preservados integralmente**. Diferente da exclusão LGPD (PROF-RF-008), é totalmente reversível.

**Critérios de aceitação:**
- **CA-1:** Operação exige reautenticação (informar senha novamente — regras gerais).
- **CA-2:** Em sucesso, `users.status` vira `inactive`, `deactivated_at` é registrado, a sessão ativa é encerrada.
- **CA-3:** Tentativa de login com conta `inactive` recebe resposta específica HTTP 403 `{ reason: "account_inactive" }` em vez do payload normal. Frontend exibe tela "Sua conta está desativada — deseja reativar?".
- **CA-4:** Conta `inactive` **não recebe** e-mails transacionais nem aparece em listagens administrativas comuns (admin pode filtrar por status para localizar, se necessário).
- **CA-5:** Assinaturas em curso (pagas ou doadas) **continuam consumindo o tempo** durante o período de desativação — sem pausa automática no MVP.

**Erros previstos:**
- **E-1:** Senha incorreta na reautenticação → HTTP 401.

---

## PROF-RF-008 — Reativar conta

**Prioridade:** Essencial (MVP).
**Ator:** Visitante (sem sessão) que tenta logar em conta `inactive`.
**Pré-condições:** Conta com `status=inactive`.

**Descrição:**
Fluxo de retorno ao sistema após uma desativação. Acessado a partir da resposta "account_inactive" do login (PROF-RF-007 CA-3).

**Critérios de aceitação:**
- **CA-1:** Frontend, ao ver `reason: "account_inactive"`, oferece botão "Reativar conta".
- **CA-2:** Reativação confirma a senha já informada no login e, em sucesso, atualiza `status` para `active`, limpa `deactivated_at`, emite a sessão normalmente.
- **CA-3:** Conta `deleted` **não** pode ser reativada — retorna mensagem genérica "Conta não encontrada" (não vaza estado).

**Erros previstos:**
- **E-1:** Conta `deleted` → HTTP 404, mensagem genérica.

---

## PROF-RF-009 — Excluir conta (LGPD — anonimização)

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.
**Pré-condições:** Sessão ativa; conta `active` ou `inactive`.

**Descrição:**
Atende o direito de eliminação previsto na LGPD (art. 18, VI). É **irreversível**. Anonimiza as PII do usuário **preservando integridade referencial** com questões cadastradas, estatísticas e doações (ADR-0015).

**Critérios de aceitação:**
- **CA-1:** Operação exige reautenticação (senha) **e** marcação explícita do checkbox "Entendo que esta ação é irreversível e anonimizará meus dados".
- **CA-2:** Em sucesso, em uma única transação:
  - `name` → `"Usuário excluído"`
  - `email` → `sha256(email_original) + "@deleted.local"` (preserva unicidade, não é roteável, não permite re-identificar)
  - `phone`, `dob`, `sex`, `avatar_url` → `null`
  - `password_hash` → `null` (impede qualquer login futuro)
  - `status` → `deleted`, `deleted_at` → timestamp atual
  - Sessão ativa é encerrada e qualquer token pendente (verificação, recuperação, troca de e-mail) é invalidado.
- **CA-3:** Avatar no Cloudinary é deletado (best-effort — falha não bloqueia o sucesso da anonimização).
- **CA-4:** **Questões cadastradas pelo usuário** (caso seja Parceiro/Admin) **permanecem** com `author_id` apontando para o registro anonimizado.
- **CA-5:** **Doações de assinatura recebidas ou concedidas** permanecem com referência para o registro anonimizado (para integridade do painel financeiro). Assinatura paga vigente é cancelada via gateway no mesmo fluxo (Módulo 6 detalha).
- **CA-6:** Sistema envia e-mail de confirmação **ao endereço original** ("Sua conta foi excluída. Os dados pessoais foram removidos conforme a LGPD.") **antes** da anonimização ser commitada, para garantir entrega ao destinatário real.
- **CA-7:** Operação não pode ser desfeita. Tentativa de login com o e-mail original retorna mensagem genérica "Conta não encontrada".

**Erros previstos:**
- **E-1:** Reautenticação falha → HTTP 401.
- **E-2:** Checkbox não marcado → HTTP 422.

---

## PROF-RF-010 — Política de sessão única

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (transversal, observado em login e em cada request autenticada).
**Pré-condições:** ADR-0014.

**Descrição:**
Cada usuário pode ter **no máximo 1 sessão ativa** simultânea. Novo login bem-sucedido encerra automaticamente toda sessão anterior do mesmo `user_id`. O usuário do dispositivo anterior é avisado, na próxima requisição autenticada, de que foi desconectado.

**Critérios de aceitação:**
- **CA-1:** Em todo login bem-sucedido (AUTH-RF-004), antes de emitir o novo cookie, todas as sessões existentes do mesmo `user_id` são invalidadas no servidor.
- **CA-2:** Quando uma sessão é invalidada por substituição (não por logout do próprio usuário), o motivo é gravado: `revocation_reason = "session_replaced"`.
- **CA-3:** Requisição autenticada com sessão invalidada por substituição recebe HTTP 401 com payload `{ reason: "session_replaced" }`. Frontend exibe modal/tela "Você foi desconectado porque entrou em outro dispositivo" e força fluxo de login.
- **CA-4:** Como há, por construção, no máximo 1 sessão, não é necessário UI "minhas sessões" no MVP.
- **CA-5:** Esta regra **substitui** AUTH-RF-008 CA-4 (que originalmente permitia múltiplas sessões).

---

## PROF-RF-011 — Buscar/listar usuários (admin)

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Sessão ativa com `role=admin`.

**Descrição:**
Admin localiza usuários para realizar ações administrativas (promover a parceiro, revogar papel, doar assinatura).

**Critérios de aceitação:**
- **CA-1:** Endpoint `GET /admin/users` com filtros: `email` (match exato, case-insensitive), `name` (prefixo, case-insensitive), `role` (`client`/`partner`/`admin`), `status` (`active`/`inactive`/`deleted`).
- **CA-2:** Resposta paginada (page size padrão 20, máximo 100). Campos retornados: `id`, `name`, `email`, `role`, `status`, `created_at`, `last_login_at`. **Sem** telefone, DOB, sexo (admin não precisa para ações de papel/assinatura).
- **CA-3:** Acesso restrito ao papel `admin`. Middleware RBAC consolidado virá no Módulo 3.
- **CA-4:** Contas `deleted` aparecem na busca apenas se o admin filtrar explicitamente por `status=deleted` (para fins de auditoria).

**Erros previstos:**
- **E-1:** Acesso de não-admin → HTTP 403.

---

## PROF-RF-012 — Promover Cliente a Parceiro

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Sessão ativa com `role=admin`; usuário-alvo com `role=client` e `status=active`.

**Descrição:**
Admin promove um Cliente a Parceiro, concedendo acesso à tela de cadastro de perguntas (escopo detalhado no Módulo 4).

**Critérios de aceitação:**
- **CA-1:** Admin localiza usuário via PROF-RF-011 e dispara a promoção (`PATCH /admin/users/:id/role`).
- **CA-2:** Em sucesso, `role` vira `partner`. Entrada no `audit_log`: `{ actor_id, target_id, action: "promote_partner", at }`.
- **CA-3:** Promoção **não** dispara doação de assinatura automaticamente — é ato separado do admin (Módulo 6).
- **CA-4:** Tentar promover um usuário que já é `partner` ou `admin` → E-1.
- **CA-5:** Tentar promover conta `inactive` ou `deleted` → E-2.

**Erros previstos:**
- **E-1:** Papel-alvo inválido (já parceiro/admin) → HTTP 409.
- **E-2:** Conta não-ativa → HTTP 409.

---

## PROF-RF-013 — Revogar papel de Parceiro

**Prioridade:** Essencial (MVP).
**Ator:** Administrador.
**Pré-condições:** Sessão ativa com `role=admin`; usuário-alvo com `role=partner`.

**Descrição:**
Admin reverte um Parceiro para Cliente (perda do direito de cadastrar perguntas).

**Critérios de aceitação:**
- **CA-1:** `PATCH /admin/users/:id/role` com `role=client`.
- **CA-2:** Em sucesso, `role` vira `client`. Entrada no `audit_log`: `{ actor_id, target_id, action: "revoke_partner", at, reason? }`. Campo `reason` é opcional mas recomendado.
- **CA-3:** Questões cadastradas pelo ex-parceiro **permanecem** no sistema (mesmo princípio de preservação da exclusão LGPD).
- **CA-4:** Assinatura doada já concedida ao parceiro **não é** automaticamente revogada — admin pode revogar/encerrar separadamente se quiser (Módulo 6).
- **CA-5:** Tentar revogar quem **não é** parceiro → E-1. Admin **não pode** revogar a si mesmo nem outro admin por essa rota (revogação de admin é via whitelist, ADR-0005).

**Erros previstos:**
- **E-1:** Papel atual não é `partner` → HTTP 409.

---

## Pendências deste módulo

- **PROF-P-01 — Retenção de conta desativada.** Conta `inactive` deve ser auto-excluída após N dias inativos? Sugestão para o MVP: **não** auto-excluir; manter indefinidamente até o usuário reativar ou solicitar exclusão.
- **PROF-P-02 — Detalhamento da listagem admin.** Admin deve ver, no detalhe de um usuário, o histórico de assinaturas e doações? Decidir junto ao Módulo 6.
- **PROF-P-03 — Cooldown na troca de e-mail.** Além do rate limit de 1 por 24h, deve haver cooldown extra logo após uma troca bem-sucedida (proteção anti-takeover de conta comprometida)?
- **PROF-P-04 — Hash determinístico do e-mail anonimizado.** Usar SHA-256 puro ou com sal global do app? Sal global impede correlacionar dois "Usuários excluídos" entre instâncias do app, mas exige proteção do sal. Decidir no detalhamento técnico.
