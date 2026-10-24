# Matriz de testes

Comando a partir da raiz do repositório (as raízes app/teste vêm de `kof.toml
[sources]` — ver `14-modules.md`):

```bash
kof test --target jvm --timeout 150
```

Toolchain exigido (ver `13-kof-apis.md`): CLI do Kof **0.6.3** (o `kof`
0.5.0-beta do PATH não entende as duas raízes de `kof.toml` — usar o jar
compilado em `/home/mel/Kof4j/kof-cli/target/kof-cli-0.6.3.jar`) com
**JDK 25** (`JAVA_HOME=/home/mel/tools/jdk-25`; o java 21 do sistema não
abre as classes do CLI novo — `UnsupportedClassVersionError`).

`--timeout` é obrigatório na prática: o E2E sobe listeners TCP cujo `accept()`
bloqueado só cede quando o listener é fechado; o teste encerra o Agent (arquivo
sentinela) e os listeners (sentinela) para que o processo saia sozinho — sem
isso o runner esperaria para sempre (`--timeout 0` = sem limite é o default).

## Unitários (arquivos reais — sem camada `integration_*` separada)

| # | Alvo | Arquivo | O que prova |
|---|---|---|---|
| U1 | enquadramento + bytes + frame + tunnel + sessão | `tests/protocol_test.kf` (10 testes) | split por `\n`, buffer parcial, base64 ida-e-volta, `t` de cada frame, HELLO/WELCOME, streams independentes |
| U2 | hostnames + exposição + texto | `tests/hosts_test.kf` (7 testes) | `<name>.<handle>.kofcloud.io`, handle inválido, `PRIVATE` não roteia, `OFFLINE` não roteia, parse seguro |
| U3 | identidade + cifra + tokens | `tests/identity_test.kf` (7 testes) | par X25519, session key, HKDF, AES-GCM, hash constante, prefixo `kft_` |
| U4 | backoff | `tests/backoff_test.kf` (3 testes) | crescimento, teto, jitter, defaults |
| U5 | registry (conta/device/serviço/store) | `tests/registry_test.kf` (7 testes) | handle duplicado, login, hostname, `PRIVATE` sem rota, revoke poda, persistência |
| U6 | limites | `tests/limits_test.kf` (6 testes) | dispositivos ativos, idempotência, revoke libera vaga, serviços por dispositivo, streams com `setLimits` |
| U7 | credencial de serviço | `tests/service_token_test.kf` (7 testes) | emissão só pela conta dona, token vale apenas para o próprio serviço, poda ao remover serviço/revogar dispositivo, persistência |
| U8 | erros estáveis (R6) | `tests/errors_test.kf` (5 testes) | códigos `KCFnnn` no control plane e no Edge, 404 opaco entre contas, owner-gate do status, `Hint:` sem vazar segredo |
| U9 | domínio próprio | `tests/domains_test.kf` (7 testes) | validação de hostname, registro/idempotência, verificação sem rede, rota só após VERIFIED + poda, API (409/404 opacos), Edge via túnel real, `Hint:` |
| U10 | métricas (Edge + control plane) | `tests/metrics_test.kf` (6 testes) | nomes/formato Prometheus, recusa de túnel por motivo, contagem ok/negada + bytes fim a fim, `GET /v1/metrics` sem auth, contadores de lifecycle, exposição sem dado de cliente |
| U11 | rate limit por hostname | `tests/ratelimit_test.kf` (2 testes) | pedido além do limite → 429 `KCF107` + `Retry-After`, cota por hostname isolada, com e sem túnel, métrica `_rate_limited`, `Hint:` |
| U12 | métricas do Agent | `tests/agent_metrics_test.kf` (3 testes) | contadores sem dado de cliente, fluxo real (connect/stream/health) via túnel, recusa fatal por motivo |
| U13 | suspensão + auditoria | `tests/abuse_test.kf` (3 testes) | SUSPENDED sai da rota (opaco), health/API não dessuspendem, device recusado no túnel, enforcement em request+PING, auditoria filtrada por conta |
| U14 | frame DATA fatiado + teto de linha | `tests/frames_test.kf` (4 testes) | chunk adaptativo (16 KiB), codec com fail-closed, HELLO gigante fecha sem WELCOME, 1.5 MiB fim a fim |
| U15 | idle timeout | `tests/idle_test.kf` (4 testes) | PING não renova, stream preso listado, Edge derruba e Agent reconecta sozinho, RESET sem matar sessão |
| U16 | conexões simultâneas por serviço | `tests/conns_test.kf` (2 testes) | vaga por serviceId (limite 1 no teste), 429 `KCF108` + `Retry-After` com concorrência real, liberação, isolamento entre serviços, métrica `_service_overloaded` |
| U17 | deploy declarativo | `tests/deploy_test.kf` (5 testes) | manifesto válido/inválido com erro posicional, port número/string, fim a fim (publica sem duplicar, down remove só o manifesto), verify acusa/aprova |
| U18 | daemon user-level | `tests/daemon_test.kf` (3 testes) | unit com Restart/NoNewPrivileges/aponta connect, paths user-level, sem systemctl nao quebra |
| U19 | hot-reload + handshake por stream | `tests/reload_test.kf` (1 teste) + `tests/protocol_test.kf` (OPENED aceito/recusado) | servico publicado depois do connect roteia sem reconnect; OPEN recusado responde 502 sem travar o relay |

