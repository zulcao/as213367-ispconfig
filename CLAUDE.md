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

A zona tem o Browser Integrity Check ligado (bloqueia p.ex. `Python-urllib` com 403/erro 1010). Uma **Configuration Rule** (hostname `ispconfig.zulcao.com.br` → Browser Integrity Check Off) desliga-o só neste host, para os coletores de geofeed/prefixlen não serem bloqueados. Testar: `curl -s -o /dev/null -w '%{http_code}\n' -A "Python-urllib/3.9" https://ispconfig.zulcao.com.br/geofeed.csv` → `200`.

Não há build, testes nem lint.

## Regras do `geofeed.csv`

- Formato por linha: `prefix,country,region,city,postal` — sempre 5 campos; campos vazios ficam vazios (ex.: postal). Comentários com `#`. LF, UTF-8.
- `country` em ISO 3166-1 alpha-2 (`PT`); `region` em código ISO 3166-2 (`PT-11`, não "Lisboa"); `city` é texto livre.
- Só prefixos **anunciados pelo AS213367**. Confirmar antes de acrescentar:
  `curl -s "https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS213367"`
- Prefixos sem sobreposição, em notação de rede estrita (`ipaddress.ip_network(p, strict=True)`).
  Exceção: blocos atribuídos ao utilizador (inet(6)num com `org: ORG-IBZ2-RIPE`) entram a pedido dele antes de serem anunciados. Em 2026-10-22 ainda não anunciados: `2a0f:6280:1f00::/40`, `2a0c:9a40:e180::/48` e `2001:678:13a0::/48` (todos já com ROA e route6).
- NÃO incluir `192.67.35.12/31` (sub-rede da HYEHOST, não é do utilizador).
- `5.231.99.0/24` (GHOSTnet): lease de 1 mês **já cancelado**; em uso só até ~2026-11-10 (lease termina ~2026-11-13). Retirar as linhas do geofeed e do prefixlen quando o utilizador deixar de o anunciar — não publicar dados sobre espaço que já não é dele. Não vale a pena corrigir os inetnum por IP (limitação do painel GHOSTnet). O bloco IPv4 a manter é o `187.7.200.0/24`.
- Cada prefixo anunciado está dividido em duas metades (/48 → 2×/49, /40 → 2×/41, /24 → 2×/25): a **primeira metade** é `PT,PT-11,Alverca do Ribatejo`, a **segunda** `BR,BR-ES,Vitória`. Ao acrescentar um prefixo novo, seguir a mesma regra.
- Geofeed e prefixlen cobrem os mesmos prefixos anunciados, mas não precisam da mesma granularidade: o split PT/BR é só do geofeed.

## Regras do `prefixlen.csv` (RFC 9977)

- Formato: `prefix,end-site prefix length,end-sites` — exatamente 2 vírgulas; `1` no 3.º campo = sem CGN/proxy; `prefix,,` = não divulgar.
- **Linhas obrigatoriamente em CRLF** (ao contrário do geofeed). `.gitattributes` marca o ficheiro `-text` para o git não normalizar; editar preservando `\r\n`.
- Uma entrada por prefixo (duplicados são erro); sub-prefixos permitidos (longest-prefix match).
- Atual: um prefixo anunciado por linha, sem split. End-site = **sítio** (o utilizador não tem clientes; tem dois sítios, PT e BR), por isso o tamanho é o de cada metade do geofeed: `/48,49,1`, `/24,25,1`. Exceções: o `2a0f:6280:1f00::/40` e o `187.7.200.0/24` têm inet(6)num próprios por metade com `prefixlen:`, por isso entram como `/41,41,1` e `/25,25,1` (duas linhas cada).
- **Regra RFC 9632/9977:** o consumidor usa o inet(6)num **mais específico** que referencia o ficheiro e ignora linhas fora do range dele. Cada linha (geofeed e prefixlen) tem de caber dentro desse objeto — se o LIR criar objetos mais pequenos, partir as linhas na mesma granularidade.
- `public/_headers` dá `Cache-Control: public, max-age=604800` ao `/prefixlen.csv` (sec. 7: sinalizar refetch; dados mudam raramente). `_headers` não é servido.

## Publicação do geofeed e prefixlen (fora do repo)

Os atributos `geofeed:` (RFC 9632) e `prefixlen:` (RFC 9977) existem no `inetnum`/`inet6num` da RIPE DB (opcionais, um de cada); **não** no `aut-num`. Os consumidores usam o inet(6)num **mais específico** com referência. Alterações exigem o maintainer do objeto (os LIR), salvo onde houver `mnt-lower: IGBZU-mnt`.

Para consultar, incluir o objeto exato e os mais específicos — **não** usar só `-l` (devolve apenas o menos específico e esconde o exato):
`curl -s "https://rest.db.ripe.net/search.json?query-string=<prefixo>&type-filter=inet6num&flags=r&flags=B&flags=M"` (repetir sem `flags=M` para o exato).

Estado em 2026-10-26 (maintainer do utilizador: `IGBZU-mnt`; org: `ORG-IBZ2-RIPE`):

