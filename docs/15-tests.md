# Matriz de testes

Comando: `kof test --target jvm --timeout 120` a partir da raiz do repositório
(as raízes app/teste vêm de `kof.toml [sources]` — ver `14-modules.md`).

`--timeout` é obrigatório na prática: o E2E sobe listeners TCP cujo `accept()`
bloqueado só cede quando o listener é fechado; o teste encerra o Agent (arquivo
sentinela) e os listeners (sentinela) para que o processo saia sozinho — sem
isso o runner esperaria para sempre (`--timeout 0` = sem limite é o default).

## Unitários

| # | Alvo | Arquivo | O que prova |
|---|---|---|---|
| U1 | enquadramento | `tests/protocol_framing_test.kf` | split por `\n`, buffer parcial, frame vazio, linha corrompida |
| U2 | bytes | `tests/protocol_bytes_test.kf` | base64 ida-e-volta, byte assinado (+256), limite de frame |
| U3 | tunnel protocol | `tests/protocol_tunnel_test.kf` | encode/decode de cada `t`, campos obrigatórios, tipo desconhecido |
| U4 | control protocol | `tests/protocol_control_test.kf` | rotas, payloads, erros |
| U5 | hostnames | `tests/hosts_test.kf` | `<name>.<handle>.kofcloud.io`, validação de handle, host desconhecido |
| U6 | exposição | `tests/exposure_test.kf` | `PRIVATE` não gera rota; `PUBLIC`/`AUTHENTICATED` geram |
| U7 | identidade | `tests/identity_test.kf` | par X25519, session key simétrica, HKDF determinístico |
| U8 | tokens | `tests/tokens_test.kf` | formato, comparação segura, expiração |
| U9 | modelo de serviço | `tests/service_test.kf` | estado de health, transições |
| U10 | backoff | `tests/backoff_test.kf` | crescimento, teto, jitter no intervalo |
| U11 | store | `tests/store_test.kf` | round-trip JSON em disco, arquivo ausente |
| U12 | cifra do túnel | `tests/identity_test.kf` | AES-GCM com a chave derivada, payload adulterado falha |
| U13 | limites | `tests/limits_test.kf` | dispositivos ativos, serviços por dispositivo, streams por túnel, liberação de vaga |
| U14 | credencial de serviço | `tests/service_token_test.kf` | emissão só pela conta dona, token vale apenas para o próprio serviço, poda ao remover serviço/revogar dispositivo, persistência |
| U15 | erros estáveis (R6) | `tests/errors_test.kf` | códigos `KCFnnn` no control plane e no Edge, 404 opaco entre contas, owner-gate do status, `Hint:` sem vazar segredo |
| U16 | domínio próprio | `tests/domains_test.kf` | validação de hostname, registro/idempotência, verificação sem rede, rota só após VERIFIED + poda, API (409/404 opacos), Edge via túnel real, `Hint:` |
| U17 | métricas do Edge | `tests/metrics_test.kf` | nomes/formato Prometheus, recusa de túnel por motivo, contagem ok/negada + bytes fim a fim, exposição sem dado de cliente |
| U18 | métricas do control plane | `tests/metrics_test.kf` | `GET /v1/metrics` sem auth, contadores de lifecycle via HTTP e via registry, sem email/hostname no export |
| U19 | rate limit por hostname | `tests/ratelimit_test.kf` | 4º pedido rápido → 429 `KCF107` + `Retry-After`, cota por hostname isolada, com e sem túnel, métrica `_rate_limited`, `Hint:` |

## Integração

| # | Alvo | Arquivo | O que prova |
|---|---|---|---|
| I1 | Agent ↔ Control Plane | `tests/integration_control_test.kf` | criar conta, registrar device, publicar, listar, revogar |
| I2 | Agent ↔ Edge | `tests/integration_tunnel_test.kf` | HELLO/WELCOME, STATUS, PING/PONG, GOAWAY |
| I3 | Edge ↔ Agent ↔ serviço local | `tests/integration_relay_test.kf` | requisição pública chega ao `localhost` e volta |
| I4 | multiplexação | `tests/integration_mux_test.kf` | dois streams concorrentes no mesmo túnel, sem misturar |

## E2E

| # | Fluxo | Arquivo | O que prova |
|---|---|---|---|
| E1 | ponta a ponta | `tests/e2e_test.kf` | conta → device → connect → publish → HTTP → túnel → localhost → resposta |
| E2 | múltiplos serviços | `tests/e2e_multiservice_test.kf` | 3 hostnames no mesmo túnel, cada um na sua porta; derrubar um não afeta os outros |
| E3 | AUTHENTICATED | `tests/e2e_authenticated_test.kf` | 401 sem credencial (com `WWW-Authenticate`), 401 com token inválido, 401 com token da conta, 200 com `kfs_` do serviço; PUBLIC segue livre |

O E2E sobe, em processos/tasks separados: control plane, edge, um serviço local
(`web.app()`), e o agent. Faz uma requisição HTTP real pelo Edge e confere o
corpo.

## Cenários de falha

| # | Cenário | Como é testado |
|---|---|---|
| F1 | internet cai / túnel interrompe | fechar o `Conn` do túnel e conferir que o Agent entra em backoff e reconecta |
| F2 | IP muda | simulado por reconexão a partir de um socket novo |
| F3 | Agent reiniciado | recriar o Agent e conferir que os serviços locais **não** são reiniciados |
| F4 | serviço local indisponível | derrubar o serviço → `STATUS OFFLINE` → Edge devolve `503` |
| F5 | certificado expirado | fora do núcleo (TLS externo) — documentado em `12-tls.md` |
| F6 | dispositivo revogado | `device revoke` → `WELCOME{ok:false, DEVICE_REVOKED}` → Agent não reconecta |
| F7 | serviço removido | `unpublish` → rota removida → `404` |
| F8 | múltiplos serviços | 3 hostnames no mesmo túnel, cada um para sua porta |
| F9 | conexões simultâneas | 16 requisições concorrentes; todas respondem corretamente |
| F10 | frame acima do limite | `RESET{LIMIT_EXCEEDED}` sem derrubar o túnel |

## Critério de sucesso do MVP

```
kof cloud login
kof cloud connect
kof cloud publish site --port 8080

# de outro lugar:
curl http://site.mel.kofcloud.io
# → Internet → Edge → túnel → Agent → localhost:8080 → resposta
```

sem port forwarding, sem IP público, sem VPN, sem servidor externo para a
aplicação, sem reverse proxy manual, sem certificado manual.
