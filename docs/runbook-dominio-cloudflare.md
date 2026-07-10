# Runbook — Configurar Cloudflare para bomberquiz.com.br (Fase 2 do domínio)

> Detalha em passo a passo a execução do checklist "Próximo passo — Domínio e
> e-mail" em [`tarefas.md`](tarefas.md#próximo-passo--domínio-e-e-mail-fase-2-antecipada).
> Quando todos os passos abaixo estiverem concluídos e validados, marcar os
> itens correspondentes em `tarefas.md` como feitos (ver Passo 8).

## Contexto

O domínio `bomberquiz.com.br` foi registrado em 2026-07-10. O projeto já tinha
essa migração planejada em detalhe:

- **[`arquitetura.md`](arquitetura.md)** (seção "Domínios, URLs e cookies") define a
  topologia de Fase 2: frontend em `app.bomberquiz.com.br` → Cloudflare Pages,
  backend em `api.bomberquiz.com.br` → Fly.io, cookie `Domain=.bomberquiz.com.br`.
- **ADR-0028** ([`decisoes.md:239`](decisoes.md)) decidiu: `.com.br` é o domínio
  oficial, `.com` é só redirect 301; e-mail transacional (Resend) fica isolado em
  `send.bomberquiz.com.br` para não colidir com o MX da caixa humana de suporte
  (`suporte@bomberquiz.com.br`, Hostinger Business) na raiz.
- **[`tarefas.md:92-99`](tarefas.md)** ("Próximo passo — Domínio e e-mail") já
  lista a ordem de execução dos 5 passos, travada apenas pela confirmação do
  registro — que agora aconteceu.

**Divergência resolvida com o usuário (2026-07-10):** o pedido original de
configuração mencionava `www.bomberquiz.com.br`, mas a arquitetura documentada
usa `app.`. Decisão: manter `app.bomberquiz.com.br` como origem real (Cloudflare
Pages) e adicionar `www.bomberquiz.com.br` como redirect 301 para `app.`,
cobrindo quem digitar `www` por hábito — sem conflitar com a ADR-0028.

**Sobre "quantos subdomínios tenho direito":** não há limite prático. Uma zona
Cloudflare (mesmo no plano Free) suporta milhares de registros DNS — os 4
subdomínios deste plano (`app`, `www`, `api`, `send`) mais o registro MX da raiz
não chegam nem perto de qualquer limite.

## Passo 0 — Zona no Cloudflare

- [x] **2026-07-10** — Nameservers trocados no **hPanel da Hostinger**
  (Domínios → `bomberquiz.com.br` → Nameservers/DNS) — **não** no site do
  Registro.br diretamente. O domínio foi registrado via Hostinger, reseller
  autorizada do Registro.br para `.com.br`; é a Hostinger quem sincroniza a
  mudança de nameservers com o Registro.br por trás. Propagação estimada:
  1-2h (pode levar até 24-48h em casos raros de `.com.br`).
- [ ] Conferir status **Active** da zona `bomberquiz.com.br` no dashboard
  Cloudflare antes de seguir para o Passo 1 — os passos abaixo (Pages custom
  domain, DKIM etc.) exigem zona ativa.

## Passo 1 — DNS de `send.bomberquiz.com.br` (Resend)

- [ ] Resend Dashboard → **Domains** → **Add Domain** → `send.bomberquiz.com.br`
      (não a raiz — é isso que evita o conflito de MX com o Hostinger).
- [ ] Resend gera um conjunto de registros (tipicamente: 1 `MX`, 1 `TXT` SPF, e
      2-3 `CNAME` de DKIM, todos com host prefixado por `send.`). Copiar
      exatamente como mostrado.
- [ ] Criar cada registro em Cloudflare DNS → **Records** → **Add record**, com
      **Proxy status = DNS only (nuvem cinza)** — MX/TXT não podem ser
      proxiados, e os CNAME de DKIM também precisam ficar DNS-only para o
      Resend validar.
- [ ] Voltar ao Resend e clicar **Verify DNS Records**. Pode levar alguns
      minutos até propagar.

## Passo 2 — MX da raiz `bomberquiz.com.br` (Hostinger Business Email)

- [ ] Resgatar a caixa Hostinger Business (Starter) já disponível, associando-a
      a `bomberquiz.com.br`.
- [ ] Hostinger fornece os registros MX (e geralmente SPF/DKIM próprios) para a
      **raiz** do domínio — não confundir com os do Passo 1, que são para
      `send.`.
- [ ] Criar os registros em Cloudflare DNS, host = `@` (raiz), **Proxy status =
      DNS only**.
- [ ] Confirmar recebimento criando/testando `suporte@bomberquiz.com.br`.

## Passo 3 — Atualizar `EMAIL_FROM` (código + secret)

Hoje `EMAIL_FROM` usa o sandbox `onboarding@resend.dev` (paliativo desde
2026-07-08, ver `api/src/infra/email/resend.adapter.ts` e `api/.env.example:17`).
Só depois do Passo 1 verificado no Resend:

- [ ] Editar `EMAIL_FROM` para algo como
      `BomberQuiz <noreply@send.bomberquiz.com.br>` em `api/.env` (local) e, em
      produção, via `fly secrets set
      EMAIL_FROM="BomberQuiz <noreply@send.bomberquiz.com.br>" -a bomberquiz-api`.
- [ ] Repetir para `bomberquiz-api-staging` se aplicável.
- [ ] Disparar um e-mail de teste (ex. fluxo de verificação de cadastro) para
      confirmar entrega antes de seguir.

## Passo 4 — Frontend: `app.bomberquiz.com.br` + `www.bomberquiz.com.br`

