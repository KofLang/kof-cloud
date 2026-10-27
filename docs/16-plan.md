# Plano incremental de implementação

Cada fase é validada antes da seguinte. Nada avança com teste vermelho.

## Fase 1 — Túnel persistente

Objetivo: uma máquina estabelece um túnel persistente com o KofCloud.

| Passo | Entrega | Prova |
|---|---|---|
| 1.1 | `protocol/` — enquadramento, frame, tunnel protocol | U1 verdes (`protocol_test.kf`) |
| 1.2 | `identity/` — par X25519, session key, tokens | U3 verdes (`identity_test.kf`) |
| 1.3 | `model/` + `store/` | U5 verdes (`registry_test.kf`) |
| 1.4 | `control/` — control plane mínimo (conta, device, connect) | E4 verde (`control_test.kf`) |
| 1.5 | `edge/TunnelListener` + `TunnelSession` | handshake em `protocol_test.kf` + E1 |
| 1.6 | `agent/Agent` — connect, heartbeat, reconexão | F1, F2 verdes (backoff U4 + E11 `idle-e2e`) |

Resultado: `HELLO`/`WELCOME` estabelecidos, `PING`/`PONG` mantendo a sessão,
reconexão com backoff.

## Fase 2 — Publicação HTTP

Objetivo: `kof cloud publish site --port 8080` → `http://site.<handle>.kofcloud.io`.

| Passo | Entrega | Prova |
|---|---|---|
| 2.1 | `edge/Router` + `Streams` | U2 verdes + E8/E12 (multiplexacao real) |
| 2.2 | `edge/HttpFront` — listener HTTP/1.1 | E1 verde |
| 2.3 | `agent/Relay` — stream ↔ localhost | E1 verde |
| 2.4 | `agent/Health` + `STATUS` no Edge | F4 verde (E4 + `hosts_test.kf`) |
| 2.5 | CLI: `login`, `connect`, `publish`, `unpublish`, `service list` | E1 verde |

Resultado: aplicação local acessível publicamente pelo Edge.

## Fase 3 — DNS + HTTPS

Objetivo: `https://site.mel.kofcloud.io` sem configuração manual.

| Passo | Entrega | Prova |
|---|---|---|
| 3.1 | hostname automático no `publish` | U2 verdes (`hosts_test.kf`) |
| 3.2 | roteamento por `Host` com wildcard por conta | E1, F8 verdes |
| 3.3 | TLS do Edge via `web.app().listenSecure` (JVM) | teste manual documentado |
| 3.4 | emissão/renovação externa de certificado (wildcard) | documentado em `12-tls.md` |

Resultado: HTTPS por hostname, sem certificado manual do usuário.

## Fase 4 — múltiplos serviços

Um Agent, N hostnames, N portas locais, todos no mesmo túnel. Ja coberto
no E1/E2; a fase formaliza os limites por dispositivo.

**Estado: ✅ implementado.** Limites aplicados: `kofcloud.limits.devices`
(dispositivos ativos por conta), `kofcloud.limits.services` (por dispositivo) e
`kofcloud.limits.streams` (por túnel, com `429`/`RESET LIMIT_EXCEEDED`).
Prova: `tests/limits_test.kf` (6 testes) + `tests/e2e_multiservice_test.kf`
(3 serviços no mesmo túnel, com isolamento de falha).

## Fase 5 — autenticação

`PUBLIC` / `AUTHENTICATED` / `PRIVATE` no modelo e no Edge. `PRIVATE` já existe
desde a Fase 2 (sem rota); `AUTHENTICATED` adiciona a checagem de token antes do
túnel.