- `2001:678:13a0::/48` (`ASSIGNED PI`, PT+BR) — `RIPE-NCC-END-MNT` + `IGBZU-mnt` (do utilizador); geofeed e prefixlen nossos. ✔ ROA (AS213367, maxLength 48) e route6 (`mnt-by: IGBZU-mnt`) criados pelo utilizador em 2026-10-22.
- `2a0c:9a40:e180::/48` (`ASSIGNED`, PT+BR) — iFog (`ch-ifog-1-mnt`) + `IGBZU-mnt`; geofeed e prefixlen nossos. ✔
- `2a0f:6280:1f00::/40` (`ALLOCATED-BY-LIR`) + `/41` PT e `/41` BR (`ASSIGNED`) — HYEHOST-MNT; geofeed e prefixlen nossos nos três. ✔
- `2a0f:6280:1057::/48` (`ASSIGNED`, PT) — HYEHOST-MNT; geofeed e prefixlen nossos. ✔ `country:` só PT porque o painel da HYEHOST não aceita mais de um país — aceite, ignorar o aviso. rDNS delegado pela HYEHOST para a Cloudflare.
- `2a06:b700:1003::/48` (`ASSIGNED`, PT, netname `Bissoli`, sem org) — miraliumre-mnt; **sem** geofeed nem prefixlen. **Em descontinuação**: o utilizador migrou para o `2a0c:9a40:e180::/48` (iFog); o bloco termina a **2026-12-31** (lembrete agendado para 2026-11-30). Não pedir geofeed à Miralium; retirar as linhas do geofeed/prefixlen quando deixar de ser anunciado.
- `5.231.99.0/24` (a sair ~2026-11-10) — GHOSTnet criou 256 inetnum /32 (`.0–.127` PT, `.128–.255` BR, org `ORG-GG3-RIPE`); geofeed nosso, **sem** prefixlen.
- `187.7.200.0/24` (`SUB-ALLOCATED PA`, org `ORG-IBZ2-RIPE`, `mnt-by: ipbnb-mnt`, `mnt-lower: IGBZU-mnt`, abuse-c ainda `am34346` da IPBNB) + `/25` PT e `/25` BR (`ASSIGNED PA`, `mnt-by: IGBZU-mnt`, criados pelo utilizador em 2026-10-26, `abuse-c: IBN21-RIPE` → abuse@zulcao.com.br); geofeed e prefixlen nossos nos /25. ✔ ROA AS213367 ml24.

## rDNS e /41 (estado 2026-10-26)

| Zona reversa | Delegada por | NS Cloudflare | DNSSEC |
|---|---|---|---|
| `0.a.3.1.8.7.6.0.1.0.0.2.ip6.arpa` (2001:678:13a0::/48) | RIPE NCC (domain `IGBZU-mnt`) | nancy/roan | ✔ DS 2371 |
| `0.8.1.e.0.4.a.9.c.0.a.2.ip6.arpa` (2a0c:9a40:e180::/48) | RIPE NCC (domain `IGBZU-mnt`) | nancy/roan | ✔ DS 2371 |
| `f.1.0.8.2.6.f.0.a.2.ip6.arpa` (/40) | HYEHOST | luciane/tate | ✔ |
| `7.5.0.1.0.8.2.6.f.0.a.2.ip6.arpa` (2a0f:6280:1057::/48) | HYEHOST | nancy/roan | ✔ |
| `200.7.187.in-addr.arpa` | RIPE (domain `ipbnb-mnt`) | dexter/shaz | ✘ — DS fica para um ticket futuro à IPBNB |

- **Validar DNS por DoH** (`https://dns.google/resolve?name=…&type=SOA`, ver `AD`): a rede local do utilizador interceta a porta 53 (respostas com `ra ad` sem `aa`, timeouts a 1.1.1.1/8.8.8.8), por isso `dig` local não é fiável para estas verificações.
- PTR IPv6 na Cloudflare: o nome tem de completar os 32 nibbles do endereço (ex.: `2a0f:6280:1057::1` na zona do /48 → `1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0`). FCrDNS não é necessário por agora (decisão do utilizador).
- `2a0f:6280:1f00::/40`: ROA maxLength 41 (intencional). /40 + os dois /41 **anunciados** desde 2026-10-26 a partir do OPNsense (FRR), RPKI valid, `route6` dos /41 criados pela HYEHOST. Prefix-list de saída `PL-OUT-V6`: `permit 2a0f:6280:1f00::/40 le 41` (no FRR o `ge` tem de ser > comprimento). O FRR tem `bgp network import-check`: cada `network` precisa de rota na RIB → âncoras blackhole (gateway **Null6** em System → Routes do OPNsense, não no vtysh) para o /40 e para cada /41. Quando o sítio BR tiver router próprio, o `/41` BR (`2a0f:6280:1f80::/41`) passa a ser originado lá e sai deste router.

## Repositório

Remote: `git@github.com:zulcao/as213367-ispconfig.git` (público, org `zulcao`; a conta `gh` é `CattleWhisper`).
