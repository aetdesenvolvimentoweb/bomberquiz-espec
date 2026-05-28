# Especificação de requisitos — BomberQuiz

> Documento vivo. Atualize conforme as decisões forem tomadas.

## Visão geral

BomberQuiz é um sistema de perguntas e respostas para auxiliar no estudo e fixação dos conteúdos cobrados no **Teste de Avaliação Profissional (TAP) do Corpo de Bombeiros Militar do Estado de Goiás (CBMGO)**. O usuário realiza testes curtos, **com ou sem cronômetro**, e acompanha sua evolução por matéria para concentrar esforços onde tem mais dificuldade.

O sistema tem duas frentes:
1. **Cadastro de questões** (administradores e parceiros).
2. **Realização de testes** (clientes / usuários assinantes).

## Modelo de negócio

- **Período gratuito** de acesso (duração a definir).
- Após o período, acesso por **assinatura paga** com planos:
  - Mensal
  - Trimestral
  - Semestral
  - Anual
- **Assinaturas doadas:** administradores podem conceder assinatura gratuita a qualquer usuário por período arbitrário (ver "Doação de assinatura"). É o mecanismo usado para remunerar parceiros pelo trabalho de cadastro e também para uso promocional/marketing.
- **Parceria por contrapartida:** parceiros recebem assinatura doada em troca de cadastro de questões. _A definir: regras de equivalência (quantas questões = quanto tempo de assinatura), critérios de aprovação das questões cadastradas._

## Atores (papéis)

### Administrador
- Equipe **pequena** no início: **máximo 5 administradores**.
- Controle de acesso via **whitelist de e-mails** mantida em configuração (não autoatribuição). Quando um usuário whitelist faz login pela primeira vez, é promovido a administrador automaticamente. Ver ADR-0005.
- Cadastra perguntas e respostas (sem restrição de matéria).
- Cadastra **Eixos Temáticos** e **Matérias** (CRUD completo).
- **Doa assinaturas** a usuários (clientes ou parceiros) — período arbitrário, com motivo registrado. Ver "Doação de assinatura" abaixo.
- Gerencia parceiros (promove cliente a parceiro; revoga o papel).
- Acessa o painel administrativo-financeiro:
  - Estatísticas por pergunta (nível de dificuldade derivado de erros/acertos acumulados).
  - Quantidade de perguntas cadastradas por cada parceiro.
  - Assinaturas vigentes (por plano, por status), incluindo assinaturas doadas.
  - Parte financeira: receitas, fluxo de caixa.

### Cliente (Usuário)
- Faz cadastro próprio (ver "Cadastro e dados do usuário").
- Assina o aplicativo (ou usa no período gratuito, ou via assinatura doada).
- Responde questionários nas matérias do TAP.
- Acompanha a **evolução do seu desempenho por matéria**, identificando pontos fortes e fracos.
- _A definir: pode escolher matérias específicas para o teste? Pode definir tamanho do teste (nº de questões)? Cronômetro é opcional por teste ou configurável globalmente?_

### Parceiro
- Tem todos os direitos de um Cliente (responde testes, acompanha desempenho).
- **Adicionalmente** cadastra perguntas e respostas — acessa a tela de gerenciamento de perguntas.
- **Exclusão:** pode excluir **apenas as perguntas que ele próprio cadastrou**. Não pode excluir perguntas de outros parceiros ou de administradores.
- **Edição:** _a definir — provavelmente também restrita às próprias._
- Não é assinante pagante: recebe **assinatura doada** por administrador como contrapartida pelo trabalho de cadastro.
- _Questão em aberto:_ parceiro pode cadastrar em qualquer matéria, ou é vinculado a matérias específicas (especialidade)? Ver "Questões em aberto".

## Cadastro e dados do usuário

Todos os papéis (Administrador, Cliente, Parceiro) compartilham o mesmo cadastro base. O papel é uma **propriedade** do usuário, atribuída conforme regras descritas em "Atores".