**Estado: ✅ implementado.** Credencial de serviço `kfs_` emitida pelo control
plane (`POST /v1/services/token`) e validada no Edge; o token da conta não
autoriza acesso público; o Edge remove a credencial antes de repassar ao
serviço local; `401` com `WWW-Authenticate: Bearer`. CLI: `kof cloud service
token <nome>` (revogacao: `kof cloud service token-revoke <nome> <kfs_>` → `POST /v1/services/token/revoke`, 404 opaco se alheio; inventario: `kof cloud service token-list <nome>` → `POST /v1/services/tokens`, so id curto + emissao, nunca o valor; expiracao opcional `--ttl <horas>`, expirado vira 401 e some do inventario). Prova: `tests/service_token_test.kf` (11 testes) +
`tests/e2e_authenticated_test.kf`.

**Endurecimento (padrão kof-share R6): ✅ implementado.** Erros estáveis
`KCF001`–`KCF106` (`model/Errors.kf`): nunca removidos nem renumerados; 404
opaco entre contas (existente-alheio indistinguível de inexistente);
owner-gate em `POST /v1/services/status` (só a conta dona altera); CLI com
`Hint:` humana (`printFail`); validação de `publish` devolve o HTTP certo por
família (400 validação, 404 dono/recurso, 429 limite). Prova:
`tests/errors_test.kf` (5 testes: códigos do control plane, 404 opaco,
limites/owner-gate no registry, códigos do Edge, Hint).

## Fase 6 — domínio próprio

**Estado (slice 1): ✅ implementado.** Registro PENDING com instrução CNAME
(`POST /v1/domains` → `CNAME api.example.com -> site.mel.kofcloud.io`),
verificação por `net.resolve` contra os destinos esperados do Edge (config
`kofcloud.edgeHosts`, comparação exata, NXDOMAIN/falha vira `KCF124`
indistinguível de "resolveu para outro lugar"), rota ativa só depois de
VERIFIED, poda junto com serviço/dispositivo. Dono verificado antes do DNS
(sem trabalho de rede para conta alheia). CLI: `domain add --service`,
`domain verify`, `domain remove`; erros `KCF120`–`KCF124` com `Hint:`.
Prova: `tests/domains_test.kf` (7 testes: validação, registro/idempotência,
verificação sem rede, rota/poda, API, Edge via túnel real, Hint).

Falta (fatiado de propósito): TLS por SNI no Edge (o Kof não expõe — ver
`12-tls.md`) e automação de DNS por provedor (Cloudflare/Route53). Sem isso,
o hostname próprio serve HTTP pela borda; o HTTPS continua dependendo de
terminador externo.

## Fase 7 — deploy

**Estado (slice 1 — deploy declarativo): ✅ implementado.** `kofcloud.json`
(`--file` para outro caminho) declara `services[]` (nome, porta numero ou
string, address/exposure opcionais); `kof cloud deploy` publica em lote
(idempotente, sem remover o que nao declarou) e `kof cloud down` remove so
os do manifesto. Sem build nem artefato — so `publish` em lote sobre as
primitivas existentes. Parser manual com erro posicional
(Map<String, String> nao aceita numero — ClassCast medido). Prova:
`tests/deploy_test.kf` (5 testes). `deploy --check` verifica publicado +
respondendo (sem rollback).

**Estado (slice 3 — daemon user-level): ✅ implementado.** `kof cloud daemon
install|status|uninstall`: gera `kofcloud-agent.service` (mesmo `connect`,
`Restart=always`, `NoNewPrivileges`, enable manual de proposito), erro
honesto sem systemd. So Linux; Windows/macOS na Fase 8. Prova:
`tests/daemon_test.kf` (3 testes).

**Estado (hot-reload — debito quitado): ✅ implementado.** O Agent re-busca
`services[]` via `/v1/connect` a cada `6×healthMs` e troca a tabela DA SESSAO
(objeto compartilhado — Map nao atravessa `spawn` por valor); streams em voo
nao sao tocados; remocao sai da tabela. Junto, handshake por stream: o Edge
espera OPENED antes de DATA, e OPEN recusado vira 502 em vez de travar o
relay para sempre (buraco real: publicar-depois-do-connect pendurava a vaga).
Prova: `tests/reload_test.kf` + OPENED em `tests/protocol_test.kf`.

