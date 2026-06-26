# Módulo 5 — Quiz (cliente)

> Identificador dos RFs deste módulo: `QUIZ-RF-NNN`.
> Ver [`../requisitos.md`](../requisitos.md) para visão geral; [`../decisoes.md`](../decisoes.md) para ADRs; [`../arquitetura.md`](../arquitetura.md) para implementação.

## Convenções deste módulo

- Cobre tudo que o **Cliente** faz no app além da gestão de conta: montar quizzes, responder, ver resultado, acompanhar evolução. Parceiro e Admin **também** acessam o quiz (herdam o papel de cliente — ver ADR-0007).
- Quiz é **online-only** no MVP. PWA pode cachear assets e o app shell, mas a sessão de quiz exige comunicação com o backend a cada resposta para validar e atualizar estatísticas. Quiz totalmente offline fica como pendência.
- Quiz é **efêmero**: cliente que fecha o app no meio **perde** a sessão em andamento (sem pause/resume no MVP — ver pendências).
- Sistema **nunca envia ao cliente** o `correct_index` nem `explanation` de uma questão antes da resposta. Esses campos são retornados na resposta da submissão (QUIZ-RF-002 CA-3).
- Acesso aos endpoints deste módulo exige sessão ativa **e** assinatura vigente (paga ou doada) **ou** estar dentro do período gratuito. Bloqueio detalhado em QUIZ-RF-009; regras de assinatura no Módulo 6.
- Mesma cultura de Zod, audit log, mensagens genéricas e single-session dos módulos anteriores.

## Regras gerais

| Regra | Valor |
|---|---|
| Modos de quiz suportados no MVP | `tap_simulation`, `free_subject`, `free_axis` |
| Tamanho do quiz livre (`free_subject`/`free_axis`) | escolha do cliente entre **10, 20, 30 ou 50** questões |
| Tamanho do simulado TAP | soma dos `tap_weight` das matérias com `status=active` e `tap_weight > 0`. A prova real do TAP tem **50 questões**, então a soma dos pesos deve totalizar ~50. Teto defensivo de **60** questões (folga para variação do edital): se a soma exceder, o sistema bloqueia o início com aviso ao admin de revisar os `tap_weight` (provável erro de cadastro) |
| Cronômetro | opcional, **tempo total**. Padrão = 3 minutos × nº de questões. Ajustável pelo cliente entre 1–5 min/questão |
| Modo de exibição da justificativa | `after_each` ou `at_end` (escolha do cliente ao iniciar o quiz) |
| Mínimo de questões publicadas para iniciar quiz | 5 (modos livres); para simulado TAP, ao menos 1 questão em cada matéria com `tap_weight > 0` |
| Cooldown / não-repetição de questões | **sem** cooldown no MVP — sorteio uniforme entre `status=published` |
| Filtro por nível de dificuldade no sorteio | **não** disponível no MVP |
| Limite de quizzes simultâneos por cliente | 1 (qualquer iniciar enquanto há um `in_progress` encerra o anterior como `abandoned`) |
| Retenção de quizzes finalizados | sem auto-expiração — ficam no histórico do cliente indefinidamente |

## Estrutura de dados (resumo do domínio)

- **`quiz_sessions`** — sessão única do quiz: `id`, `user_id`, `mode`, `scope_id?` (matéria/eixo quando aplicável), `total_questions`, `timer_enabled`, `time_limit_seconds?`, `explanation_mode`, `started_at`, `finished_at?`, `status` (`in_progress`/`finished`/`expired`/`abandoned`), `correct_count`, `answered_count`.
- **`quiz_session_questions`** — ligação questão↔sessão na ordem sorteada: `id`, `quiz_session_id`, `question_id`, `position` (1..N), `submitted_index?`, `is_correct?`, `answered_at?`. **Snapshot** da pergunta (alternativas, gabarito, justificativa) é armazenado aqui para que edições futuras da pergunta não distorçam o histórico do cliente.
- **`user_subject_stats`** (agregado materializado) — `user_id`, `subject_id`, `total_answers`, `correct_answers`, `last_answered_at`, `stats_reset_at?`.
- Estatísticas por eixo são derivadas das de matéria (não materializadas separadamente).

---

## QUIZ-RF-001 — Iniciar um quiz

