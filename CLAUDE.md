# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é

Geofeed RFC 8805 do **AS213367**, servido como um único ficheiro estático (`public/geofeed.csv`) em Cloudflare Workers Static Assets. **Não há código de Worker** (`wrangler.jsonc` não tem `main`) — isto é intencional; não acrescentar `src/` nem um Worker para servir o CSV. O utilizador comunica em português europeu.

`AGENTS.md` é o texto genérico do template Cloudflare (Local Explorer, KV/D1/etc.) e não se aplica a este projeto, que não tem bindings.

## Comandos

```bash
npm install                      # wrangler local (devDependency); usar npx, não o wrangler do Homebrew
npx wrangler dev                 # local em http://localhost:8787/geofeed.csv  (/ dá 404, é normal)
npx wrangler deploy --dry-run    # valida: deve ler 1 ficheiro, "No bindings found"
npx wrangler deploy              # publica em https://ispconfig.zulcao.com.br/geofeed.csv
```

O domínio é um Custom Domain (`routes[].custom_domain` no `wrangler.jsonc`): a Cloudflare cria o registo DNS e o certificado no deploy. A zona `zulcao.com.br` está na conta Cloudflare do utilizador. Com `routes` definido, o URL `workers.dev` fica desligado por omissão.

Não há build, testes nem lint.

## Regras do `geofeed.csv`

- Formato por linha: `prefix,country,region,city,postal` — sempre 5 campos; campos vazios ficam vazios (ex.: postal). Comentários com `#`. LF, UTF-8.
- `country` em ISO 3166-1 alpha-2 (`PT`); `region` em código ISO 3166-2 (`PT-11`, não "Lisboa"); `city` é texto livre.
- Só prefixos **anunciados pelo AS213367**. Confirmar antes de acrescentar:
  `curl -s "https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS213367"`
- Prefixos sem sobreposição, em notação de rede estrita (`ipaddress.ip_network(p, strict=True)`).
  Exceção: `2a0f:6280:1f00::/40` (HYEHOST) foi incluído a pedido do utilizador antes de ser anunciado (ROA AS213367 maxLength 48 e `route6` já existem em 2026-10-21).
- NÃO incluir `192.67.35.12/31` (sub-rede da HYEHOST, não é do utilizador).
- `5.231.99.0/24` (GHOSTnet): lease de 1 mês **já cancelado**; em uso só até ~2026-11-10 (lease termina ~2026-11-13). Retirar as linhas do geofeed e do prefixlen quando o utilizador deixar de o anunciar — não publicar dados sobre espaço que já não é dele. Não vale a pena corrigir os inetnum por IP (limitação do painel GHOSTnet). O bloco IPv4 a manter é o `187.7.200.0/24`.
- Cada prefixo anunciado está dividido em duas metades (/48 → 2×/49, /40 → 2×/41, /24 → 2×/25): a **primeira metade** é `PT,PT-11,Alverca do Ribatejo`, a **segunda** `BR,BR-ES,Vitória`. Ao acrescentar um prefixo novo, seguir a mesma regra.
- Geofeed e prefixlen cobrem os mesmos prefixos anunciados, mas não precisam da mesma granularidade: o split PT/BR é só do geofeed.

## Regras do `prefixlen.csv` (RFC 9977)

- Formato: `prefix,end-site prefix length,end-sites` — exatamente 2 vírgulas; `1` no 3.º campo = sem CGN/proxy; `prefix,,` = não divulgar.
- **Linhas obrigatoriamente em CRLF** (ao contrário do geofeed). `.gitattributes` marca o ficheiro `-text` para o git não normalizar; editar preservando `\r\n`.
- Uma entrada por prefixo (duplicados são erro); sub-prefixos permitidos (longest-prefix match).
- Atual: um prefixo anunciado por linha, sem split. End-site = **sítio** (o utilizador não tem clientes; tem dois sítios, PT e BR), por isso o tamanho é o de cada metade do geofeed: `/48,49,1`, `/40,41,1`, `/24,25,1`.
- `public/_headers` dá `Cache-Control: public, max-age=604800` ao `/prefixlen.csv` (sec. 7: sinalizar refetch; dados mudam raramente). `_headers` não é servido.

## Publicação do geofeed e prefixlen (fora do repo)

Os atributos `geofeed:` (RFC 9632) e `prefixlen:` (RFC 9977) existem no `inetnum`/`inet6num` da RIPE DB (opcionais, um de cada); **não** no `aut-num`. Os consumidores usam o inet(6)num **mais específico** com referência. Alterações exigem o maintainer do objeto (os LIR), salvo onde houver `mnt-lower: IGBZU-mnt`.

Para consultar, incluir o objeto exato e os mais específicos — **não** usar só `-l` (devolve apenas o menos específico e esconde o exato):
`curl -s "https://rest.db.ripe.net/search.json?query-string=<prefixo>&type-filter=inet6num&flags=r&flags=B&flags=M"` (repetir sem `flags=M` para o exato).

Estado em 2026-10-21 (maintainer do utilizador: `IGBZU-mnt`; org: `ORG-IBZ2-RIPE`):

- `2a0f:6280:1f00::/40` (`ALLOCATED-BY-LIR`) + `/41` PT e `/41` BR (`ASSIGNED`) — HYEHOST-MNT; geofeed e prefixlen nossos nos três. ✔
- `2a0f:6280:1057::/48` (`ASSIGNED`, PT) — HYEHOST-MNT; geofeed e prefixlen nossos. ✔
- `2a06:b700:1003::/48` (`ASSIGNED`, PT, netname `Bissoli`, sem org) — miraliumre-mnt; **sem** geofeed nem prefixlen.
- `5.231.99.0/24` (a sair ~2026-11-10) — GHOSTnet criou 256 inetnum /32 (`.0–.127` PT, `.128–.255` BR, org `ORG-GG3-RIPE`); geofeed nosso, **sem** prefixlen.
- `187.7.200.0/24` (`ASSIGNED PA`, PT, org já `ORG-IBZ2-RIPE`) — IPBNB-MNT; geofeed ainda da IPBNB (`api.ipbnb.com`), **sem** prefixlen; ticket preparado (ROA, SUB-ALLOCATED PA + mnt-lower, ou 2×/25).

## Repositório

Remote: `git@github.com:zulcao/as213367-ispconfig.git` (público, org `zulcao`; a conta `gh` é `CattleWhisper`).