**Estado (presenca real do tunel): ✅ implementado.** O Edge espelha
bind/unbind num mapa do Registry (nao persistido); `GET /v1/presence`
(autenticado, filtrado por conta) lista devices com sessao viva; `kof cloud
status` mostra `Tunel: ONLINE/OFFLINE` confirmado pelo Edge (nao inferido do
processo local) e o `doctor` ganhou a camada do tunel. Prova:
`tests/presence_test.kf` (vivo listado, morto some, 401 sem token, filtro
por conta).

Falta fatiado (Fase 7): build/artefato, auto-update. `kof cloud doctor` (somente-leitura: login, control plane,
device, servicos do manifesto) responde "onde quebrou" sem alterar nada.

## Débitos de limites (antes do deploy)

**Estado (frame DATA fatiado + teto de linha): ✅ implementado.** Envio em
blocos de 16 KiB (`kofcloud.limits.sendChunkBytes`) já codificados, um frame
DATA por bloco; recepção recusa linha acima de `maxTunnelLineBytes()`
(2×frameBytes + margem, fail-closed com `LINE_OVERFLOW` no Edge). Chunk
adaptativo (encolhe até caber — o base64 do stdlib passa por UTF-8 e expande
bytes altos). Prova: `tests/frames_test.kf` (4 testes: fatiamento +
recomposição byte-idêntica, codec fail-closed, HELLO gigante fecha sem
WELCOME, 1.5 MiB fim a fim).

**Estado (idle timeout): ✅ implementado.** `lastActivityMs` por sessão
(renovado só por frame útil — PING/PONG não contam) + varredura a cada 5 s
nos dois lados: túnel ocioso cai (`kofcloud.limits.tunnelIdleMs`, default
120 s; métrica `_idle_timeout` no Edge) e stream preso leva `RESET
IDLE_TIMEOUT` (`kofcloud.limits.streamIdleMs`, default 30 s) sem derrubar a
sessão. EOF limpo virou throw (reconecta com backoff), exceto com stopFile
ou GOAWAY — antes, queda silenciosa parava o Agent. Prova:
`tests/idle_test.kf` (4 testes).

**Estado (conexões simultâneas por serviço — fecha os débitos): ✅
implementado.** Contador por `serviceId` no Edge
(`kofcloud.limits.serviceConns`, default 64); excedeu → `429 KCF108` +
`Retry-After` antes do OPEN (sem consumir stream nem tocar no Agent).
Aproximado sob corrida (documentado); métrica `_service_overloaded`.
Prova: `tests/conns_test.kf` (2 testes: concorrência real com servidor
lento, liberação, isolamento, métrica, Hint).

## Abuse prevention (parcial, antes do deploy)

**Estado (slice 1 — rate limit por hostname): ✅ implementado.**
`security.rateLimit` (fixed-window, medido) fiado no Edge público:
`kofcloud.limits.httpRequests` por hostname e janela
(`kofcloud.limits.httpWindowMs`, defaults 100/60 s); excedeu → `429 KCF107`
+ `Retry-After`, **antes** de rotear (não revela se o host existe, não
consome túnel). Chave = hostname (o Edge TCP não expõe IP de origem).
Métrica `_rate_limited`, `Hint:` sem dado de cliente. Prova:
`tests/ratelimit_test.kf` (2 testes: 4o pedido → 429 com/sem túnel, cota
isolada por hostname, Retry-After, métrica, Hint).