### Dados solicitados no cadastro

| Campo | Obrigatório? | Observações |
|---|---|---|
| Nome completo | Sim | — |
| E-mail | Sim | **Único no sistema.** Canal principal de contato e identificador para login. Deve ser **verificado** após o cadastro (envio de link/código). |
| Telefone (WhatsApp) | Sim | Canal secundário de contato, recuperação de senha, notificações. _A definir: validar via OTP no WhatsApp?_ |
| Data de nascimento | Sim | Permite ações de fidelização (saudação de aniversário) e segmentação. _A definir: idade mínima?_ |
| Sexo | Sim | Mesma justificativa de fidelização/segmentação. _Considerar opção "prefere não informar" para conformidade._ |
| Senha | Sim | Armazenada com hash adequado (a definir). Regras mínimas de força a definir. |
| Avatar | **Não** | Opcional. Sem upload, usuário recebe **avatar genérico** (placeholder do sistema). |

### Considerações

- **LGPD:** o sistema coleta dados pessoais (nome, e-mail, telefone, DOB, sexo). Será necessário: política de privacidade, termo de consentimento no cadastro, mecanismo de exclusão/portabilidade de dados.
- **Recuperação de senha:** via e-mail e/ou WhatsApp (a decidir o canal padrão).
- **Verificação de identidade:** mínimo é verificar o e-mail no cadastro. Verificação de WhatsApp pode vir em etapa posterior, especialmente se for usada para recuperação de senha.

## Doação de assinatura

Funcionalidade administrativa de primeira classe (não um workaround). Permite a um administrador conceder assinatura gratuita a um usuário existente.

**Campos de uma doação:**
- Usuário beneficiário (busca por e-mail).
- Período concedido (em dias, ou data de início + data de fim).
- Motivo / categoria (ex: `parceria-cadastro-conteudo`, `marketing`, `cortesia`, `compensacao`). Campo obrigatório para auditoria e relatórios.
- Administrador que concedeu (registrado automaticamente).
- Data da concessão (automática).

**Regras:**
- Doações são acumuláveis: se o usuário já tem assinatura ativa (paga ou doada), o período da doação **estende** a data de expiração atual.
- Doações aparecem no painel financeiro **separadas** das receitas (não contam como receita; contam como custo de aquisição/parceria, conforme o motivo).
- Administrador pode **revogar** uma doação ainda não totalmente consumida? _A definir._

## Domínio: estrutura dos conteúdos do TAP

O conteúdo do TAP é organizado em três níveis hierárquicos. Esses três níveis são todos **dados cadastráveis**, não enumerações fixas em código, porque o edital muda a cada ano.

### Nível 1 — Eixo Temático
> Termo de trabalho. Alternativas consideradas: "Grupo Temático", "Área de Conhecimento", "Bloco". Decisão registrada em `docs/decisoes.md` (ADR-0003).

Agrupamento de matérias afins. Exemplos:
- **Salvamento**
- **Prevenção e Combate a Incêndio**
- _(outros eixos a serem cadastrados conforme edital vigente)_

### Nível 2 — Matéria
Cada matéria pertence a **um** eixo temático e geralmente corresponde a uma **fonte oficial** (manual operacional, norma técnica, lei, protocolo). Cada matéria tem um **peso/quantidade de questões cobradas no TAP** — isso é usado pelo sistema na hora de **montar um quiz que simule a distribuição real da prova**.

Exemplos:

| Eixo Temático | Matéria | Fonte oficial |
|---|---|---|
| Salvamento | Salvamento Terrestre | Manual Operacional de Salvamento Terrestre |
| Salvamento | Salvamento em Altura | Manual Operacional de Salvamento em Altura |
| Salvamento | Salvamento Aquático | Manual Operacional de Guarda-Vidas |
| Prevenção e Combate a Incêndio | Combate a Incêndio Urbano | Manual Operacional de Combate a Incêndio Urbano |
| Prevenção e Combate a Incêndio | Combate a Incêndio Florestal | Manual de Prevenção e Combate a Incêndios Florestais |
| Prevenção e Combate a Incêndio | Norma Técnica 01 — Procedimentos Administrativos | NT-01 CBMGO |
| Prevenção e Combate a Incêndio | Norma Técnica 11 — Saídas de Emergência | NT-11 CBMGO |

