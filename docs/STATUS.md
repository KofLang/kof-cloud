# STATUS — estado medido do KofCloud

 retrato honesto em 2026-10-24: o que esta verde, como provar, o que falta.
 Nada aqui e aspiracional — cada linha tem prova executavel.

## Estado

| Area | Estado | Prova |
|---|---|---|
| Fases 1–3 (tunel, publish, hostname) | ✅ verde | `e2e_test.kf` |
| Fase 4 (multi-servico + limites) | ✅ verde | `e2e_multiservice_test.kf`, `limits_test.kf` |
| Fase 5 (AUTHENTICATED kfs_ + revoke + token-list + TTL) | ✅ verde | `service_token_test.kf` (11), `e2e_authenticated_test.kf` |
| Fase 6 slice 1 (dominio proprio) | ✅ verde | `domains_test.kf` (sem TLS/SNI — externo) |
| Observabilidade (3 slices) | ✅ verde | `metrics_test.kf`, `agent_metrics_test.kf` |
| Abuse prevention (2 slices) | ✅ verde | `ratelimit_test.kf`, `abuse_test.kf` |
| Debitos de limites (frame, idle, conns) | ✅ verde | `frames_test.kf`, `idle_test.kf`, `conns_test.kf` |
| Fase 7 slices 1–4 (deploy, --check, daemon, doctor) | ✅ verde | `deploy_test.kf`, `daemon_test.kf`, `doctor_test.kf` |
| Presença real do túnel | ✅ verde | `presence_test.kf` (1), `hosts_test.kf` (bodyContainsId) |
| Hot-reload + handshake por stream | ✅ verde (debito quitado) | `reload_test.kf`, `protocol_test.kf` |
| Fase 7 slice 5+ (build/artefato) | ⏳ nao iniciado | — |
| TLS/SNI no Edge, ACME | ⏳ externo por desenho | `12-tls.md` |

## Como provar (uma sessao)

```bash
export PATH=/home/mel/tools/jdk-25/bin:$PATH
KOF=/home/mel/Kof4j/kof-cli/target/kof-cli-0.6.3.jar
java -jar $KOF test --target jvm --timeout 150   # 25 suites, 0 falhas
java -jar $KOF build --output /tmp/kofcloud-check # compila sem erro
```

Toolchain exigido: CLI **0.6.3** + **JDK 25** (o `kof` 0.5.0-beta do PATH nao
entende as duas raizes de `kof.toml`; o java 21 nao abre o CLI novo).
Ver `13-kof-apis.md` e `14-modules.md`.

## Marcadores por area (goldens)

Cada suite imprime `PASS <nome>` por teste e `0 failed of N tests` no fim;
o run total imprime `25 passed, 0 failed`. Falha = nome do teste + assert
+ abort sem cleanup (listeners presos ate o `--timeout` matar — ver
`15-tests.md` sobre sentinelas).

| Suite | Arquivo | Testes |
|---|---|---|
| framing/bytes/protocolo/sessao/OPENED | `tests/protocol_test.kf` | 11 |
| hostnames/exposicao/texto/log/loopback | `tests/hosts_test.kf` | 11 |
| identidade/cifra/tokens | `tests/identity_test.kf` | 7 |
| backoff | `tests/backoff_test.kf` | 3 |
| registry | `tests/registry_test.kf` | 7 |
| limites | `tests/limits_test.kf` | 6 |
| service token | `tests/service_token_test.kf` | 11 |
| erros R6 | `tests/errors_test.kf` | 7 |
| dominios | `tests/domains_test.kf` | 7 |
| metricas | `tests/metrics_test.kf` | 6 |
| rate limit | `tests/ratelimit_test.kf` | 2 |
| agent metrics | `tests/agent_metrics_test.kf` | 3 |
| abuso | `tests/abuse_test.kf` | 3 |
| frames | `tests/frames_test.kf` | 4 |
| idle | `tests/idle_test.kf` | 4 |
| conexoes | `tests/conns_test.kf` | 2 |
| deploy | `tests/deploy_test.kf` | 5 |
| daemon | `tests/daemon_test.kf` | 5 |
| doctor | `tests/doctor_test.kf` | 5 |
| hot-reload | `tests/reload_test.kf` | 1 |
| control plane | `tests/control_test.kf` | 2 |
| presenca | `tests/presence_test.kf` | 1 |
| e2e | `tests/e2e_test.kf` | 1 |
| e2e multiservico | `tests/e2e_multiservice_test.kf` | 1 |
| e2e autenticado | `tests/e2e_authenticated_test.kf` | 1 |

Total: 116 testes. Ver `15-tests.md` para a matriz.

## Limites honestos (medidos, nao prometidos)

- Rate limit e por processo; sem consenso entre nos.
- Contador por servico e aproximado sob corrida (sem lock no Kof).
- `bytesToString` e O(n2): payloads grandes sao lentos (fatiado em 16 KiB).
- `Map<String, Int>` como campo quebra o verificador JVM (usar String).
- `await` com valor de retorno quebra o verificador JVM.
- Map passado como argumento de `spawn` e copiado por valor: estado
  compartilhado entre tasks vive em objeto (classe), nao em Map solto.
- `web.app()` fecha conexao para corpo de ~1.5 MiB (testes usam TCP cru).
- Processo fantasma de run travado prende portas (kill -9).

Ver `17-limitations.md` para a lista completa e `13-kof-apis.md` para as
medicoes do runtime.