**Estado (slice 2 — suspensão + auditoria): ✅ implementado.**
`SUSPENDED` reversível para serviço (sai da rota, opaco como PRIVATE) e
dispositivo (túnel recusado com `DEVICE_SUSPENDED`, fatal — sem reconexão);
health do Agent e status via API **não** dessuspendem (só `unsuspend`
libera); enforcement imediato no Edge (request derruba a sessão) e contínuo
via PING (cai em até um heartbeat); trilha de auditoria em anel (100
eventos, só metadados) com `GET /v1/audit` filtrado por conta + `kof cloud
audit`. Prova: `tests/abuse_test.kf` (3 testes: rota/health/API, device no
túnel, Edge fim a fim + audit via HTTP).

## Observabilidade mínima (parcial, antes do deploy)

**Estado (slice 1): ✅ implementado.** `edge/Metrics.kf` conta via
`observability.*`: túneis aceitos/recusados por motivo, sessões abertas
(gauge), requisições ok/negadas por motivo, erros de upstream, bytes
somados, streams e latência (histograma). Sem cardinalidade por cliente:
nenhum hostname, corpo, credencial ou chave chega às métricas.

**Estado (slice 2): ✅ implementado.** `control/Metrics.kf` conta lifecycle
no Registry (contas, logins, devices, serviços, tokens, domínios) e
`GET /v1/metrics` serve o export Prometheus sem auth (igual ao `/v1/health`).
Prova: `tests/metrics_test.kf` (6 testes: formato, recusa por motivo, Edge
fim a fim, privacidade, endpoint + contadores via HTTP e via registry).

**Estado (slice 3 — fecha): ✅ implementado.** `agent/Metrics.kf` conta
connects, reconnects (backoff), recusas fatais por motivo, health reports,
streams servidos e erros locais — agregados, sem nome de serviço, endereço,
porta, conteúdo, token ou chave. O Agent roda em processo separado, então
seus contadores vivem no próprio processo (sem protocolo novo no MVP).
Prova: `tests/agent_metrics_test.kf` (3 testes: contadores puros, fluxo
real via túnel, recusa fatal por motivo).

## Fase 8 — produto multiplataforma

Linux → Windows → macOS → Android. O core é compartilhado; só daemon,
filesystem, processos e UI variam. Bloqueado por lacunas do Kof em
native/JS (`13-kof-apis.md`).

**Pagos fora de fase (débitos que viraram feature):** hot-reload de serviço
(refresher via `/v1/connect` + handshake OPENED/502 — sem ele, publicar
depois do `connect` pendurava o relay para sempre). Prova:
`tests/reload_test.kf` + OPENED em `tests/protocol_test.kf`. Lição registrada:
Map nao atravessa `spawn` (usar objeto compartilhado) — ver `STATUS.md`.

## Fase 9 — TLS/HTTPS na borda (cloud robusta)

**Motivação:** hoje o Edge fala HTTP/1.1 em claro — o `kfs_` viaja legível
entre cliente e Edge. Cloud robusta de verdade exige HTTPS ponta a ponta
(cliente→Edge) + túnel cifrado (Edge→Agent, já existe com
`KOFCLOUD_TUNNEL_SECRET`).

**Estado: ⏳ não iniciado.** Desenho fatiado (cada slice é um incremento
testável, na ordem):

* **Slice 1 — Edge HTTPS com PEM do operador.** `serve` ganha
  `--tls-cert/--tls-key`; sem flags, HTTP (comportamento atual + aviso);
  com flags, `web.app().listenSecure` (JVM) no lugar do listener cru.
  Roteamento por `Host` e relay inalterados (o núcleo trabalha com bytes
  já decifrados). Prova: e2e HTTPS com cert autoassinado gerado no teste.
* **Slice 2 — HSTS + redirecionamento HTTP→HTTPS.** Resposta 301 para
  `http://` quando o TLS está ativo + header `Strict-Transport-Security`.
  Sem flag nova (segue o modo do slice 1).
