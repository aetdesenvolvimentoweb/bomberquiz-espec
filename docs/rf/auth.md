# Módulo 1 — Autenticação e cadastro de usuário

> Identificador dos RFs deste módulo: `AUTH-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- O **canal de identidade** é o **e-mail**, único no sistema.
- Senhas são armazenadas com **argon2id** (preferido) ou bcrypt. Nunca em claro.
- Sessões via **cookie httpOnly + SameSite=Lax + Secure** (Better-Auth padrão).
- Mensagens de erro voltadas ao usuário **não vazam informação** sobre existência de conta (ex.: "credenciais inválidas" cobre tanto e-mail inexistente quanto senha errada).
- Todo formulário e todo endpoint valida entrada com **Zod** antes de chegar ao use case.

## Regras gerais (aplicam-se a vários RFs)

| Regra | Valor |
|---|---|
| Tamanho mínimo da senha | 10 caracteres |
| Complexidade da senha | ao menos 1 letra e 1 número; recusar senhas em lista de "piores 10.000" (via biblioteca ou serviço como HIBP API) |
| Tamanho máximo do nome | 120 caracteres |
| Formato do telefone | DDI 55 + DDD + número, total 13 dígitos (`55619XXXXXXXX`) — formato E.164 |
| Expiração do token de verificação de e-mail | 24 horas |
| Expiração do token de recuperação de senha | 1 hora |
| Duração da sessão | 7 dias (cookie persistente); refresh automático a cada uso |
| Rate limit de login | 5 tentativas falhas em 15 min por IP + e-mail; após exceder, lockout exponencial |
| Rate limit de reenvio de e-mail / recuperação de senha | 3 solicitações por hora por e-mail |

---

## AUTH-RF-001 — Cadastro de novo usuário

**Prioridade:** Essencial (MVP).
**Ator:** Visitante (sem sessão).
**Pré-condições:** Nenhuma.

**Descrição:**
Visitante preenche formulário de cadastro com seus dados pessoais, aceita os termos de uso e a política de privacidade (LGPD), e cria sua conta. A conta é criada com o papel **Cliente** e fica **pendente de verificação de e-mail** até AUTH-RF-002 ser concluído.

**Campos solicitados:**

| Campo | Obrigatório | Validação |
|---|---|---|
| Nome completo | Sim | 3–120 caracteres, ao menos um espaço (nome + sobrenome) |
| E-mail | Sim | Formato válido, único no sistema (case-insensitive) |
| Telefone (WhatsApp) | Sim | E.164 (`55…`), 13 dígitos |
| Data de nascimento | Sim | Data válida; idade ≥ 18 anos (idade mínima para ingresso no CBMGO) |
| Sexo | Sim | Uma das opções: `Masculino`, `Feminino`, `Prefere não informar` |
| Senha | Sim | Regras gerais acima |
| Confirmação de senha | Sim | Igual a "Senha" |
| Aceite LGPD | Sim | Checkbox; armazena `consent_version` |

**Critérios de aceitação:**
- **CA-1:** Quando todos os campos passam na validação, sistema cria usuário com `role=client`, `email_verified_at=null`, `consent_version=<versão atual>` e dispara e-mail de verificação (AUTH-RF-002).
- **CA-2:** Resposta de sucesso é HTTP 201 + payload `{ userId, message: "Verifique seu e-mail" }`. Frontend redireciona para tela de "Verifique seu e-mail".
- **CA-3:** O campo de avatar no banco é uma **URL** (`avatar_url`), nunca bytes da imagem. Sem avatar enviado, o campo fica `null` e o frontend renderiza um placeholder genérico (gerado client-side a partir das iniciais do nome, ou imagem estática). O upload propriamente dito vai para **Cloudinary** (free tier, transformações automáticas de crop/redimensionamento) — ver ADR-0013.
- **CA-4:** Senha é hasheada com argon2id antes de qualquer persistência. Texto puro **nunca** é logado.
- **CA-5:** O e-mail é normalizado para minúsculas antes da gravação e da checagem de unicidade.

**Erros previstos:**
- **E-1:** E-mail já cadastrado → HTTP 409, mensagem genérica "Não foi possível criar a conta. Verifique seus dados ou tente recuperar a senha."
- **E-2:** Qualquer validação Zod falha → HTTP 422 com lista de campos inválidos.
- **E-3:** Aceite LGPD não marcado → HTTP 422.

---

## AUTH-RF-002 — Verificação de e-mail

**Prioridade:** Essencial (MVP).
**Ator:** Usuário recém-cadastrado.
**Pré-condições:** AUTH-RF-001 concluído; e-mail de verificação enviado.

**Descrição:**
Após o cadastro, o sistema envia um e-mail contendo um link com token único. Ao clicar, o token é validado e o e-mail marcado como verificado.

**Critérios de aceitação:**
- **CA-1:** Token tem 32+ bytes de entropia (crypto-random), armazenado em hash no banco.
- **CA-2:** Token expira em 24h (regras gerais). Após expirado, retorna mensagem "Link expirado. Solicite um novo."
- **CA-3:** Token é de **uso único** — invalidado após o primeiro uso bem-sucedido.
- **CA-4:** Em sucesso, `email_verified_at` recebe o timestamp atual.
- **CA-5:** Em sucesso, o usuário é automaticamente logado e redirecionado para a tela inicial pós-login.
- **CA-6:** Se o usuário já estava verificado e clica no link novamente, retorna mensagem amigável "Seu e-mail já está verificado" e segue para login.

**Erros previstos:**
- **E-1:** Token inválido / inexistente → HTTP 400, mensagem "Link inválido ou expirado".
- **E-2:** Token expirado → idem E-1 (mensagem unificada para não vazar status).

---

## AUTH-RF-003 — Reenvio do e-mail de verificação

**Prioridade:** Essencial (MVP).
**Ator:** Visitante (não logado, antes de verificar) ou Usuário com e-mail pendente.
**Pré-condições:** Existe conta com `email_verified_at=null`.

**Descrição:**
Permite ao usuário solicitar o reenvio do e-mail de verificação.

**Critérios de aceitação:**
- **CA-1:** Solicitação invalida tokens de verificação anteriores ainda ativos do mesmo usuário.
- **CA-2:** Um novo token é gerado e enviado.
- **CA-3:** Resposta é **sempre** "Se houver uma conta pendente para esse e-mail, enviamos o link de verificação", independentemente de a conta existir, estar já verificada ou não existir — evita enumeração.
- **CA-4:** Rate limit aplicado (regras gerais: 3/h por e-mail).

---

## AUTH-RF-004 — Login com e-mail e senha

**Prioridade:** Essencial (MVP).
**Ator:** Usuário cadastrado.
**Pré-condições:** Conta existe e tem `email_verified_at` preenchido.

**Descrição:**
Usuário informa e-mail e senha, obtém sessão ativa.

**Critérios de aceitação:**
- **CA-1:** Se credenciais válidas e e-mail verificado, sistema cria sessão (cookie httpOnly) e responde HTTP 200 com payload do usuário (sem senha).
- **CA-2:** **Promoção automática a Admin (AUTH-RF-010)** é avaliada nesse momento — se aplicável, o `role` é atualizado antes do payload de resposta.
- **CA-3:** Login falho conta para o rate limit (regras gerais).
- **CA-4:** Em sucesso, último login (`last_login_at`) é registrado.

**Erros previstos:**
- **E-1:** Credenciais inválidas → HTTP 401, mensagem genérica "E-mail ou senha incorretos".
- **E-2:** E-mail não verificado → HTTP 403, mensagem específica "Verifique seu e-mail antes de entrar" com botão para reenviar (AUTH-RF-003).
- **E-3:** Rate limit excedido → HTTP 429, mensagem "Muitas tentativas. Tente novamente em X minutos."

---

## AUTH-RF-005 — Logout

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.

**Descrição:** Encerra a sessão atual.

**Critérios de aceitação:**
- **CA-1:** Cookie de sessão é invalidado no servidor e removido no cliente.
- **CA-2:** Logout afeta **apenas a sessão corrente**. Outras sessões do mesmo usuário (outro dispositivo) permanecem.
- **CA-3:** Endpoint é idempotente: chamar logout sem sessão ativa retorna 200 mesmo assim.

---

## AUTH-RF-006 — Solicitar recuperação de senha

**Prioridade:** Essencial (MVP).
**Ator:** Visitante.
**Pré-condições:** Nenhuma.

**Descrição:**
Usuário informa o e-mail e recebe um link de redefinição **por e-mail** (canal único no MVP). WhatsApp como canal alternativo foi descartado para o MVP — fica reservado para uso futuro em divulgação e marketing.

**Critérios de aceitação:**
- **CA-1:** Resposta é **sempre** "Se houver uma conta com esse e-mail, enviamos as instruções de recuperação", independentemente da existência da conta — evita enumeração.
- **CA-2:** Se a conta existe, um token de recuperação (32+ bytes, hash no banco) é criado com expiração de 1h (regras gerais).
- **CA-3:** Solicitação invalida tokens anteriores de recuperação do mesmo usuário.
- **CA-4:** Rate limit aplicado (regras gerais: 3/h por e-mail).
- **CA-5:** Conta com e-mail ainda não verificado **também pode** solicitar recuperação (caso do usuário que esqueceu antes mesmo de verificar).

---

## AUTH-RF-007 — Redefinir senha com token

**Prioridade:** Essencial (MVP).
**Ator:** Visitante com token válido.
**Pré-condições:** AUTH-RF-006 concluído; token ainda válido e não usado.

**Descrição:**
Usuário acessa link/código, informa nova senha (com confirmação) e a redefine.

**Critérios de aceitação:**
- **CA-1:** Nova senha valida contra as regras gerais.
- **CA-2:** Token é de **uso único** — invalidado após sucesso.
- **CA-3:** Em sucesso, **todas as sessões ativas** do usuário são invalidadas (proteção contra sequestro). Usuário é redirecionado para login.
- **CA-4:** Em sucesso, sistema dispara e-mail de notificação "Sua senha foi alterada" para o e-mail cadastrado, contendo timestamp e IP (alerta de segurança).
- **CA-5:** Se o token corresponde a uma conta com e-mail não verificado, a redefinição **também verifica o e-mail** (chegou ao endereço, é válido).

**Erros previstos:**
- **E-1:** Token inválido / expirado / já usado → HTTP 400, mensagem unificada "Link inválido ou expirado".

---

## AUTH-RF-008 — Sessão e expiração

**Prioridade:** Essencial (MVP).
**Ator:** Usuário logado.

**Descrição:**
Política de duração e renovação de sessão.

**Critérios de aceitação:**
- **CA-1:** Sessão dura 7 dias a partir da última atividade (sliding expiration).
- **CA-2:** Cada request autenticada atualiza o `last_seen_at` da sessão (no máximo 1× por minuto, para evitar write amplification).
- **CA-3:** Sessão inativa por >7 dias é purgada por job periódico.
- **CA-4:** Múltiplas sessões simultâneas do mesmo usuário (vários dispositivos) são permitidas.
- **CA-5:** Endpoint `GET /me` retorna o usuário corrente a partir da sessão; usado pelo frontend para hidratar estado após reload da PWA.

---

## AUTH-RF-009 — Rate limiting de autenticação

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (transversal).

**Descrição:**
Proteção contra força bruta e abuso nos endpoints sensíveis: login, cadastro, reenvio de verificação, recuperação de senha.

**Critérios de aceitação:**
- **CA-1:** Login: 5 falhas em 15 min por par (IP, e-mail) → lockout exponencial (1min, 5min, 30min, 2h, 24h).
- **CA-2:** Cadastro: 10 cadastros por hora por IP.
- **CA-3:** Reenvio de verificação / recuperação de senha: 3/hora por e-mail.
- **CA-4:** Resposta HTTP 429 com header `Retry-After`.
- **CA-5:** Eventos de rate limit ficam em log para análise.

---

## AUTH-RF-010 — Promoção automática a Administrador via whitelist

**Prioridade:** Essencial (MVP).
**Ator:** Sistema.
**Pré-condições:** ADR-0005; existe `config/admin-whitelist.ts` com lista de e-mails.

**Descrição:**
No login bem-sucedido (AUTH-RF-004), o sistema verifica se o e-mail do usuário consta na whitelist. Se sim e o papel atual não for `admin`, promove imediatamente.

**Critérios de aceitação:**
- **CA-1:** Verificação ocorre em **toda** autenticação bem-sucedida (cobre o caso de adicionar um e-mail à whitelist após o usuário já existir).
- **CA-2:** Promoção registra entrada em `audit_log` (quem foi promovido, quando, via whitelist).
- **CA-3:** Remoção de um e-mail da whitelist **não rebaixa** automaticamente — exige ação manual de admin (ADR-0005). Isso evita perda acidental de admin por edit do arquivo.
- **CA-4:** Whitelist comporta no máximo 5 entradas; uma 6ª gera erro no boot do app (`config/env.ts` valida).
- **CA-5:** Comparação de e-mail é case-insensitive e ignora `+suffix` no local-part (`me+admin@x.com` casa com `me@x.com`).

---

## AUTH-RF-011 — Consentimento LGPD no cadastro

**Prioridade:** Essencial (MVP).
**Ator:** Visitante (durante AUTH-RF-001).

**Descrição:**
Aceite explícito da política de privacidade e dos termos de uso é obrigatório no cadastro. A versão aceita fica gravada na conta.

**Critérios de aceitação:**
- **CA-1:** Cadastro só é concluído se o checkbox de aceite estiver marcado.
- **CA-2:** `consent_version` é gravado na tabela `users` (ex.: `"2026-05-28"`).
- **CA-3:** Quando a versão dos termos mudar, o sistema apresenta tela de reaceite no próximo login. O acesso fica bloqueado até o reaceite.
- **CA-4:** O texto dos termos e da política fica em URLs públicas estáveis (`/termos`, `/privacidade`).
- **CA-5:** O usuário pode solicitar **exclusão da conta** (LGPD direito de eliminação) por uma rota dedicada — escopo detalhado em módulo futuro de "Perfil e papéis".

---

## Pendências deste módulo — resolvidas em 2026-05-28

- ✅ **AUTH-P-01 — Idade mínima para uso.** Definida em **18 anos** (idade mínima para ingresso no CBMGO).
- ✅ **AUTH-P-02 — Opções do campo "sexo".** Definidas: `Masculino`, `Feminino`, `Prefere não informar`.
- ✅ **AUTH-P-03 — Canal padrão de recuperação de senha.** Apenas **e-mail** no MVP. WhatsApp reservado para divulgação/marketing em fase futura.
- ✅ **AUTH-P-04 — Verificação de WhatsApp via OTP.** **Não** implementado no MVP. WhatsApp é apenas contato declarado, sem verificação.
- ✅ **AUTH-P-05 — Avatar.** Armazenado como **URL** no banco (`avatar_url`), upload via **Cloudinary** (ADR-0013). Frontend gera placeholder client-side a partir das iniciais do nome quando `avatar_url=null`.
