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
- NÃO incluir `192.67.35.12/31` (sub-rede da HYEHOST, não é do utilizador). Um futuro bloco IPv4 próprio, anunciado pelo AS213367, entra com `PT`.

## Publicação do geofeed (fora do repo)

O atributo `geofeed:` **não é válido em `aut-num`** — só em `inetnum`/`inet6num` (RFC 9632). Os /48 não têm objeto próprio na RIPE DB; estão dentro dos blocos dos LIR patrocinadores, por isso a referência ao URL exige ticket a cada um:

- `2a06:b700:1003::/48` ⊂ `2a06:b700::/29` (BR-MIRALIUMRE, `miraliumre-mnt`)
- `2a0f:6280:1057::/48` ⊂ `2a0f:6280::/30` (HYEHOST, `HYEHOST-MNT`)

## Repositório

Remote: `git@github.com:zulcao/as213367-ispconfig.git` (público, org `zulcao`; a conta `gh` é `CattleWhisper`).