Outros eixos prováveis (a confirmar com o edital): **Suporte Básico de Vida / APH**, **Legislação**, **Atividades Técnicas / Vistorias**.

### Nível 3 — Pergunta e Respostas
Cada pergunta pertence a **uma** matéria. Estrutura mínima (a refinar nos RFs):
- Enunciado.
- Alternativas (provavelmente múltipla escolha — quantidade fixa? variável?).
- Indicação da(s) alternativa(s) correta(s).
- Metadados: autor (admin ou parceiro), data de cadastro, status (rascunho/aprovada/arquivada?), estatísticas acumuladas (erros/acertos).
- _A definir:_ comentário/justificativa da resposta? referência à página/artigo da fonte oficial? imagens?

## Cadastros do sistema

Decorrem da estrutura acima:

1. **Eixos Temáticos** — CRUD restrito a administradores.
2. **Matérias** — CRUD restrito a administradores. Cada matéria vincula-se a um eixo e armazena fonte + quantidade de questões esperada no TAP.
3. **Perguntas e Respostas** — cadastro por administradores e parceiros (regras de edição/exclusão e aprovação a definir, ver "Questões em aberto").

## Requisitos funcionais

Organizados em **6 módulos**, cada um em arquivo próprio sob `docs/rf/`. Identificadores no formato `<MODULO>-RF-NNN` (ex.: `AUTH-RF-001`). Cada RF traz: prioridade (Essencial / Desejável / Futuro), ator, pré-condições, descrição, critérios de aceitação (CA-N) e erros previstos (E-N).

| # | Módulo | Arquivo | Status |
|---|---|---|---|
| 1 | Autenticação e cadastro de usuário | [`rf/auth.md`](rf/auth.md) | ✅ Rascunho |
| 2 | Perfil e papéis | [`rf/profile.md`](rf/profile.md) | ✅ Rascunho |
| 3 | Conteúdo (admin) — eixos, matérias, perguntas | `rf/content-admin.md` | ⏳ Pendente |
| 4 | Conteúdo (parceiro) | `rf/content-partner.md` | ⏳ Pendente |
| 5 | Quiz (cliente) | `rf/quiz.md` | ⏳ Pendente |
| 6 | Assinaturas e doações | `rf/subscriptions.md` | ⏳ Pendente |

Pendências por módulo (decisões a tomar antes da implementação) ficam no rodapé de cada arquivo, identificadas como `<MODULO>-P-NN`.

## Requisitos não-funcionais

> Performance, usabilidade, plataforma alvo, offline-first, multi-usuário, etc.

- **Porte esperado:** mercado-teto de ~2.000 bombeiros ativos no CBMGO; base usual estimada em 300–500 usuários ativos (referência: último TAP teve ~300 concorrentes). Decisões de arquitetura priorizam **custo baixo** e **manutenibilidade**, não escala massiva.
- **Plataforma:** aplicação **web mobile-first PWA** — instalável, com cache offline básico de questões já carregadas. Sem app nativo no MVP.
- **Idioma:** pt-BR em toda a interface, e-mails e mensagens.
- **Segurança:** seguir OWASP Top 10. Hash de senha com argon2id ou bcrypt. CSP, rate limiting, validação de entrada (Zod) em todas as rotas.
- **LGPD:** consentimento explícito, política de privacidade acessível, mecanismo de exclusão/portabilidade de dados.
- **Acessibilidade:** componentes acessíveis (shadcn já é a11y-friendly); meta de WCAG AA.
- **Disponibilidade:** alvo 99% (não 99.9% — não justifica o custo no porte).
- **Performance:** Time-to-Interactive < 3s em 4G; First Contentful Paint < 1.5s.