**Prioridade:** Essencial (MVP).
**Ator:** Cliente (qualquer papel com acesso ativo).
**Pré-condições:** Sessão ativa; acesso ao app vigente (assinatura ou período gratuito); cliente não tem outro quiz `in_progress` (se tiver, será encerrado como `abandoned` — CA-6).

**Descrição:**
Cliente define modo, escopo e preferências; sistema sorteia as questões e abre a sessão. A primeira questão pode ser entregue na mesma resposta ou via QUIZ-RF-002 — o frontend tem o array completo (sem gabaritos) para evitar latência entre questões.

**Critérios de aceitação:**
- **CA-1:** `POST /quizzes` com payload:
  ```json
  {
    "mode": "tap_simulation" | "free_subject" | "free_axis",
    "subject_id"?: "...",         // obrigatório se mode=free_subject
    "axis_id"?: "...",            // obrigatório se mode=free_axis
    "size"?: 10 | 20 | 30 | 50,   // obrigatório se mode=free_*; ignorado em tap_simulation
    "timer_enabled": boolean,
    "time_per_question_seconds"?: number,  // 60..300 (1-5 min); default 180; ignorado se timer_enabled=false
    "explanation_mode": "after_each" | "at_end"
  }
  ```
  UI: na tela de iniciar quiz, `timer_enabled` é um **toggle/switch principal** ("Cronômetro: ligado/desligado"). Quando ligado, expõe o slider de tempo por questão; quando desligado, oculta o slider. Garante que o usuário perceba claramente que cronômetro é opcional.
- **CA-2:** Validações:
  - `tap_simulation`: exige ao menos 1 questão `published` em cada matéria `active` com `tap_weight > 0`. Caso contrário → E-1 com lista de matérias sem questões. Se a soma dos `tap_weight` exceder o teto de 60 (regras gerais) → E-6 (provável erro de cadastro de peso).
  - `free_subject`: matéria deve estar `active` e ter ao menos 5 questões `published`. Senão → E-2.
  - `free_axis`: eixo deve estar `active` e ter ao menos 5 questões `published` somando todas as matérias. Senão → E-3.
  - `size` ∈ {10, 20, 30, 50} para os modos livres.
- **CA-3:** Sorteio:
  - `tap_simulation`: para cada matéria com `tap_weight > 0`, sorteia exatamente `tap_weight` questões. Total = soma dos pesos. Ordem das questões é aleatorizada após o sorteio (não agrupa por matéria).
  - `free_subject`: sorteia `size` questões da matéria escolhida.
  - `free_axis`: sorteia `size` questões distribuídas entre as matérias do eixo proporcionalmente aos `tap_weight` (ou uniforme se todas tiverem peso 0).
  - Em todos os modos, ordem das **alternativas** de cada questão é aleatorizada por sessão (4 permutações) para evitar memorização posicional.
- **CA-4:** Resposta HTTP 201 com:
  ```json
  {
    "quiz_id": "...",
    "started_at": "...",
    "time_limit_seconds": 360,
    "explanation_mode": "after_each",
    "total_questions": 20,
    "questions": [
      { "position": 1, "question_id": "...", "subject_name": "...", "statement": "...", "image_url": null,
        "alternatives": ["A...", "B...", "C...", "D..."] }
      // ... sem `correct_index` nem `explanation`
    ]
  }
  ```
- **CA-5:** Quiz é criado com `status=in_progress`, snapshot das questões/alternativas/gabarito/justificativa é gravado em `quiz_session_questions` no momento da criação (para imunidade a edições posteriores das perguntas).
- **CA-6:** Se já existe quiz `in_progress` para esse cliente, ele é finalizado como `status=abandoned` antes da criação do novo. **Assimetria intencional das estatísticas:** estatísticas de respostas já submetidas no abandonado **são preservadas** (já foram contabilizadas em QUIZ-RF-002 a cada resposta), mas as questões **não respondidas** do quiz abandonado **não contam como erro** — diferente de `finished`/`expired` (ver QUIZ-RF-003 CA-2 e QUIZ-RF-004 CA-3). O abandono não penaliza o cliente que perdeu conexão ou fechou o app por engano.
- **CA-7:** Entrada em `audit_log` apenas para casos sensíveis (criação rotineira de quiz não auditadas — alto volume).