- [ ] Cloudflare Dashboard → **Workers & Pages** → projeto `bomberquiz-web` →
      **Custom domains** → **Set up a domain** → `app.bomberquiz.com.br`. Como
      a zona já está ativa no Cloudflare, o CNAME é criado **automaticamente**
      (não criar manualmente — evita erro 522). O proxy fica ligado por
      padrão, o que é o esperado para Pages.
- [ ] Para `www.bomberquiz.com.br` → redirect 301 para `app.bomberquiz.com.br`:
      Cloudflare Dashboard → zona `bomberquiz.com.br` → **Rules → Redirect
      Rules** → criar regra: se `Hostname equals www.bomberquiz.com.br`,
      redirecionar (301, preserve path/query) para
      `https://app.bomberquiz.com.br/$1`. Isso exige um registro DNS proxied
      para `www` (CNAME para `app.bomberquiz.com.br` ou para o próprio Pages,
      proxy ligado) para o Cloudflare conseguir interceptar a requisição antes
      de aplicar o redirect.
- [ ] Testar `https://app.bomberquiz.com.br` e `https://www.bomberquiz.com.br`
      no navegador (SSL deve emitir automaticamente via Cloudflare, minutos).

## Passo 5 — Backend: `api.bomberquiz.com.br` (Fly.io)

- [ ] `flyctl certs add api.bomberquiz.com.br -a bomberquiz-api` (dispara
      emissão do certificado Let's Encrypt no Fly).
- [ ] `flyctl certs show api.bomberquiz.com.br -a bomberquiz-api` mostra o
      registro DNS exigido (normalmente um `CNAME api → bomberquiz-api.fly.dev`,
      ou A/AAAA para os IPs anycast do Fly).
- [ ] Criar esse registro em Cloudflare DNS com **Proxy status = DNS only
      (nuvem cinza)**. Recomendado manter DNS-only para a API: com o proxy
      Cloudflare ligado, o desafio de emissão do certificado do Fly pode
      falhar, e adiciona duplo-proxy desnecessário para uma API com
      sessão/cookie. Proxy Cloudflare pode ser ativado depois, como melhoria
      opcional (WAF), só após o certificado já emitido e trocando o SSL/TLS
      mode da zona para **Full (strict)**.
- [ ] Confirmar com `flyctl certs show` até o status virar `Ready`, depois
      testar `https://api.bomberquiz.com.br/health`.

## Passo 6 — Redirect `bomberquiz.com` → `bomberquiz.com.br`

Duas opções, em ordem de simplicidade:

- **Opção A (mais simples):** se o registrador do `.com` (não é Registro.br,
  que só administra `.br`) já oferece *domain forwarding* nativo, usar isso —
  não precisa tocar em Cloudflare nem mudar nameservers do `.com`.
- **Opção B (via Cloudflare):** adicionar `bomberquiz.com` como segunda zona
  Cloudflare (Passo 0 repetido para esse domínio), depois criar uma **Redirect
  Rule** dinâmica (disponível no Free) na zona `bomberquiz.com`: match wildcard
  de hostname/path → `https://bomberquiz.com.br/${1}` com preserve query
  string, 301 permanente. Exige registro DNS proxied (`A`/`CNAME` fictício,
  ex. `192.0.2.1` proxied) para o Cloudflare interceptar antes do redirect.

- [ ] Escolher opção e configurar.
- [ ] Testar `https://bomberquiz.com` → redireciona para
      `https://bomberquiz.com.br`.

## Passo 7 — Migrar URLs de Fase 2 (código + secrets)

Só depois dos passos 4 e 5 confirmados funcionando:

- [ ] `api`: `fly secrets set WEB_ORIGIN=https://app.bomberquiz.com.br
      API_BASE_URL=https://api.bomberquiz.com.br
      COOKIE_DOMAIN=.bomberquiz.com.br -a bomberquiz-api`. Conferir CORS em
      `api/src/infra/auth/better-auth.ts` / config relacionada — trocar o
      `Origin` permitido de `https://bomberquiz-web.pages.dev` para
      `https://app.bomberquiz.com.br`.
- [ ] `web`: no Cloudflare Pages (projeto `bomberquiz-web`) → **Settings →
      Environment variables** → `VITE_API_BASE_URL=https://api.bomberquiz.com.br`
      para o ambiente de produção. Rebuild/redeploy do projeto.
- [ ] Atualizar a URL de webhook do Mercado Pago no painel deles para
      `https://api.bomberquiz.com.br/webhooks/mercado-pago` (mencionado em
      `arquitetura.md:219`).
- [ ] Validação manual ponta a ponta: cadastro, login, verificação de e-mail,
      cookie de sessão no DevTools com `Domain=.bomberquiz.com.br`.

## Passo 8 — Housekeeping na documentação

- [ ] Marcar os itens correspondentes em `tarefas.md:92-100` como concluídos.
- [ ] Registrar em `tarefas.md` a decisão de manter `app.` como canônico e
      `www.` só como redirect (aditiva à ADR-0028, não a contradiz).

## Verificação end-to-end

- `dig`/`nslookup` (ou whatsmydns.net) para cada subdomínio antes de testar no
  navegador — evita debugar "erro" que na verdade é propagação de DNS.
- Resend Dashboard deve mostrar `send.bomberquiz.com.br` como **Verified**.
- `flyctl certs show api.bomberquiz.com.br` deve mostrar **Ready**.
- Cloudflare Pages custom domain deve mostrar **Active** para `app.` e `www.`
  deve responder com 301 para `app.`.
- Fluxo de cadastro real em `https://app.bomberquiz.com.br` deve enviar e-mail
  de verificação a partir de `send.bomberquiz.com.br` e não do sandbox Resend.