## Stack técnica

> Decisões consolidadas em ADRs 0008–0012. Esta seção é a visão sintética.

### Arquitetura
- **Dois serviços, dois repositórios:** `bomberquiz-api` (backend REST) e `bomberquiz-web` (frontend PWA). Contrato entre eles via OpenAPI gerado pelo backend.
- **Backend em arquitetura hexagonal** (ports & adapters) — ver `docs/arquitetura.md`.

### Backend (`bomberquiz-api`)
- Runtime: **Bun**.
- Framework HTTP: **Hono**.
- ORM: **Drizzle**.
- Banco: **PostgreSQL** no **Neon** (free tier inicial, scale-to-zero, branching para testes de integração).
- Autenticação: **Better-Auth** (e-mail + senha, verificação de e-mail, recuperação).
- Validação: **Zod** (request/response schemas).
- Documentação: **OpenAPI via `@hono/zod-openapi`** + UI **Scalar**.
- Testes: **Vitest** (unit + integration) — integração contra Neon branch ou Postgres em container.
- Hospedagem: **Fly.io** (sempre-on no free tier inicial).

### Frontend (`bomberquiz-web`)
- Bundler/framework: **Vite + React + React Router**.
- UI: **Tailwind CSS + shadcn/ui**.
- Estado servidor: **TanStack Query**. Estado UI: **Zustand**.
- PWA: **`vite-plugin-pwa`** (service worker, manifest, install prompt).
- Testes: **Vitest** + **React Testing Library** (unit/integration); **Playwright** (E2E).
- Hospedagem: **Cloudflare Pages**.

### Integrações externas
- **Pagamentos:** Mercado Pago (PIX + cartão, assinaturas recorrentes).
- **WhatsApp:** WhatsApp Cloud API (Meta) ou Z-API — _provedor a confirmar_.
- **E-mail transacional:** Resend ou AWS SES — _a confirmar_.
- **Armazenamento de imagens de questões:** Cloudflare R2 (sem cobrança de egress).
- **Armazenamento de avatares:** Cloudinary (free tier, transformações automáticas) — ver ADR-0013.

### Princípios
- Clean Architecture / Hexagonal — domínio puro, infra plugável.
- KISS, DRY, YAGNI.
- TDD onde fizer sentido (domain e use cases primeiro).
- TSDoc nos contratos públicos das camadas.
- OWASP Top 10 como checklist obrigatório antes de cada release.

## Fluxos principais

- _A definir (esboço inicial):_
  - **Cadastro do usuário:** usuário preenche dados → aceita termos (LGPD) → recebe e-mail de verificação → confirma → entra como Cliente.
  - **Login de administrador:** primeiro login de e-mail presente na whitelist promove o usuário automaticamente a Administrador.
  - **Promover Cliente a Parceiro:** administrador localiza cliente → atribui papel de parceiro → opcionalmente define matérias permitidas (ver questão em aberto).
  - **Doar assinatura:** admin busca usuário → informa período e motivo → confirma → assinatura ativa/estendida.
  - **Montar e realizar quiz:** cliente escolhe o escopo (TAP completo simulado, eixo específico, matéria específica, ou mix livre) → define cronômetro on/off → sistema sorteia questões respeitando o peso/quantidade de cada matéria quando o escopo for "TAP completo" → cliente responde → vê resultado → estatísticas atualizadas por matéria e por eixo.
  - **Cadastrar Eixo Temático:** admin cria/edita/remove eixo.
  - **Cadastrar Matéria:** admin cria matéria vinculada a um eixo + fonte oficial + quantidade de questões esperada no TAP.
  - **Cadastrar pergunta:** parceiro/admin escolhe matéria → cria enunciado + alternativas + resposta correta + metadados.
  - **Assinar (pago):** cliente escolhe plano → paga via gateway → assinatura ativa.

## Restrições e premissas