**Erros previstos:**
- **E-1:** Simulado TAP impossível (matérias sem questões) → HTTP 409 com `{ missing_subjects: [...] }`.
- **E-2:** Matéria com menos de 5 questões publicadas → HTTP 409.
- **E-3:** Eixo com menos de 5 questões publicadas → HTTP 409.
- **E-4:** Acesso bloqueado por falta de assinatura ativa (QUIZ-RF-009) → HTTP 402.
- **E-5:** Validação Zod falha → HTTP 422.
- **E-6:** Soma dos `tap_weight` excede o teto de 60 → HTTP 409 com `{ reason: "tap_weight_overflow", total_weight }` (orienta o admin a revisar os pesos).

---

## QUIZ-RF-002 — Submeter resposta a uma questão

**Prioridade:** Essencial (MVP).
**Ator:** Cliente em quiz `in_progress`.

**Descrição:**
Cliente envia o índice da alternativa escolhida. Servidor valida, registra, atualiza estatísticas e devolve feedback. O conteúdo do feedback depende do `explanation_mode` do quiz.

**Critérios de aceitação:**
- **CA-1:** `POST /quizzes/:quiz_id/answers` com `{ position: number, submitted_index: 0..3 }`.
- **CA-2:** Validações:
  - Quiz pertence ao cliente, `status=in_progress`, não expirou pelo cronômetro (QUIZ-RF-004).
  - `position` está dentro de `1..total_questions`.
  - Posição **ainda não respondida** — re-submissão para mesma posição → E-2 (resposta é imutável).
- **CA-3:** Resposta:
  - Em **`after_each`**: HTTP 200 com `{ correct: boolean, correct_index, explanation, source_reference }`.
  - Em **`at_end`**: HTTP 200 com `{ recorded: true }` (sem revelar acerto/erro nem gabarito).
- **CA-4:** Servidor grava `submitted_index`, `is_correct`, `answered_at` em `quiz_session_questions`; incrementa `answered_count`/`correct_count` na sessão; atualiza `user_subject_stats` da matéria correspondente (`total_answers += 1`, `correct_answers += is_correct ? 1 : 0`).
- **CA-5:** Quando `answered_count == total_questions`, sistema finaliza automaticamente o quiz (chama o efeito de QUIZ-RF-003 internamente). Resposta da última submissão inclui `quiz_finished: true` para o frontend redirecionar à tela de resultado.
- **CA-6:** Quiz já finalizado/expirado/abandonado → E-1.

**Erros previstos:**
- **E-1:** Quiz não está `in_progress` → HTTP 409.
- **E-2:** Posição já respondida → HTTP 409.
- **E-3:** Posição fora do intervalo → HTTP 422.
- **E-4:** Quiz não pertence ao cliente → HTTP 404.

---

## QUIZ-RF-003 — Finalizar quiz manualmente

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.
**Pré-condições:** Quiz pertence ao cliente e está `in_progress`.

**Descrição:**
Cliente decide encerrar antes de responder todas. Questões não respondidas contam como **erradas** (não respondidas = erro) para fins de estatística da matéria, e ficam marcadas como `is_correct=false`, `submitted_index=null`, `answered_at=null`.

**Critérios de aceitação:**
- **CA-1:** `POST /quizzes/:quiz_id/finish`.
- **CA-2:** Servidor marca questões não respondidas como `is_correct=false` (sem `submitted_index`), atualiza `user_subject_stats` correspondentes (`total_answers += 1` para cada uma; `correct_answers` não incrementa).
- **CA-3:** Quiz vira `status=finished`, `finished_at=now()`. Retorna o resultado (mesmo payload de QUIZ-RF-005).
- **CA-4:** UI exige confirmação se houver questões não respondidas: "Você tem N questões em branco — elas contarão como erro. Encerrar mesmo assim?"

**Erros previstos:**
- **E-1:** Quiz não está `in_progress` → HTTP 409.

---

## QUIZ-RF-004 — Encerramento automático pelo cronômetro

**Prioridade:** Essencial (MVP).
**Ator:** Sistema.
**Pré-condições:** Quiz com `timer_enabled=true` e `time_limit_seconds` definido.

**Descrição:**
Quando o cronômetro chega a zero, qualquer submissão posterior é rejeitada e o quiz é finalizado como `expired`. Servidor é a **autoridade do tempo** — frontend exibe contagem, mas servidor revalida em cada submissão.

