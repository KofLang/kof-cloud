# STATUS — estado medido do KofCloud

 retrato honesto em 2026-11-04: o que esta verde, como provar, o que falta.
 Nada aqui e aspiracional — cada linha tem prova executavel.

## Estado

| Area | Estado | Prova |
|---|---|---|
| Fases 1–3 (tunel, publish, hostname) | ✅ verde | `e2e_test.kf` |
| Health HTTP com prazo (DEGRADED real) | ✅ verde | `health_test.kf` |
| Fase 4 (multi-servico + limites) | ✅ verde | `e2e_multiservice_test.kf`, `limits_test.kf` |
| Fase 5 (AUTHENTICATED kfs_ + revoke + token-list + TTL) | ✅ verde | `service_token_test.kf` (15), `e2e_authenticated_test.kf` |
| Fase 6 slice 1 (dominio proprio) | ✅ verde | `domains_test.kf` (sem TLS/SNI — externo; `domain verify --tls-cert` julga o PEM localmente) |
| Observabilidade (3 slices) | ✅ verde | `metrics_test.kf`, `agent_metrics_test.kf` |
| Enquadramento HTTP no Edge | ✅ verde | `framing_test.kf` (14: Long, 400 KCF118/119/132/133, 1xx, query, hop-by-hop nos dois sentidos, chunked, status sem corpo, CL da resposta, upstream inválido, corpo relayado fatiado) |
| Abuse prevention (2 slices) | ✅ verde | `ratelimit_test.kf`, `abuse_test.kf` |
| Debitos de limites (frame, idle, conns) | ✅ verde | `frames_test.kf`, `idle_test.kf`, `conns_test.kf` |
| Fase 7 slices 1–4 (deploy, --check, daemon, doctor) | ✅ verde | `deploy_test.kf`, `daemon_test.kf`, `doctor_test.kf` |
| Presença real do túnel | ✅ verde | `presence_test.kf` (1), `hosts_test.kf` (bodyContainsId) |
| Hot-reload + handshake por stream | ✅ verde (debito quitado) | `reload_test.kf`, `protocol_test.kf` |
| Fase 7 slice 5 parcial (build/artefato locais) | ✅ verde | `deploy_test.kf` (passos locais: build antes do publish, artifact fail-fast + aviso; sem artefato transferível) |
| Fase 12 slice 1 (painel somente-leitura) | ✅ verde | `panel_test.kf` (3: HTML sem token, serve com `--panel`, 404 sem a flag) |
| TLS/SNI no Edge, ACME | ⏳ externo por desenho | `12-tls.md` |

## Como provar (uma sessao)

```bash
export PATH=/home/mel/tools/jdk-25/bin:$PATH
KOF=/home/mel/Kof4j/kof-cli/target/kof-cli-0.6.3.jar
java -jar $KOF test --target jvm --timeout 150   # 69 suites, 0 falhas
java -jar $KOF build --output /tmp/kofcloud-check # compila sem erro
```

Toolchain exigido: CLI **0.6.3** + **JDK 25** (o `kof` 0.5.0-beta do PATH nao
entende as duas raizes de `kof.toml`; o java 21 nao abre o CLI novo).
Ver `13-kof-apis.md` e `14-modules.md`.

## Marcadores por area (goldens)

Cada suite imprime `PASS <nome>` por teste e `0 failed of N tests` no fim;
o run total imprime `69 passed, 0 failed`. Falha = nome do teste + assert
+ abort sem cleanup (listeners presos ate o `--timeout` matar — ver
`15-tests.md` sobre sentinelas).