- Público-alvo é brasileiro (CBMGO), portanto pt-BR em toda a interface.
- **LGPD aplicável** — sistema coleta dados pessoais identificáveis. Necessário política de privacidade, consentimento explícito no cadastro, e mecanismo de exclusão a pedido do titular.
- E-mail é identificador único de usuário (não há login por CPF, username, etc.).
- Whitelist de administradores fica em **configuração da aplicação**, não cadastrada via UI por outro admin (evita autoescalada de privilégios). Máximo de 5 entradas no início.
- _Demais restrições a definir._

## Questões em aberto

### Resolvidas (mantidas aqui para referência histórica até virarem RF)
- ✅ Política de **exclusão** de questões pelo parceiro → só as próprias.
- ✅ Múltiplos administradores → sim, máximo 5 via whitelist (ADR-0005).
- ✅ Mecânica para parceiro receber assinatura → doação de assinatura (funcionalidade administrativa).
- ✅ Tipo de aplicação → PWA web mobile-first (ADR-0010).
- ✅ Forma de armazenamento → Postgres no Neon, via Drizzle (ADR-0009).
- ✅ Gateway de pagamento → Mercado Pago (ADR-0012).
- ✅ Política de saída de conta → dois fluxos: desativar (reversível) e excluir (anonimização irreversível das PII, preservando FKs). ADR-0015 / PROF-RF-007..009.
- ✅ Política de sessões simultâneas → máximo 1 sessão ativa por usuário; novo login encerra a anterior. ADR-0014 / PROF-RF-010.
- ✅ Edição de dados de outros usuários pelo admin → não permitida (admin só altera papel). PROF-RF-002 / PROF-RF-012 / PROF-RF-013.

### Em aberto
- Como medir/definir o "nível" de uma pergunta a partir dos erros/acertos (fórmula, thresholds).
- Critério de aprovação de questões cadastradas por parceiros (workflow de revisão? auto-aprovação?).
- Política de **edição** de questões pelo parceiro (provavelmente só as próprias, ainda a confirmar).
- Janela de tempo para editar/excluir antes de a questão "travar" (após ter sido respondida por N usuários?).
- Duração do período gratuito.
- Equivalência "questões cadastradas ↔ tempo de assinatura" para parceiros (informa quanto o admin deve doar).
- **Filtro de matéria do parceiro:** cada parceiro é vinculado a matérias específicas (especialidade) ou pode cadastrar em qualquer matéria?
- **Hierarquia entre administradores:** todos têm os mesmos poderes ou há "super-admin" (ex.: só ele pode promover parceiros / revogar admin)?
- **Estrutura da pergunta:** quantidade fixa ou variável de alternativas? Única correta ou múltiplas corretas? Pergunta pode incluir imagem/diagrama (importante para conteúdo operacional de bombeiro)?
- **Comentário/justificativa da resposta:** mostrar após responder? obrigatório no cadastro?
- **Referência à fonte:** cada pergunta deve apontar para capítulo/artigo/página do manual/norma de origem?
- **Versionamento da pergunta:** quando o manual é atualizado, como tratar perguntas baseadas em versão antiga?
- **Tipos de quiz:** simulado completo do TAP (respeitando pesos), prático livre por matéria, "matérias em que pior performo" (sugerido pelo sistema), prova cronometrada igual ao TAP real?
- **Reset de estatísticas:** cliente pode zerar seu histórico? Após mudança significativa de uma pergunta, suas estatísticas zeram?
- **Cadastro do usuário — específicos:**
  - Idade mínima para uso (especialmente se for menor de 18).
  - Verificação obrigatória de WhatsApp via OTP, ou só de e-mail?
  - Política de "esqueci minha senha": canal padrão (e-mail ou WhatsApp).
  - Campo "sexo" — usar opções fixas (Masculino/Feminino/Prefere não informar) ou aberto?
- **Doação de assinatura:** admin pode revogar uma doação ainda não consumida? Existe limite por admin/por mês?