**Critérios de aceitação:**
- **CA-1:** Em cada `POST /quizzes/:quiz_id/answers`, servidor calcula `elapsed = now() - started_at`. Se `elapsed > time_limit_seconds + tolerância (5s)`, transição automática para `status=expired`.
- **CA-2:** Resposta para a submissão tardia: HTTP 409 com `{ reason: "quiz_expired", elapsed_seconds, time_limit_seconds }`. Frontend redireciona à tela de resultado.
- **CA-3:** Questões ainda não respondidas são tratadas como em QUIZ-RF-003 CA-2 (contam como erradas).
- **CA-4:** Job periódico (5 em 5 minutos) trata duas situações:
  - **Expiração por cronômetro:** quizzes `in_progress` com `timer_enabled=true` e `started_at + time_limit + 5s < now()` são marcados como `expired` (questões não respondidas contam como erro).
  - **Auto-abandono por inatividade:** quizzes `in_progress` (com **ou sem** cronômetro) sem submissão nova nas últimas **24 horas** são marcados como `abandoned`. Diferente do `expired`, questões não respondidas no auto-abandono **não contam como erro** (mesma regra de QUIZ-RF-001 CA-6 — assimetria intencional).
- **CA-5:** Tolerância de 5s acomoda latência de rede; não é "tempo extra" anunciado ao usuário.
- **CA-6:** A regra de 24h do auto-abandono é fixa no MVP; não há indicador visual ao cliente do tempo restante para o abandono.

---

## QUIZ-RF-005 — Visualizar resultado de quiz

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Tela de resultado mostrada ao finalizar (manual, automático ou por expiração). Também acessível para revisitar quizzes passados via QUIZ-RF-006/007.

**Critérios de aceitação:**
- **CA-1:** `GET /quizzes/:quiz_id/result` retorna:
  ```json
  {
    "quiz_id": "...",
    "mode": "...",
    "status": "finished" | "expired" | "abandoned",
    "started_at": "...", "finished_at": "...",
    "duration_seconds": 432,
    "total_questions": 20,
    "answered_count": 18,
    "correct_count": 13,
    "accuracy": 0.65,
    "breakdown_by_subject": [
      { "subject_id": "...", "subject_name": "...", "total": 5, "correct": 3, "accuracy": 0.6 }
    ],
    "questions": [
      { "position": 1, "question_id": "...", "subject_name": "...", "statement": "...", "image_url": null,
        "alternatives": ["...","...","...","..."], "submitted_index": 2, "correct_index": 1,
        "is_correct": false, "explanation": "...", "source_reference": "..." }
    ]
  }
  ```
- **CA-2:** Lista de questões usa o **snapshot** gravado em `quiz_session_questions` — edições posteriores à pergunta original (CONT-RF-011) **não afetam** o resultado deste quiz.
- **CA-3:** Independente do `explanation_mode` da sessão, na tela de resultado o cliente vê **sempre** os gabaritos e justificativas de todas as questões.
- **CA-4:** Quiz `abandoned` (substituído por novo via QUIZ-RF-001 CA-6) também é consultável aqui, com `status=abandoned`.

**Erros previstos:**
- **E-1:** Quiz não pertence ao cliente → HTTP 404.
- **E-2:** Quiz ainda `in_progress` → HTTP 409 ("Finalize o quiz para ver o resultado").

---

## QUIZ-RF-006 — Listar quizzes anteriores

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Histórico de quizzes do próprio cliente.

**Critérios de aceitação:**
- **CA-1:** `GET /me/quizzes` retorna `{ id, mode, scope_name?, status, started_at, finished_at, total_questions, correct_count, accuracy }`, paginado (padrão 20, ordenação por `started_at DESC`).
- **CA-2:** Filtros: `mode`, `status` (`finished`/`expired`/`abandoned`; `in_progress` excluído por padrão), intervalo de datas.
- **CA-3:** Sem retenção automática — histórico persiste indefinidamente.

---

## QUIZ-RF-007 — Painel de desempenho

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Visão agregada do progresso do cliente, organizada por matéria e por eixo. É o "como estou no estudo".

