# Plano incremental de implementação

Cada fase é validada antes da seguinte. Nada avança com teste vermelho.

## Fase 1 — Túnel persistente

Objetivo: uma máquina estabelece um túnel persistente com o KofCloud.

| Passo | Entrega | Prova |
|---|---|---|
| 1.1 | `protocol/` — enquadramento, frame, tunnel protocol | U1–U3 verdes |
| 1.2 | `identity/` — par X25519, session key, tokens | U7–U8 verdes |
| 1.3 | `model/` + `store/` | U9, U11 verdes |
| 1.4 | `control/` — control plane mínimo (conta, device, connect) | I1 verde |
| 1.5 | `edge/TunnelListener` + `TunnelSession` | I2 verde |
| 1.6 | `agent/Agent` — connect, heartbeat, reconexão | F1, F2 verdes |

Resultado: `HELLO`/`WELCOME` estabelecidos, `PING`/`PONG` mantendo a sessão,
reconexão com backoff.

## Fase 2 — Publicação HTTP

Objetivo: `kof cloud publish site --port 8080` → `http://site.<handle>.kofcloud.io`.

| Passo | Entrega | Prova |
|---|---|---|
| 2.1 | `edge/Router` + `Streams` | U5, U6, I4 verdes |
| 2.2 | `edge/HttpFront` — listener HTTP/1.1 | I3 verde |
| 2.3 | `agent/Relay` — stream ↔ localhost | I3 verde |
| 2.4 | `agent/Health` + `STATUS` no Edge | F4 verde |
| 2.5 | CLI: `login`, `connect`, `publish`, `unpublish`, `service list` | E1 verde |

Resultado: aplicação local acessível publicamente pelo Edge.

## Fase 3 — DNS + HTTPS

Objetivo: `https://site.mel.kofcloud.io` sem configuração manual.

| Passo | Entrega | Prova |
|---|---|---|
| 3.1 | hostname automático no `publish` | U5 verde |
| 3.2 | roteamento por `Host` com wildcard por conta | I3, F8 verdes |
| 3.3 | TLS do Edge via `web.app().listenSecure` (JVM) | teste manual documentado |
| 3.4 | emissão/renovação externa de certificado (wildcard) | documentado em `12-tls.md` |

Resultado: HTTPS por hostname, sem certificado manual do usuário.

## Fase 4 — múltiplos serviços

Um Agent, N hostnames, N portas locais, todos no mesmo túnel. Já coberto por
I4/F8 na Fase 2–3; a fase formaliza os limites por dispositivo.

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

CNAME + validação por `net.resolve` + ativação da rota. Depende de TLS por SNI,
que o Kof ainda não expõe — documentado.

## Fase 7 — deploy

`kof cloud deploy` sobre as mesmas primitivas (identity + device + tunnel +
service). Não cria infraestrutura paralela.

## Fase 8 — produto multiplataforma

Linux → Windows → macOS → Android. O core é compartilhado; só daemon,
filesystem, processos e UI variam. Bloqueado por lacunas do Kof em
native/JS (`13-kof-apis.md`).

## Ordem de execução no MVP

```
Fase 1 → Fase 2 → Fase 3
```

validando cada etapa. Nada de Fase 4+ antes de a Fase 3 estar estável.
