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
token <nome>`. Prova: `tests/service_token_test.kf` (7 testes) +
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
`tests/daemon_test.kf` (3 testes). Falta fatiado: build/artefato,
auto-update.

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

## Fase 8 — produto multiplataforma

Linux → Windows → macOS → Android. O core é compartilhado; só daemon,
filesystem, processos e UI variam. Bloqueado por lacunas do Kof em
native/JS (`13-kof-apis.md`).

## Ordem de execução no MVP

```
Fase 1 → Fase 2 → Fase 3
```

validando cada etapa. Nada de Fase 4+ antes de a Fase 3 estar estável.