**Critérios de aceitação:**
- **CA-1:** `GET /me/performance` retorna:
  ```json
  {
    "overall": { "total_answers": 412, "correct_answers": 271, "accuracy": 0.658, "quizzes_finished": 18 },
    "by_axis": [
      { "axis_id": "...", "axis_name": "Salvamento",
        "total": 180, "correct": 120, "accuracy": 0.667,
        "subjects": [
          { "subject_id": "...", "subject_name": "Salvamento Terrestre",
            "total": 90, "correct": 65, "accuracy": 0.722, "last_answered_at": "...", "status_badge": "forte" }
        ]
      }
    ],
    "weakest_subjects": [ /* top 3 com menor accuracy entre matérias com ≥ 10 respostas */ ],
    "strongest_subjects": [ /* top 3 com maior accuracy entre matérias com ≥ 10 respostas */ ],
    "stats_reset_at": null
  }
  ```
- **CA-2:** `status_badge` por matéria: `unrated` (< 10 respostas), `fraco` (< 50%), `medio` (50–75%), `forte` (≥ 75%). Diferente da dificuldade da pergunta (CONT-RF-017) — esta é específica do cliente.
- **CA-3:** Dados derivados de `user_subject_stats`; estatísticas por eixo somadas em runtime a partir das matérias (cache pode ser adicionado posteriormente se houver problema de performance).
- **CA-4:** Endpoint alvo P95 < 400ms.

---

## QUIZ-RF-008 — Reset de estatísticas do cliente

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Cliente zera o próprio histórico de desempenho. Atende cenários como "voltei a estudar depois de um ano" ou "minha média antiga não reflete onde estou agora".

**Critérios de aceitação:**
- **CA-1:** `POST /me/performance/reset` exige reautenticação (senha) e checkbox de confirmação ("Entendo que essa ação é irreversível").
- **CA-2:** Em sucesso, em uma única transação:
  - `user_subject_stats`: para cada linha do cliente, zera `total_answers` e `correct_answers`; preenche `stats_reset_at = now()`.
  - **Histórico de `quiz_sessions` permanece intacto** — quizzes passados continuam visíveis em QUIZ-RF-006, podem ser revisitados (QUIZ-RF-005), mas **não recontam** para as estatísticas agregadas a partir do reset.
- **CA-3:** Operação **não** afeta as estatísticas globais das perguntas (CONT-RF-017 — dificuldade da pergunta agrega sobre todos os clientes).
- **CA-4:** Entrada em `audit_log` com `action=reset_own_stats`.
- **CA-5:** Reset **total** (não seletivo por matéria no MVP) — ver pendência.

**Erros previstos:**
- **E-1:** Reautenticação falhou → HTTP 401.
- **E-2:** Checkbox não marcado → HTTP 422.

---

## QUIZ-RF-009 — Bloqueio de acesso por falta de assinatura ativa

**Prioridade:** Essencial (MVP).
**Ator:** Sistema (middleware transversal).
**Pré-condições:** Detalhes de assinatura/período gratuito definidos no Módulo 6.

**Descrição:**
Endpoints de quiz (`POST /quizzes`, `POST /quizzes/:id/answers`, `GET /me/performance`) só funcionam quando o cliente está em **período gratuito ativo** **ou** com **assinatura vigente** (paga ou doada). Bloqueio retorna HTTP 402 com payload sinalizando a UI a oferecer planos.

**Critérios de aceitação:**
- **CA-1:** Middleware avalia, a cada request a endpoint de quiz, se `user.access_status` é `active` (valor calculado pelo Módulo 6, consolidando trial + subscriptions). Se não, retorna HTTP 402 com `{ reason: "subscription_required", trial_used: boolean, last_active_until: "..."? }`.
- **CA-2:** Visualização do **histórico** (QUIZ-RF-006) e do resultado de quizzes passados (QUIZ-RF-005) **permanece disponível** mesmo sem assinatura ativa — é apenas leitura de dados próprios.
- **CA-3:** Painel de desempenho (QUIZ-RF-007) e evolução temporal (QUIZ-RF-010) também permanecem em leitura (cliente pode acompanhar o histórico antigo mesmo sem renovar — incentiva volta).
- **CA-4:** Iniciar novo quiz e responder questão **exigem** acesso ativo.

---

## QUIZ-RF-010 — Evolução temporal do desempenho

**Prioridade:** Essencial (MVP).
**Ator:** Cliente.