## Integração

| # | Alvo | Arquivo | O que prova |
|---|---|---|---|
| I1 | Agent ↔ Control Plane | `tests/integration_control_test.kf` | criar conta, registrar device, publicar, listar, revogar |
| I2 | Agent ↔ Edge | `tests/integration_tunnel_test.kf` | HELLO/WELCOME, STATUS, PING/PONG, GOAWAY |
| I3 | Edge ↔ Agent ↔ serviço local | `tests/integration_relay_test.kf` | requisição pública chega ao `localhost` e volta |
| I4 | multiplexação | `tests/integration_mux_test.kf` | dois streams concorrentes no mesmo túnel, sem misturar |

## E2E (processos/tasks separados: control plane, edge, serviço local, agent)

| # | Fluxo | Arquivo | O que prova |
|---|---|---|---|
| E1 | ponta a ponta | `tests/e2e_test.kf` | conta → device → connect → publish → HTTP → túnel → localhost → resposta |
| E2 | múltiplos serviços | `tests/e2e_multiservice_test.kf` | 3 hostnames no mesmo túnel, cada um na sua porta; derrubar um não afeta os outros |
| E3 | AUTHENTICATED | `tests/e2e_authenticated_test.kf` | 401 sem credencial (com `WWW-Authenticate`), 401 com token inválido, 401 com token da conta, 200 com `kfs_` do serviço; PUBLIC segue livre |
| E4 | control plane (HTTP real) | `tests/control_test.kf` (2 testes) | ciclo conta → device → publish → list → revoke + 401 sem token |
| E5 | erros R6 fim a fim | `tests/errors_test.kf` (5 testes, `errors-edge`) | códigos no Edge via túnel real (KCF100/101/102, offline, conteúdo) |
| E6 | domínio próprio fim a fim | `tests/domains_test.kf` (`domains-e2e`) | hostname próprio roteia junto do nativo após VERIFIED |
| E7 | métricas fim a fim | `tests/metrics_test.kf` (`metrics-e2e`) | contagem ok/negada + bytes com túnel real |
| E8 | rate limit fim a fim | `tests/ratelimit_test.kf` (`ratelimit`) | 4º pedido → 429 mesmo com serviço saudável |
| E9 | suspensão fim a fim | `tests/abuse_test.kf` (`abuse-e2e`) | serviço some (404) com túnel vivo; device cai a sessão |
| E10 | frames grandes fim a fim | `tests/frames_test.kf` (`frames-e2e`) | 1.5 MiB fatiado atravessa o túnel |
| E11 | idle + reconexão | `tests/idle_test.kf` (`idle-e2e`) | sweeper derruba, Agent reconecta sozinho, túnel novo serve |
| E12 | concorrência por serviço | `tests/conns_test.kf` (`conns`) | limite 1 com tráfego real simultâneo |
| E13 | deploy fim a fim | `tests/deploy_test.kf` (`deploy-e2e`) | publica sem duplicar, down preserva o extra |

Não há camada `integration_*` separada: os arquivos `control_test.kf`,
`errors_test.kf`, `metrics_test.kf` etc. já sobem HTTP/TCP real (são E2E na
prática). `protocol_test.kf` cobre o handshake HELLO/WELCOME e PING/PONG em
memória.

## Cenários de falha (onde cada um é provado)

| # | Cenário | Como é testado |
|---|---|---|
| F1 | internet cai / túnel interrompe | `idle-e2e`: sweeper derruba → Agent reconecta sozinho (`idle_test.kf`); queda limpa vira throw + backoff |
| F2 | IP muda | reconexão a partir de socket novo (mesmo caminho do F1: HELLO novo, mesma identidade) |
| F3 | Agent reiniciado | `agentMetrics-fatal` + `abuse-e2e`: novo Agent conecta do zero; serviços locais não são tocados |
| F4 | serviço local indisponível | `e2e_multiservice_test.kf` (derruba um, outros seguem) + `hosts_test.kf` (OFFLINE não roteia → `503 KCF103`) |
| F5 | certificado expirado | fora do núcleo (TLS externo) — documentado em `12-tls.md` |
| F6 | dispositivo revogado/suspenso | `service_token_test.kf` (poda) + `abuse_test.kf` (`DEVICE_REVOKED`/`DEVICE_SUSPENDED` fatal, sem reconexão) |
| F7 | serviço removido/suspenso | rota some → `404` opaco (`abuse_test.kf`); `domains_test.kf` (poda) |
| F8 | múltiplos serviços | 3 hostnames no mesmo túnel, cada um para sua porta (`e2e_multiservice_test.kf`) |
| F9 | conexões simultâneas | limite por serviço: 2º pedido → `429 KCF108` (`conns_test.kf`); streams: `RESET LIMIT_EXCEEDED` (`limits_test.kf`) |
| F10 | frame acima do limite | linha gigante → conexão derrubada, sem WELCOME (`frames_test.kf`); `RESET LIMIT_EXCEEDED` (`limits_test.kf`) |

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
