# Rubrica de aprovação de perguntas — BomberQuiz

> Documento operacional para administradores. Não é especificação técnica.
> Versão inicial — refinar com a prática conforme volume de revisões cresce.

## Para que serve

Quando um parceiro envia uma pergunta para revisão (PART-RF-004), o admin de plantão precisa decidir entre **aprovar** (CONT-RF-015) ou **rejeitar** (CONT-RF-016). Sem critérios escritos, cada admin decide por intuição própria e o parceiro recebe feedback inconsistente.

Esta rubrica é o **checklist objetivo** que o admin consulta antes de bater o martelo. Aplique de cima para baixo — qualquer "não" em itens **bloqueantes** (marcados com 🚫) é motivo imediato de rejeição. Itens marcados com ✏️ podem ser **corrigidos diretamente pelo admin** (CONT-RF-011 antes de aprovar — CONT-RF-015 CA-5), evitando uma rodada inútil de rejeição.

---

## 1. Conteúdo (o que a pergunta afirma)

- 🚫 **Gabarito defensável.** A alternativa correta deve ser inequivocamente correta segundo a fonte oficial citada (manual, NT, lei). Se houver mais de uma alternativa que possa ser considerada correta, é rejeição.
- 🚫 **Sem informação fora do escopo do TAP.** A pergunta cobra conteúdo coberto pelo edital. Curiosidades laterais não entram.
- 🚫 **Justificativa fundamentada.** O campo `explanation` precisa explicar *por que* a correta é correta — preferencialmente citando o trecho do manual/norma. "Porque sim" ou paráfrase da alternativa não bastam.
- ✏️ **Fonte declarada.** O campo `source_reference` (texto livre) deve apontar o documento e, se possível, capítulo/artigo/página. Se ausente mas a justificativa for boa, admin pode preencher e aprovar.

## 2. Forma das alternativas

- 🚫 **Exatamente 4 alternativas distintas.** Repetições ou alternativas quase idênticas (que só diferem por palavra-acessório) são rejeição.
- 🚫 **Sem "obviamente erradas".** Toda alternativa deve ser **plausível** para alguém que não estudou aquela parte do manual. Distratores absurdos (ex: "atirar no incêndio") entregam o gabarito por exclusão.
- ✏️ **Comprimento similar entre alternativas.** Alternativas muito longas ou muito curtas em relação às outras tendem a chamar atenção e enviesar a escolha. Admin pode reescrever para uniformizar.
- 🚫 **Sem "todas as anteriores" ou "nenhuma das anteriores".** Atrapalham o sorteio aleatório de alternativas no quiz.

## 3. Enunciado

- 🚫 **Pergunta clara e única.** O candidato deve saber exatamente o que está sendo perguntado, sem precisar interpretar duplo sentido.
- ✏️ **Sem pegadinha que confunde mais que ensina.** Negações triplas ("qual NÃO é o procedimento que não deve ser evitado") são para excluir, não para testar conhecimento.
- ✏️ **Português correto.** Erros grosseiros de concordância, ortografia ou pontuação. Admin pode corrigir antes de aprovar.
- 🚫 **Sem dados pessoais ou ofensivos.** Pergunta não cita pessoas reais, casos reais identificáveis, nem usa linguagem ofensiva.

## 4. Imagem (quando houver)

- 🚫 **Imagem legível.** Se a pergunta depende da imagem para ser respondida, ela tem que estar nítida em tela de celular.
- 🚫 **Imagem relevante.** Imagem decorativa sem relação com o conteúdo é rejeição (engana o usuário a procurar pistas onde não há).
- 🚫 **Sem direito autoral comprometido.** Imagem deve ser de fonte oficial (CBMGO, ABNT, manuais públicos) ou produção própria. Sem fotos genéricas raspadas da web.

## 5. Duplicidade

- ✏️ **Sem duplicata óbvia.** Antes de aprovar, busque por palavras-chave do enunciado no catálogo (CONT-RF-009). Se houver duplicata praticamente idêntica, rejeite a nova **ou** aprove a melhor das duas e arquive a outra.

---

## Quando rejeitar vs. aprovar com edição

| Situação | Ação |
|---|---|
| Gabarito errado | 🚫 **Rejeitar.** Não corrigir gabarito de outro autor — pode haver entendimento diferente da fonte. Devolva com explicação. |
| Alternativas absurdas / repetidas | 🚫 **Rejeitar.** É retrabalho conceitual; parceiro precisa pensar de novo. |
| Justificativa rasa ou ausente | 🚫 **Rejeitar** se conteúdo principal estiver fraco; ✏️ **completar e aprovar** se o admin tiver a fonte à mão e a pergunta no geral for boa. |
| Erro de português / formatação | ✏️ **Editar e aprovar.** Não devolve rodada inútil por vírgula. |
| Fonte ausente mas pergunta boa | ✏️ **Preencher fonte e aprovar.** |
| Duplicata exata de pergunta já publicada | 🚫 **Rejeitar** com link/ID da existente. |

---

## Como escrever um motivo de rejeição útil

O motivo (CONT-RF-016 CA-1, 10–500 caracteres) é a única coisa que o parceiro vê. Escreva **acionável**, não genérico:

- ❌ "Pergunta confusa." → não diz o que mudar.
- ✅ "O enunciado tem dupla negação ('procedimento que não deve ser evitado'). Reescreva afirmativamente: 'qual procedimento deve ser adotado?'"

- ❌ "Gabarito errado." → não diz qual deveria ser.
- ✅ "Segundo a NT-11 art. 5º §2º, a saída de emergência deve ter no mínimo 1,20m. A alternativa C (1,10m) está incorreta; a correta é a B (1,20m)."

- ❌ "Alternativas ruins."
- ✅ "As alternativas B e C dizem essencialmente a mesma coisa ('uso de capacete' vs 'utilização de capacete'). Substitua uma delas por um distrator diferente, como 'uso de óculos de proteção'."

Pense no parceiro relendo o motivo às 22h tentando entender o que arrumar — quanto mais específico, menos atrito.

---

## O que **não** é motivo de rejeição

Não rejeite por:
- Estilo de escrita ligeiramente diferente do habitual (desde que correto).
- Tema "menos importante" da matéria — se está no edital, vale.
- Preferência pessoal por outro distrator igualmente plausível.
- Imagem desnecessária mas inofensiva (apenas peça remoção via edição).

Diferenças de gosto não bloqueiam aprovação. Diferenças de **conteúdo objetivo** bloqueiam.

---

## Métricas que o admin pode acompanhar

- **Tempo médio de revisão** (data de submissão → aprovação/rejeição). Objetivo: 48h corridas no MVP.
- **Taxa de rejeição por parceiro.** Acima de 50% sugere problema sistêmico — vale conversar 1:1 antes de continuar rejeitando.
- **Recorrência de motivos.** Se o mesmo motivo aparece muito, pode virar treinamento/FAQ para os parceiros.

Esses indicadores não estão expostos no MVP — surgem da consulta do `audit_log`.

---

## Histórico de versões

- **v1 — 2026-05-28** — Versão inicial. Resolve PART-P-04.
