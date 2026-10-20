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
  Exceção: `187.7.200.0/24` é do utilizador (inetnum `net-IGOR-BISSOLI-ZULCAO`, via IPBNB) e foi incluído a pedido dele antes de ser anunciado (sem ROA em 2026-10-20).
- NÃO incluir `192.67.35.12/31` (sub-rede da HYEHOST, não é do utilizador). O `5.231.99.0/24` (inetnum da GHOSTnet) é alugado pelo utilizador e entra.
- Cada prefixo anunciado está dividido em duas metades (/48 → 2×/49, /24 → 2×/25): a **primeira metade** é `PT,PT-11,Alverca do Ribatejo`, a **segunda** `BR,BR-ES,Vitória`. Ao acrescentar um prefixo novo, seguir a mesma regra.
- Geofeed e prefixlen cobrem os mesmos prefixos anunciados, mas não precisam da mesma granularidade: o split PT/BR é só do geofeed.

## Regras do `prefixlen.csv` (RFC 9977)

- Formato: `prefix,end-site prefix length,end-sites` — exatamente 2 vírgulas; `1` no 3.º campo = sem CGN/proxy; `prefix,,` = não divulgar.
- **Linhas obrigatoriamente em CRLF** (ao contrário do geofeed). `.gitattributes` marca o ficheiro `-text` para o git não normalizar; editar preservando `\r\n`.
- Uma entrada por prefixo (duplicados são erro); sub-prefixos permitidos (longest-prefix match).
- Atual: um prefixo anunciado por linha, sem split — /48 IPv6 com end-sites `/64` (`/48,64,1`), /24 IPv4 com end-sites `/25` (`/24,25,1`, uma metade PT e outra BR).
- `public/_headers` dá `Cache-Control: public, max-age=604800` ao `/prefixlen.csv` (sec. 7: sinalizar refetch; dados mudam raramente). `_headers` não é servido.

## Publicação do geofeed e prefixlen (fora do repo)

Na RIPE DB o prefixlen usa o atributo `prefixlen:` no `inet6num` (já implementado pela RIPE NCC), com o URL `https://ispconfig.zulcao.com.br/prefixlen.csv`. Vai no mesmo pedido aos LIR que o `geofeed:`.

O atributo `geofeed:` **não é válido em `aut-num`** — só em `inetnum`/`inet6num` (RFC 9632). Os /48 não têm objeto próprio na RIPE DB; estão dentro dos blocos dos LIR patrocinadores, por isso a referência ao URL exige ticket a cada um:

- `2a06:b700:1003::/48` ⊂ `2a06:b700::/29` (BR-MIRALIUMRE, `miraliumre-mnt`)
- `2a0f:6280:1057::/48` ⊂ `2a0f:6280::/30` (HYEHOST, `HYEHOST-MNT`)
- `5.231.99.0/24` — inetnum próprio, `GHOSTNET-MNT` (GHOSTnet GmbH); sem `geofeed:`.
- `187.7.200.0/24` — inetnum próprio, `IPBNB-MNT`; já tem `geofeed:` a apontar para a IPBNB (`api.ipbnb.com`), que prevalece sobre o nosso enquanto não for trocado.

## Repositório

Remote: `git@github.com:zulcao/as213367-ispconfig.git` (público, org `zulcao`; a conta `gh` é `CattleWhisper`).