| Suite | Arquivo | Testes |
|---|---|---|
| framing/bytes/protocolo/sessao/OPENED | `tests/protocol_test.kf` | 12 |
| hostnames/exposicao/texto/log/loopback/status/publish/ttl/serve/flag/dup/extra/zero/domain-list/custom-for/service-name-match/login-trim/serve-state/serve-waf/serve-tls/audit-sbom/host-normalize/u-matrix/u-counts/prova-counts | `tests/hosts_test.kf` | 33 |
| identidade/cifra/tokens | `tests/identity_test.kf` | 6 |
| forca bruta no login (F5) | `tests/login_limit_test.kf` | 3 |
| vocabulario fechado do status (F5) | `tests/status_report_test.kf` | 2 |
| backoff + drift de Edge + retry offline | `tests/backoff_test.kf` | 5 |
| registry | `tests/registry_test.kf` | 12 |
| limites + varredura + push de SERVICES + trava de citação + família em fonte única | `tests/limits_test.kf` | 11 |
| service token | `tests/service_token_test.kf` | 15 |
| erros R6 | `tests/errors_test.kf` | 11 |
| dominios (+ certificado x hostname) | `tests/domains_test.kf` | 18 |
| metricas | `tests/metrics_test.kf` | 13 |
| rate limit | `tests/ratelimit_test.kf` | 2 |
| agent metrics | `tests/agent_metrics_test.kf` | 3 |
| abuso | `tests/abuse_test.kf` | 5 |
| frames | `tests/frames_test.kf` | 4 |
| idle + shutdown limpo do Edge | `tests/idle_test.kf` | 5 |
| conexoes | `tests/conns_test.kf` | 2 |
| deploy + scan | `tests/deploy_test.kf` | 16 |
| daemon | `tests/daemon_test.kf` | 14 |
| doctor | `tests/doctor_test.kf` | 17 |
| hot-reload | `tests/reload_test.kf` | 1 |
| control plane | `tests/control_test.kf` | 6 |
| presenca | `tests/presence_test.kf` | 1 |
| e2e | `tests/e2e_test.kf` | 1 |
| e2e multiservico | `tests/e2e_multiservice_test.kf` | 1 |
| e2e autenticado | `tests/e2e_authenticated_test.kf` | 1 |
| eventos do agent | `tests/agent_events_test.kf` | 6 |
| teto de corpo (WAF) | `tests/waf_body_test.kf` | 2 |
| detectar WAF + corpo + headers + encoded + allowlist (matcher) | `tests/waf_detect_test.kf` | 8 |
| timeout de upstream (WAF) | `tests/waf_timeout_test.kf` | 2 |
| allowlist WAF (Fase 10 slice 3) | `tests/waf_allow_test.kf` | 5 |
| painel somente-leitura | `tests/panel_test.kf` | 3 |
| rotacao de kfs_ (Fase 19 s3) | `tests/rotate_test.kf` | 6 |
| `security.txt` + `SECURITY.md` (Fase 19 s1 parcial) | `tests/security_test.kf` | 3 |
| incidentes (Fase 17 s1 + template pós-mortem s3) | `tests/incidents_test.kf` | 19 |
| contenção (Fase 17 s2) | `tests/freeze_test.kf` | 17 |
| status page (Fase 17 s3 parcial) | `tests/status_page_test.kf` | 6 |
| publish com allowlist pela CLI (F10 s3) | `tests/publish_allow_test.kf` | 4 |
| webhook de incidentes (F17 s3) + extração de host de URL | `tests/webhook_test.kf` | 12 |
| cadeia de auditoria (F19 s4) | `tests/audit_chain_test.kf` | 21 |
| scan de ambiente e PEM (F19 s2) | `tests/scan_extra_test.kf` | 5 |
| higiene de credenciais (F19 s2) | `tests/token_hygiene_test.kf` | 5 |
| painel de gestão completa (F12 s2) + webhook, diagnóstico, onboarding, fluxo guiado e contenção (F12 s2/s3) | `tests/panel_actions_test.kf` | 21 |
| ranking de Edges (F14 s2) | `tests/edge_select_test.kf` | 5 |
| porta do Edge sem wrap (F14 s2) | `tests/edge_select_test.kf` + `tests/webhook_test.kf` | +13 asserts |
| carga explícita + migrate (F16 s1) | `tests/migrate_test.kf` | 5 |
| readiness + fail-closed (F16 s3) | `tests/readiness_test.kf` | 2 |
| falha de bind visível do serve (F16 s3) | `tests/serve_bind_test.kf` | 3 |
| leitura de métricas pela CLI | `tests/metrics_cmd_test.kf` | 2 |
| listagens em `--json` (CI) | `tests/json_output_test.kf` | 2 |
| teto de corpo por rota (F10 s2) | `tests/route_cap_test.kf` | 2 |
| credencial da conta com ciclo de vida (F19 s3) | `tests/account_token_test.kf` | 11 |
| health HTTP com prazo (F4) | `tests/health_test.kf` | 5 |
| teto de resposta + cabeçalho + leitura (F14 s3 parcial) | `tests/relay_limit_test.kf` | 4 |
| profundidade de fila por stream (F4) | `tests/queue_depth_test.kf` | 3 |
| teto de conexões públicas (F4) | `tests/conn_limit_test.kf` | 2 |
| enquadramento HTTP nos dois sentidos (F10/F14) | `tests/framing_test.kf` | 14 |
| presença no `device list` (F4) | `tests/device_presence_test.kf` | 5 |
| origem do serviço + presença no `service status` (F4) | `tests/service_device_test.kf` | 6 |
| PEM visível no `service status` (F9 s3) | `tests/status_cert_test.kf` | 3 |
| erro do login com cadastro implícito (F5) | `tests/login_error_test.kf` | 2 |
| flags com valor, booleanas sem duplicação e posicional sobrando (CLI) | `tests/value_flags_test.kf` | 4 |
| eventos do Edge (`logs --edge`) | `tests/edge_events_test.kf` | 6 |
| SERVICES empurrado (F14 s3) | `tests/services_push_test.kf` | 2 |
| deploy --watch (F13 s3 parcial) | `tests/deploy_watch_test.kf` | 4 |
| uso por conta + ciclo de billing (F18 s4) | `tests/usage_test.kf` | 6 |
| segredos de fonte externa (F18 s5) | `tests/secrets_test.kf` | 4 |
| push do uso para billing (F18 s4) | `tests/usage_push_test.kf` | 4 |
| provedor DNS por comando (F18 s1) | `tests/dns_cmd_test.kf` | 6 |

Total: 475 testes. Ver `15-tests.md` para a matriz.