* **Slice 3 — SNI por hostname + renovação assistida.** Um PEM por
  hostname próprio VERIFIED (mapa em memória, recarregado sem derrubar
  o Edge); `domain verify` passa a aceitar o PEM; expiração visível no
  `service status`/`doctor` (aviso a N dias, nunca bloqueio silencioso).
  ACME de verdade (DNS-01) depende de ES256 no stdlib — sem isso, emissão
  continua externa (ver `12-tls.md`).

**Bloqueios honestos:** sem TLS em socket cru (L2) — por isso `listenSecure`
(JVM-only); sem ACME e sem ES256 (só Ed25519) — por isso sem Let's Encrypt
automático ainda. Ver `12-tls.md` e `13-kof-apis.md` L2.

## Fase 10 — WAF + proteção da aplicação (cloud robusta)

**Motivação:** hoje o Edge roteia sem inspecionar — SQLi/XSS/RCE passam
direto para o serviço local. Cloud robusta filtra o óbvio na borda sem
quebrar o legítimo (fail-open com log vs fail-closed configurável).

**Estado: ⏳ não iniciado.** Desenho fatiado:

* **Slice 1 — regras básicas por padrão (modo detectar).** Assinaturas
  mínimas em método/path/query/corpo pequeno (traversal `../`, null byte,
  `UNION SELECT`/`OR 1=1` canônicos, `<script` em query): casa → `403 KCF109`
  + métrica `_waf_denied` + log só com padrão+hostname (nunca corpo).
  Default **detectar** (conta + header `X-KofCloud-WAF: would-block`);
  `--waf=enforce` bloqueia. Prova: unit das assinaturas + e2e (ataque 403,
  legítimo 200).
* **Slice 2 — limites de corpo/tempo por rota.** Teto de request body por
  serviço (`kofcloud.limits.httpBodyBytes`, default 1 MiB; excedeu →
  `413 KCF110`) + timeout de upstream (502 após N s, sem pendurar o stream).
* **Slice 3 — allowlist por serviço.** `exposure: AUTHENTICATED` ganha
  `allowMethods: [GET, POST]` e `allowPaths: ["/api/*"]` opcionais no
  manifesto/API; fora da lista → `403 KCF111`. Fail-closed por desenho.

**Regra de ouro:** WAF nunca vaza conteúdo em log/métrica (só padrão que
casou + hostname) e nunca quebra o legítimo por default (slice 1 começa
em detectar).

## Fase 11 — criptografia pós-quântica (cloud robusta)

**Motivação:** X25519 + AES-GCM são clássicos — um adversário que grave o
tráfego hoje quebra amanhã ("harvest now, decrypt later"). Cloud robusta
de verdade oferece KEM pós-quântico no túnel sem regredir quem não tem.

**Estado: ⏳ não iniciado. Bloqueado no stdlib.** Desenho (para quando o
Kof expuser o primitivo):

* **Slice 1 — inventário + agilidade.** `protocol/Version.kf` negocia
  `kex: x25519 | ml-kem-512-x25519`; handshake anuncia capacidades e cai
  para X25519 puro quando um lado não tem PQC (interop primeiro).
* **Slice 2 — híbrido ML-KEM+X25519.** `session key = HKDF(kem_secret ||
  xof_shared)`; AES-GCM continua (é o KEM que troca, não a cifra).
  Prova: e2e com túnel híbrido + fallback clássico.
* **Slice 3 — assinatura PQC no control plane.** Trocar Ed25519 por
  ML-DSA onde hoje há assinatura de longo prazo (tokens continuam hash
  opaco — não precisam migrar).

**Bloqueio honesto:** o stdlib do Kof hoje expõe X25519, HKDF, AES-GCM e
Ed25519 — **sem ML-KEM/ML-DSA**. Sem o primitivo não há slice que ande;
esta fase existe no plano como compromisso público, não como promessa.
Ver `13-kof-apis.md` (tabela de cripto).

## Ordem de execução no MVP

```
Fase 1 → Fase 2 → Fase 3
```

validando cada etapa. Nada de Fase 4+ antes de a Fase 3 estar estável.