**Descrição:**
Mostra a evolução do desempenho do próprio cliente **ao longo do tempo** (agregação mensal). Resposta à demanda do cliente de "ver onde estava em janeiro vs. fevereiro vs. março" — sem comparação com outros usuários (não há leaderboard no MVP).

**Critérios de aceitação:**
- **CA-1:** `GET /me/performance/timeline?months=12` retorna:
  ```json
  {
    "stats_reset_at": null,
    "months": [
      { "month": "2026-05", "total_answers": 84, "correct_answers": 62, "accuracy": 0.738, "quizzes_finished": 4 },
      { "month": "2026-04", "total_answers": 0, "correct_answers": 0, "accuracy": null, "quizzes_finished": 0 },
      { "month": "2026-03", "total_answers": 120, "correct_answers": 70, "accuracy": 0.583, "quizzes_finished": 6 }
    ]
  }
  ```
- **CA-2:** Parâmetro `months` é opcional (default **12**, mínimo **1**, máximo **24**).
- **CA-3:** Meses sem atividade aparecem no array com zeros e `accuracy: null` (frontend desenha continuidade do gráfico — interpola ou exibe gap, decisão de UI).
- **CA-4:** Dados são derivados de `quiz_session_questions` com `answered_at != null`, agrupados por mês civil no fuso `America/Sao_Paulo`. Não materializa snapshot mensal no MVP — agregação é em runtime (porte do app justifica).
- **CA-5:** Reset de estatísticas (QUIZ-RF-008) **trunca** a timeline: meses anteriores a `stats_reset_at` ficam zerados, e o campo `stats_reset_at` no payload sinaliza a UI para exibir uma marca visual ("você zerou as estatísticas em X").
- **CA-6:** Endpoint alvo P95 < 500ms para usuário com até ~3 anos de histórico (cenário extremo).
- **CA-7:** Quizzes `abandoned`/`expired`/`in_progress` contribuem **apenas com respostas efetivamente submetidas** (mesma regra de assimetria de QUIZ-RF-001 CA-6).

---

## Pendências deste módulo — resolvidas em 2026-05-28

- ✅ **QUIZ-P-02 — Pause/resume.** Decidido **manter sem pause no MVP**. Em troca, a assimetria de estatísticas foi formalizada (QUIZ-RF-001 CA-6, QUIZ-RF-004 CA-4): quiz `abandoned` preserva apenas respostas efetivamente submetidas; questões não respondidas **não contam como erro**. Quiz sem atividade por 24h vira `abandoned` automaticamente. Reavaliar pause/resume se o uso do simulado TAP (50 questões) mostrar fricção.
- ✅ **QUIZ-P-03 — Reset seletivo por matéria.** Decidido **adiar**. QUIZ-RF-008 mantém reset total; reset seletivo é considerado exagero para o porte e cenários típicos.
- ✅ **QUIZ-P-06 — Tempo customizável / UI do cronômetro.** Decidido: **toggle/switch principal** "Cronômetro: ligado/desligado" na tela de iniciar quiz; quando ligado, expõe slider 1–5 min/questão; quando desligado, oculta o slider. Formalizado em QUIZ-RF-001 CA-1.
- ✅ **QUIZ-P-07 — Leaderboard.** Decidido **não implementar** leaderboard entre clientes — conflita com privacidade e o porte do app limita a utilidade. **Em substituição,** criado **QUIZ-RF-010 — Evolução temporal do desempenho** (timeline mensal própria do usuário): "% de acerto em janeiro, fevereiro, março".

## Pendências deste módulo — adiadas conscientemente para pós-MVP

- **QUIZ-P-01 — Modo offline.** Quiz totalmente offline com sincronização. Exige resolução de conflitos de stats; reavaliar após observação do uso real do PWA.
- **QUIZ-P-04 — Modo "pontos fracos".** Sistema sorteia das matérias com menor acurácia do cliente. Requer volume mínimo de histórico para ser útil (≥ 10 respostas/matéria); reavaliar quando base de usuários gerar volume.
- **QUIZ-P-05 — Filtro por nível de dificuldade no sorteio.** "Só hards" / "só easies". Sorteio uniforme em `published` é suficiente no MVP; reavaliar quando volume de questões por dificuldade tornar o filtro útil.
