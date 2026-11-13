# O que está implementado (e verificado)

Este documento descreve o estado **real** do código, não o plano. Tudo listado
como verificado foi medido com o CLI real e/ou com a suíte de testes.

## Verificado ponta a ponta

O critério de sucesso do pedido (§44) foi atingido em processos separados:

```
kof run src/Main.kf serve --control-port 28140 --tunnel-port 27060 --http-port 28141
kof run src/Main.kf login mel --password segredo --control http://127.0.0.1:28140
kof run src/Main.kf publish site --port 29010
kof run src/Main.kf connect

curl -H "Host: site.mel.kofcloud.io" http://127.0.0.1:28141/
→ hello from localhost
```

Sem port forwarding, sem IP público, sem VPN, sem reverse proxy manual. Host
desconhecido devolve `404`; serviço publicado mas sem túnel devolve `503`.

## Componentes implementados

| Módulo | Conteúdo | Teste |
|---|---|---|
| `protocol/` | enquadramento por linha, frame JSON, tunnel protocol, sessão multiplexada | `protocol_test.kf` |
| `identity/` | par X25519, HKDF-SHA256, AES-GCM, tokens opacos com hash | `identity_test.kf` |
| `model/` + `util/` | Account/Device/Service/Exposure, hostnames, parse de texto | `hosts_test.kf` |
| `store/` | persistência JSON | `registry_test.kf` |
| `control/` | registry + HTTP `/v1` (contas, devices, serviços, domínios, connect, heartbeat) | `control_test.kf` |
| `edge/` | listener de túnel, router por `Host`, HTTP front, relay, parada limpa | `e2e_test.kf` |
| `agent/` | connect, reconexão com backoff, heartbeat, health, relay, parada | `e2e_test.kf`, `backoff_test.kf` |
| `cli/` | dispatcher + comandos (`login`, `connect`, `publish`, …) | aceitação manual |
| limites | `util/Limits.kf` + enforcement no registry/sessão | `limits_test.kf` |
| múltiplos serviços | roteamento por `Host` no mesmo túnel | `e2e_multiservice_test.kf` |
| credencial de serviço | `kfs_` emitido no control plane, validado no Edge; revogacao por token (`token-revoke`) + inventario sem valores (`token-list`) + expiracao opcional (`--ttl`, `KCF062`) | `service_token_test.kf`, `e2e_authenticated_test.kf` |
| erros estáveis (R6) | `model/Errors.kf`: `KCFnnn` no control plane e no Edge + `Hint:` na CLI | `errors_test.kf` |
| domínio próprio (slice 1) | `model/CustomDomain.kf` + registro/verificação/roda no registry, `POST /v1/domains[/verify][/remove]`, CLI `domain add/verify/remove/list` (list mostra PENDING/VERIFIED no custom) + `service status` mostra os customs vinculados (linha `Dominio: custom <host> <status>`); `domain verify --tls-cert` julga o PEM localmente antes da rede (SAN/CN cobre o hostname, curinga de um rótulo; aviso de 30 dias; só caminho/nomes, nunca o conteúdo) | `domains_test.kf` |
| métricas do Edge (slice 1) | `edge/Metrics.kf`: túneis/recusas, sessões, ok/negadas por motivo, bytes, latência via `observability.*`; faixas `le_50…le_2000`/`gt_2000` para latência de cauda (rótulo derivado da lista única) | `metrics_test.kf` |
| métricas do control plane (slice 2) | `control/Metrics.kf`: lifecycle no Registry + `GET /v1/metrics` sem auth | `metrics_test.kf` |
| métricas do Agent (slice 3) | `agent/Metrics.kf`: connects, reconnects, fatais por motivo, health, streams, erros locais | `agent_metrics_test.kf` |
| rate limit por hostname (slice 1) | `security.rateLimit` no Edge público: 429 `KCF107` + `Retry-After` antes de rotear | `ratelimit_test.kf` |
| suspensão + auditoria (slice 2) | `SUSPENDED` reversível (serviço/dispositivo), enforcement em request+PING, anel de auditoria + `GET /v1/audit` + `kof cloud audit` | `abuse_test.kf` |
| frame DATA fatiado + teto de linha | `Payload.encodeChunks` (blocos de 16 KiB) + `LineCodec` fail-closed + `maxTunnelLineBytes` | `frames_test.kf` |
| conexoes simultaneas por servico | contador por `serviceId` no Edge + `429 KCF108` antes do OPEN | `conns_test.kf` |
| REVOKED terminal (F4) | `setDeviceStatus` recusa qualquer transição de `REVOKED`; `device unsuspend`/`suspend` num revogado → `409 KCF023` (`deviceIsRevoked`, fonte única) — antes `unsuspend` gravava `ACTIVE` e desfazia a contenção de uma máquina comprometida, sem re-registro nem nova chave | `abuse_test.kf` |
| domínio em serviço homônimo (F6) | `handleRegisterDomain` recusa nome de serviço ambíguo (estado legado com o mesmo nome em 2 devices) com `409 KCF140` e não registra nada — antes `serviceByName` escolhia o primeiro e o CNAME/rota apontava para a máquina errada em silêncio | `domains_test.kf` |
| publish em device revogado (F4) | `publishService` recusa device `REVOKED` com `KCF023` (borda: `409`, nada criado) — antes criava um serviço que nunca serviria (o device não abre mais túnel e o revoke já removeu os serviços dele); `SUSPENDED` segue publicável (reversível) | `registry_test.kf`, `control_test.kf` |
| republicar atualiza a config (F7) | `publishService` reescreve a config mutável do serviço existente (porta/address/protocol/exposure/allowlist) preservando id/hostname/createdAt/status; config idêntica não escreve nem audita (`serviceConfigEquals`); auditoria `service.updated` + métrica `kofcloud_control_services_updated_total` — antes devolvia o existente sem aplicar nada e o `deploy` imprimia "publicado" com a config velha enquanto `--plan` prometia "~ vai atualizar" | `registry_test.kf` |
| deploy declarativo (slice 1) | `kofcloud.json` + `deploy`/`down` sobre publish em lote (idempotente, sem tocar fora do manifesto; `down --prune` poda órfãos do device atual). Artefato declarado tem de ser arquivo legível (`artifactProblem` via `readText`, não `exists` — diretório recusado); nome de serviço duplicado no manifesto falha posicional; nome já usado noutro device da conta → `409 KCF139` (antes devolvia 200 com o serviço do outro device) | `deploy_test.kf` (inclui `build` por `sh -c`: argumentos, pipes e exit code; falha traz a última linha de stderr/stdout no detalhe) |
| sufixos de métrica completos | KCF116 (cabeçalho grande demais) e KCF105 (resposta vazia) tinham caído no contador genérico; ramo morto de KCF117 removido (a recusa no `accept` tem contador próprio); invariante de fonte trava o conjunto | `metrics_test.kf` |
| check local do deploy | `checkLocalServiceFor` usa a sonda do Agent (`GET /` com prazo, < 500); o connect TCP antigo aprovava serviço 500 e serviço mudo | `deploy_test.kf` (`deploy-check-http`) |
| check acusa serviço suspenso (F7) | `verifyDeploymentFor` acusa `SUSPENSO (fora da rota)` com a dica do `unsuspend` (`suspendedByName` pura) — antes o `--check`/`doctor` davam verde para um serviço fora do ar: respondia localmente e republicar não dessuspende | `deploy_test.kf` |
| doctor distingue ciclo de vida do device | camada `doctorDeviceLine` pura: REVOKED → FALHA com o re-registro (não sugere `connect`, recusado fatal), SUSPENDED → FALHA com o `unsuspend`; antes dizia `ok dispositivo registrado` e a camada do túnel mandava rodar o connect | `doctor_test.kf` |
| diagnostico doctor | somente-leitura em camadas (login, control, device, servicos, tunel) + `--file` + `--tls-cert` (PEM ausente/invalido/expirado em 30 dias e, agora, **cobertura dos hostnames do manifesto** — PEM de outro domínio vira FALHA em vez de quebrar o HTTPS em produção; sem vazar conteúdo) | `doctor_test.kf` |
| daemon user-level (slice 3) | `cli/Daemon.kf`: unit systemd `--user` + install/status/uninstall | `daemon_test.kf` |
| presenca real do tunel | espelho bind/unbind no Registry + `GET /v1/presence` filtrado por conta + `status` ONLINE/OFFLINE confirmado | `presence_test.kf` |
| `security.txt` + `SECURITY.md` + SBOM (Fase 19 s1 parcial) | `/.well-known/security.txt` no Edge (200 `text/plain`, Contact + Policy estáticos) antes de rate limit/rota/auth; sem conta/túnel/serviço; `docs/SECURITY.md` com suporte/reporte/SLA/escopo e trava; `audit --sbom` local (48 fontes + SHA-256, sem login) | `security_test.kf` + `hosts_test.kf` (`audit-sbom`) |
| scan de segredos (Fase 19 s2 parcial) | `kof cloud scan [--file]` local (kfs_/kfc_/password no manifesto, nomeia sem vazar, exit 1 com achado); sem login/rede | `deploy_test.kf` (`deploy-scan`) |
| higiene de credenciais (F19 s2) | `service tokens` / `/v1/services/token-hygiene`: `kfs_` válidos sem expiração por conta | `token_hygiene_test.kf` |
| scan de ambiente/PEM (F19 s2) | `scan` cobre manifesto + env sensível + PEM expirando (`--tls-cert`), sem vazar valor | `scan_extra_test.kf` |
| cadeia de auditoria (F19 s4) | `prevHash`/`hash` encadeados **por conta** (o anel é global; antes o encadeamento global fazia duas contas ativas reportarem cadeia quebrada sem adulteração); `audit --verify` e `/v1/audit/verify` detectam adulteração/remoção | `audit_chain_test.kf` |
| webhook de incidentes (F17 s3) | HMAC-SHA256 (`X-KofCloud-Signature`), URL validada, segredo nunca devolvido, `kof cloud webhook set/show/remove/test` | `webhook_test.kf` |
| status page pública (Fase 17 s3) | `serve --status-page` → `/status` (HTML) e `/status.json`, sem token, só incidentes globais | `status_page_test.kf` |
| unfreeze sem freeze (F17 s2) | `unfreezeService/Device/Account` devolvem `UnfreezeResult(ok, restored)`: alvo inexistente/alheio → 404, alvo existente mas nunca congelado → `409 KCF141` e status intocado — antes gravava UNKNOWN/ACTIVE e apagava uma suspensão deliberada do operador, com o CLI dizendo "status anterior restaurado" | `freeze_test.kf` |
| contenção `freeze`/`unfreeze` (Fase 17 s2) | suspende + revoga credenciais do alvo e abre incidente; unfreeze restaura o status anterior (reversível por desenho). Alvos: `service`, `device` e `account` (conta do token, sem id — devices + serviços + todas as `kfs_`); a marca original nunca é sobrescrita por um segundo freeze | `freeze_test.kf` |
| template de post-mortem (Fase 17 s3) | `docs/incidents/README.md` — 5 seções do ciclo (causa/impacto/timeline/correção/preventiva com dono+prazo), sem dado de cliente; trava cita `freeze`/`resolve` reais | `incidents_test.kf` (`postmortem-template`) |
| incidentes (Fase 17 s1) | `model/Incident.kf` + detecção por conta (revoke em lote via auditoria e expiração em massa de `kfs_` na varredura preguiçosa) e global no Edge (recusas de túnel, rajada de 401 em serviço AUTHENTICATED e quedas repetidas por ociosidade — Edge sem heartbeat); `POST /v1/incidents`, CLI `incidents` (`--resolve` solto/repetida barra no uso — antes saía 0 e listava como se nada tivesse sido pedido); limiares/janelas dos detectores em fonte única (`util/Limits.kf`, config `kofcloud.limits.*`) | `incidents_test.kf`, `idle_test.kf` |
| rotação de `kfs_` (Fase 19 s3) | `service rotate (<nome>\|--all)`/`rotation`: emite novo e revoga antigos só após o TTL (overlap); `--all` com falha isolada por serviço; sem TTL = pendente, nunca silencioso | `rotate_test.kf` |
| daemon por SO (Fase 13 s1) | `daemonKind` + artefato systemd/launchd/Windows puro; install grava e imprime a ativação (nunca ativa sozinho) | `daemon_test.kf` (`daemon-os`) |
| allowlist WAF pela CLI (fonte única) | `--allow-method`/`--allow-path` entrou em `valueFlags()`; antes o publish com allowlist barrava sempre no erro de uso; o teste da fonte única deriva a lista dos fontes | `publish_allow_test.kf` |
| allowlist WAF pela CLI | `publish --allow-method`/`--allow-path` (repetíveis) com validação local; `service status` mostra a política | `publish_allow_test.kf` |
| allowlist WAF (Fase 10 s3) | `allowMethods`/`allowPaths` no publish → `403 KCF111` antes do relay; `KCF113`/`KCF114` para política malformada; `kofcloud.json` cobre os campos (mesma normalização, drift no `--plan`/`--check`/`doctor`) | `waf_allow_test.kf` + `deploy_test.kf` |
| health HTTP com prazo (F4) | `GET` real ao serviço local: < 500 = ONLINE, ≥ 500 ou sem resposta no prazo = DEGRADED, recusa = OFFLINE; o ciclo lê a tabela viva da sessão a cada rodada (hot-reload), não a lista do connect — removido para de ser sondado, novo passa a ser; teto do cabeçalho da sonda na fonte única (`limitHealthResponseHeadBytes`) | `health_test.kf` |
| posicional sobrando em todo comando (CLI) | `device revoke/suspend/unsuspend`, `logs`, `serve` e `webhook` recusam posicional extra antes de qualquer efeito (antes `device revoke notebook lixo` revogava `notebook` em silêncio, removendo serviços e credenciais); trava de fonte exige `extraPositional` em todo `cmd*` | `value_flags_test.kf` |
| flags com valor e booleanas na CLI | `valueFlags()` alimenta `extraPositional`; toda flag lida com `flagValue` está nela (23); toda flag booleana (`hasFlag`) nos 4 arquivos da CLI tem `flagDuplicated` no mesmo comando (`booleanFlagsWithoutDupGuard` trava 12 ocorrências); `deploy` (--check/--plan), `down` (--prune), `incidents` (--all), `migrate` (--dry-run), `logs` (--runtime), `serve` (--panel/--status-page) barram repetição em vez de executar em silêncio; `positionalArgs` unifica `firstPositional`, `secondPositional` e `extraPositional` — flag antes do posicional (`publish --port 8080 site`, `login --password s mel`, `webhook set --secret s url`) não corrompe mais o argumento | `value_flags_test.kf`, `hosts_test.kf`, `migrate_test.kf` |
| erro do login com cadastro implícito (F5) | senha errada numa conta existente acusava `KCF002` (handle em uso) porque o cadastro implícito vazava o erro dele; agora vira `KCF003` (credencial inválida) | `login_error_test.kf` |
| sumário do `down` (F7) | `downSummary`: conta o que saiu (nunca `services.size`); "nada a remover" quando nada era deste dispositivo; falha domina | `deploy_test.kf` (`deploy-down-summary`) |
| `down` noutro device (F7) | `serviceIdOnDevice`/`serviceDeviceElsewhere`: homônimo publicado noutro device não é "ausente (ok)" — mensagem diz onde vive e que o down deve ser lá | `deploy_test.kf` (`deploy-down-scope`) |
| escopo de device no `--plan` | `diffDeploymentFor`: nome tomado noutro device vira `! conflito` (o deploy recusaria com 409) e extras de outro device não são listados (o prune não os alcança) | `deploy_test.kf` (`deploy-plan`) |
| escopo de device no `--check`/`doctor` | `verifyDeploymentFor` casa nome **e** device da sessão; serviço publicado noutro device vira problema explícito | `deploy_test.kf` (`deploy-check`) |
| nome ambíguo na CLI (F4) | `resolveServiceName`/`serviceAmbiguousLine`: homônimo entre devices não é escolhido — os comandos por nome recusam listando os devices | `hosts_test.kf` (`service-name-match`) |
| presença no `service status` (F4) | linha `Tunel:` do device que serve o serviço via `/v1/presence` + aviso quando health=ONLINE com tunel OFFLINE (`serviceTunnelLine`/`serviceReachabilityNote`, puras) — antes o serviço lia ONLINE com o site fora do ar | `service_device_test.kf` |
| origem do serviço na CLI (F4) | `status`/`service list`/`service status` mostram `device: <nome>` (mesma `serviceListLine`); `status`/`token` recusam nome homônimo (estado legado) em vez de escolher o primeiro (id quando o device saiu da conta); antes a lista não dizia onde o serviço roda | `service_device_test.kf` |
| estado do agente no `status` (F4) | `agentStatusLine` deriva de presença real (ATIVO só com túnel vivo; sem presença = desconhecido; PARADO distingue parada pedida de nunca subiu) — antes dizia ATIVO só por ausência do stopFile | `device_presence_test.kf` |
| presença no `device list` (F4) | linha mostra `tunel: ONLINE/OFFLINE/desconhecido` via `/v1/presence`; falha não vira OFFLINE inventado | `device_presence_test.kf` |
| teto de conexões públicas (F4) | 1024 simultâneas; excedeu → `503 KCF117` no `accept`, sem ler o pedido; vaga liberada em `finally` | `conn_limit_test.kf` |
| profundidade de fila por stream (F4) | teto de 64 itens por stream; fila cheia → `RESET LIMIT_EXCEEDED` no stream, túnel segue (política única em `routeData`) | `queue_depth_test.kf` |
| higiene de logins (F19 s3) | `token prune` / `POST /v1/tokens/prune`: revoga as credenciais antigas da conta e mantém a da sessão atual (cada `login` emitia uma nova sem revogar a anterior) | `account_token_test.kf` |
| vocabulário fechado do status (F5) | `POST /v1/services/status` recusa status fora do vocabulário com `400 KCF048` (antes `BANANA` virava 200 e ficava no estado); `SUSPENDED` não é reportável por health | `status_report_test.kf` |
| rate limit no login (F5) | tentativas por handle no control plane (`429 KCF134` + `retryAfterSec` antes do hash); acerto reseta a janela | `login_limit_test.kf` |
| credencial da conta com ciclo de vida (F19 s3) | `kfc_` com TTL opcional, inventário sem valores, revoke por id curto e rotação com overlap; `token list/revoke/rotate/rotation`; expiração calculada em `Long` (`tokenExpiryAt`, fonte única de kfc_/kfs_ — `Int` estourava acima de 596 h e `--ttl 720` nascia vencido) | `account_token_test.kf`, `service_token_test.kf` |
| teto de resposta + cabeçalho + leitura (F14 s3 parcial) | resposta acima do teto → 502 KCF115 com stream cancelado dos dois lados (Agent para de produzir); cabeçalho sem fim → 431 KCF116; corpo acima do teto de leitura → 413 KCF110 sem acumular (declarado ou real) | `relay_limit_test.kf` |
| carga explícita + `migrate`/`state-backup` (F16 s1) | `loadStatus` ok/absent/corrupt (serve avisa), `migrate --dry-run` compara contagens sem escrever, `state-backup` preserva o arquivo antes da troca de backend | `migrate_test.kf` |
| porta do Edge sem wrap (F14 s2) | `parseEdgeAddress` lê a porta com `parsePortSafe`; `:4294968080` virava 784 e o Agent conectava no lugar errado | `edge_select_test.kf` |
| contrato `/v1` documentado | cinco rotas registradas estavam fora da tabela do `docs/06` (audit/verify, services/rotate, services/rotation, token-hygiene, webhook); teste compara as duas fontes | `control_test.kf` (`control-docs`) |
| contrato do `/v1/connect` documentado | as docs prometiam `tunnelToken` (nunca emitido) e mostravam `edgePort`/`heartbeatMs` como número, mas tudo viaja como string (o Agent decodifica `Map<String,String>`); `newTunnelToken` morto removido | `edge_select_test.kf` |
| ranking de Edges (F14 s2) | `/v1/connect` publica `edges`; Agent mede e escolhe o mais rápido | `edge_select_test.kf` |
| faixas de tamanho da resposta (F14 s1) | `response_bytes_total_size_kb_le_*` em `noteHttpFinished` | `metrics_test.kf` (`metrics-size`) |
| painel — gestão completa (F12 s2) | formulários no `serve --panel` chamando as rotas `/v1` da CLI; `confirm()` nas destrutivas; publicar exige **escolher** o dispositivo (`<select svcdev>`, nunca o primeiro da lista); 'Adicionar domínio' envia `service` (nome) — o `serviceId` fazia a rota devolver 404 KCF122; recarrega devices/presença após cada ação | `panel_actions_test.kf` |
| painel somente-leitura (Fase 12 s1) | `serve --panel` → `GET /panel` estático (`control/Panel.kf`) lendo `/v1` com o token colado na página | `panel_test.kf` |
| timeout de upstream (WAF 2b) | watchdog entrega RESET no stream → `504 KCF112` (ou parcial) + métrica `_timeout` | `waf_timeout_test.kf` |
| teto de corpo (WAF 2a) | `kofcloud.limits.httpBodyBytes` → `413 KCF110` antes do relay + métrica `_body_too_large` | `waf_body_test.kf` |
| enquadramento do corpo (F10/F14) | `Content-Length` lido em **Long** (acima de 2^31 não vira wrap/negativo) e usado como fronteira exata do corpo relayado; duplicado divergente, valor não numérico/vazio/com sinal, nome fora da forma canônica e `Transfer-Encoding` → `400 KCF118` antes de rotear + métrica `_invalid_framing` | `framing_test.kf` |
| `Host` não ambíguo (F10/F14) | `hostIsUnambiguous`: exatamente um `Host`, não vazio, sem controle/espaço → senão `400 KCF119` antes de rotear + métrica `_invalid_host`; o serviço local nunca vê um `Host` diferente do roteado | `framing_test.kf` |
| cabeçalhos hop-by-hop (F10/F14) | `headerIsForwarded`/`connectionTokens`: hop-by-hop do RFC 9110 + tokens do `Connection` + `X-Forwarded-For` do cliente não chegam ao serviço; `Proxy-Authorization` não vaza | `framing_test.kf` |
| resposta sanitizada (F10/F14) | `sanitizeResponse`: hop-by-hop e tokens do `Connection` do upstream saem, `Connection: close` do Edge entra (status/corpo intactos); vale para a resposta parcial | `framing_test.kf` |
| resposta chunked decodificada (F10/F14) | `decodeChunked`/`parseHexSafe`: `Transfer-Encoding` do upstream sai e o corpo é entregue decodificado (extensões e malformado com teto/inválido; o close delimita) | `framing_test.kf` |
| status sem corpo (F10/F14) | `statusAllowsBody`: 1xx/204/304/HEAD descartam o corpo; 1xx/204 também o `Content-Length` (304/HEAD preservam) | `framing_test.kf` |
| `Content-Length` da resposta (F10/F14) | `sanitizeResponse` corta o corpo no declarado e reescreve o CL para o tamanho real; 304/HEAD preservam; sem CL não inventa | `framing_test.kf` |
| resposta não-HTTP recusada (F10/F14) | `responseIsParseable`: exige linha `HTTP/<versão> <código 100–599>`; senão `502 KCF131` + `_invalid_response`, sem ecoar bytes do upstream | `framing_test.kf` |
| linha de requisição validada (F10/F14) | `requestLineIsValid`: três partes, método `token`, path absoluto sem controle (CR/tab quebrariam a linha ao repassar), `HTTP/1.0/1.1`; senão `400 KCF132` + `_invalid_request_line` antes de rotear | `framing_test.kf` |
| cabeçalhos validados (F10/F14) | `headerLineIsValid`/`headersAreValid`: `Nome: valor` com nome `token` e valor sem CR/LF/NUL; senão `400 KCF133` + `_invalid_header_line` antes de rotear | `framing_test.kf` |
| 1xx intermediário descartado (F10/F14) | `sanitizeResponse` pula blocos `1xx` completos antes da resposta final (teto 8; sem terminador devolve cru) | `framing_test.kf` |
| query fora da rota (F10/F14) | `pathWithoutQuery`: security.txt e allowlist decidem pelo caminho; o WAF vê o path com query; o pedido repassado mantém a query | `framing_test.kf`, `security_test.kf` |
| porta sem estouro de `Int` | `parsePortSafe`/`portFlagValue`: publish (CLI/manifesto/API) e `serve` lêem porta em `Long` com teto 65535; flag inválida falha, não cai no default | `hosts_test.kf`, `errors_test.kf` |
| inteiros de contagem/TTL sem estouro | `parseIntBounded`: `--tail`, TTL (CLI/API) e contadores `pending`/`count` lidos em `Long` com teto; fora da faixa é erro, não wrap | `hosts_test.kf`, `e2e_authenticated_test.kf` |
| `frameInt` sem estouro | campos numéricos de `HELLO`/`WELCOME`/`DATA` lidos com `parseIntBounded`; versão/heartbeat forjados pelo fio não fazem wrap | `protocol_test.kf` (`frame-int`) |
| `parseStatusLine` sem estouro | código da linha de status lido com `parseIntBounded` (teto 599); fora de 100–599 é 0 → DEGRADED | `health_test.kf` |
| auditoria de `parseIntSafe` | entrada externa em `parsePortSafe`/`parseIntBounded`; contadores internos (`Session.pending`, `EdgeState.serviceConns`, `Registry.expiryProbes`) documentados no código; `token prune` endurecido | `hosts_test.kf` |
| trava de unicidade dos identificadores U | `u-matrix` lê `docs/15-tests.md` e quebra se um `\| Unn \|` se repetir | `hosts_test.kf` (`u-matrix`) |
| totais do STATUS travados por suite | `status-counts` compara a tabela Suite/Testes do `docs/STATUS.md` com a contagem real de `test "` de cada arquivo (protocol 11→12, metrics 9→10 corrigidos) | `hosts_test.kf` (`status-counts`) |
| detectar WAF — corpo pequeno | `wafDetectBodySignature` (até 64 KiB, JSON-escapado já casa) no mesmo caminho de contagem/enforce do path; teto fonte-única (`limitWafBodyInspectBytes`, `kofcloud.limits.wafBodyInspectBytes`, default 64 KiB travado) | `waf_detect_test.kf` (`wafbody-unit`, `wafdetect-body`) |
| detectar WAF — headers | `wafDetectHeadersSignature` varre valores de headers do cliente (User-Agent/Referer/Cookie/X-*) até o mesmo teto; `wafHeaderIgnored` exclui `Authorization`/`Host`/`Content-Length`/`Connection`/`X-Forwarded-For`/`X-KofCloud-Token`; padrões vêm de `wafSignatureOf` (a mesma lista de path e corpo) | `waf_detect_test.kf` (`wafheader-unit`, e2e em `wafdetect`/`wafenforce`) |
| detectar WAF (slice 1-detectar) | `wafDetectSignature` pura (método+path) + contagem por padrão no `HttpFront` (relay segue, resposta marcada `would-block`) | `waf_detect_test.kf` (`wafdetect-unit` + `wafdetect`) |
| enforce WAF núcleo (slice 1-enforce) | `EdgeState.setWafEnforce(true)` → casa vira `403 KCF109` + `blocked` antes do relay (sem stream, sem Agent) + métrica `_waf_denied` | `waf_detect_test.kf` (`wafenforce`: ataque não chega ao local, legítimo 200 antes/depois) |
| flag WAF no serve (slice 1-flag) | `serve --waf detectar|enforce` (`resolveWafMode` pura + fail-fast: solta/repetida/lixo barra; fio no EdgeState; linha `waf:` no banner) | `hosts_test.kf` (`serve-waf`) + locks ajuda (`daemon_test.kf`) |
| fundação KCF111 (allowlist slice 3) | construtor `errWafAllowDenied()` + Hint (métrica `_waf_allow_denied`) + sufixo em `noteHttpFinished` + matcher `wafAllowCheck` puro; enforcement no HttpFront (linha 51) | `errors_test.kf` (`errors-hint`, `errors-hint-all`: 59 construtores (inclui KCF115-KCF119 do Edge, KCF125-KCF130 de contencao/webhook e KCF131-KCF139); camada 3 compara a lista canonica com os construtores) + `metrics_test.kf` + `waf_detect_test.kf` (`wafallow-unit`) |
| eventos do Agent | `agent/Events.kf`: `agentEvent` emite para `log.*` **e** `agent-runtime.log` (teto 256 KiB); `logs --runtime` | `agent_events_test.kf` |
| idle timeout (tunel + stream) | `lastActivityMs` + sweepers Edge/Agent, EOF limpo vira throw (reconecta) | `idle_test.kf` |

## Comportamentos provados

* **Túnel reverso persistente** iniciado pela máquina; sobrevive atrás de NAT
  (conexão de saída).
* **Multiplexação**: vários streams (requisições) sobre uma conexão, com filas
  por stream (canais), sem um serviço monopolizar os outros.
* **Vários serviços no mesmo túnel (Fase 4)**: um Agent serve N hostnames em N
  portas locais; cada host chega na sua porta e derrubar um serviço local não
  afeta os outros (`e2e_multiservice_test.kf`).
* **Limites aplicados**: dispositivos ativos por conta, serviços por
  dispositivo e streams simultâneos por túnel — 429 no control plane e
  429/`RESET LIMIT_EXCEEDED` no Edge (`limits_test.kf`).
* **Reconexão com backoff exponencial + jitter** (observável no log do agente
  quando o Edge não está no ar).
* **Health check**: o agente reavalia os serviços periodicamente e reporta só na
  mudança; o Edge para de encaminhar quando fica `OFFLINE` (`503`).
* **Cifra do túnel** (opcional, `KOFCLOUD_TUNNEL_SECRET`): X25519 efêmero +
  HKDF por sessão + AES-GCM nos payloads — dois lados derivam a mesma chave
  (testado).
* **Exposição**: `PRIVATE` não gera rota; `AUTHENTICATED` exige credencial de
  serviço `kfs_` **antes** do túnel (`e2e_authenticated_test.kf`), com o token
  da conta recusado e a credencial removida antes de chegar ao serviço local.
* **Revogação de dispositivo** derruba a sessão e remove as rotas dele.
* **Teto de corpo por requisição**: `Content-Length` acima de
  `kofcloud.limits.httpBodyBytes` (1 MiB) é recusado no Edge com `413 KCF110`
  antes de consumir stream ou tocar no Agent; exatamente no teto passa
  (`waf_body_test.kf`).
* **Timeout de upstream**: um serviço local que aceita a conexão e nunca
  responde não pendura o Edge — após `kofcloud.limits.tunnelIdleMs` o cliente
  recebe `504 KCF112` (ou o corpo parcial, se já havia bytes) e a vaga de
  conexão é liberada (`waf_timeout_test.kf`).
* **Observabilidade local do Agent**: os eventos reais do túnel ficam
  consultáveis depois do processo terminar (`logs --runtime`), sem depender de
  stdout — e sem gravar segredo (`agent_events_test.kf`).
* **Domínio próprio (Fase 6, slice 1)**: hostname externo registrado como
  PENDING com instrução CNAME, verificado por `net.resolve` contra os destinos
  esperados do Edge (comparação exata; NXDOMAIN/falha = `KCF124` opaco), rota
  ativa só após VERIFIED; conta alheia recebe o mesmo `404` opaco; poda junto
  com serviço/dispositivo; persistido no estado (`domains_test.kf`, 9 testes).
  Sem TLS/SNI neste slice (terminador externo — ver `12-tls.md`).
* **Observabilidade mínima (slice 1)**: `edge/Metrics.kf` conta túneis
  aceitos/recusados por motivo, sessões abertas, requisições ok/negadas por
  motivo, erros de upstream, bytes e latência — via `observability.*`
  (export Prometheus). Nenhum hostname, corpo, credencial ou chave chega às
  métricas (`metrics_test.kf`, inclui asserção de privacidade).
* **Observabilidade mínima (slice 2)**: `control/Metrics.kf` conta lifecycle
  no Registry (contas, logins, devices, serviços, tokens, domínios);
  `GET /v1/metrics` serve o export sem auth.
* **Observabilidade mínima (slice 3 — fecha)**: `agent/Metrics.kf` conta
  connects, reconnects, recusas fatais por motivo, health reports, streams
  servidos e erros locais (agregados; processo próprio, sem protocolo novo).
* **Erros estáveis (R6, padrão kof-share)**: todo erro viaja como `KCFnnn`
  (`KCF001` handle inválido ... `KCF107` rate limited); recurso de outra
  conta responde o mesmo `404` opaco de inexistente; `POST /v1/services/status`
  tem owner-gate (só a conta dona altera); a CLI imprime `Hint:` humana sem
  vazar segredos (`errors_test.kf`).
* **Rate limit por hostname (abuse prevention, slice 1)**: 100 req/60 s por
  host (configurável), `429 KCF107` + `Retry-After` antes de rotear — não
  revela existência nem consome túnel; cota isolada por hostname; métrica
  `_rate_limited` (`ratelimit_test.kf`). A janela (config em ms) é convertida
  para a unidade do stdlib (segundos) num único ponto, `httpWindowSeconds`;
  sem isso o default de 60 s valia 60.000 s de bloqueio por hostname.
* **Suspensão + auditoria (abuse prevention, slice 2)**: serviço `SUSPENDED`
  sai da rota (opaco), device `SUSPENDED` tem o túnel recusado com
  `DEVICE_SUSPENDED` (fatal); health/API não dessuspendem; enforcement
  imediato (request) e contínuo (PING); anel de 100 eventos só com metadados,
  `GET /v1/audit` filtrado por conta (`abuse_test.kf`, 3 testes).
* **Frame DATA fatiado + teto de linha (débito de limites)**: envio em blocos
  de 16 KiB adaptativos (encolhem até caber — base64 via UTF-8 expande);
  recepção recusa linha gigante com fail-closed (`LINE_OVERFLOW` no Edge,
  throw de reconexão no Agent); 1.5 MiB atravessa fatiado fim a fim
  (`frames_test.kf`, 4 testes).
* **Idle timeout (débito de limites)**: sessão sem frames úteis cai (PING/PONG
  não renovam; varredura 5 s, default 120 s, métrica `_idle_timeout` no Edge)
  e o Agent reconecta sozinho; stream preso leva `RESET IDLE_TIMEOUT`
  (default 30 s) sem derrubar a sessão; EOF limpo agora reconecta (só
  stopFile/GOAWAY param) (`idle_test.kf`, 4 testes).
* **Conexões simultâneas por serviço (fecha os débitos)**: contador por
  `serviceId` no Edge, `429 KCF108` + `Retry-After` antes do OPEN, liberação
  ao fim do relay, isolamento por serviço, métrica `_service_overloaded`
  (`conns_test.kf`, 2 testes).
* **Deploy declarativo (Fase 7, slice 1 + slice 5 parcial)**: `kofcloud.json` declara
  `services[]` (nome, porta numero ou string, address/exposure/protocol/build/
  artifact opcionais); `kof cloud deploy` roda o `build` de cada serviço (quando
  declarado) e confere o `artifact` declarado (ausente = não publica aquele
  serviço, fail-fast antes do publish) antes do publish em lote (idempotente;
  build falhou = não publica
  aquele serviço) e `kof cloud down` remove
  só os listados — nada fora do manifesto é tocado; parser manual com erro
  posicional (`deploy_test.kf`, 8 testes). `deploy --plan` mostra o diff sem
  aplicar (`+` novo, `~` atualiza, `=` igual, `-` fora do manifesto; com
  `(build)`/`(artifact)` quando o serviço declara build/artifact — inclusive no
  igual, porque o deploy executa o build e confere o artifact antes de cada
  publish, mesmo o idempotente).
  `deploy --check` confirma publicado + respondendo (mesma regra do Agent;
  sem rollback), acusa drift de config antes da resposta (porta/address/
  exposure/protocol diferentes) e avisa `artefato ausente` quando o manifesto
  declara `artifact` mas o arquivo sumiu do disco (staleness, nunca bloqueio).
  O `doctor` usa o mesmo
  `verifyDeployment` na etapa de serviços (não publicado → drift → sem
  resposta → artefato), então `--check` e `doctor` acusam igual.
* **Daemon user-level (Fase 7, slice 3 + Fase 13 slice 1)**: `cli/Daemon.kf` gera
  o artefato do SO (`daemonUnitText` systemd no Linux, `daemonLaunchdPlist`
  no macOS, `daemonWindowsScript` no Windows; mesmo `connect`,
  `Restart=always`, `NoNewPrivileges`, enable/ativacao manual); `status`/
  `uninstall`; erro honesto sem systemd (`daemon_test.kf`, 13 testes).
* **Parada limpa** do agente (`disconnect` via arquivo-sentinela) e da
  plataforma (`serve`, via fechamento do `Listener`).
* **Hot-reload de serviço (debito quitado)**: refresher via `/v1/connect`
  troca a tabela da sessão (objeto compartilhado — Map nao atravessa
  `spawn`); handshake OPENED antes de DATA, OPEN recusado = 502 imediato
  em vez de relay travado (`reload_test.kf`, OPENED em `protocol_test.kf`).

## Não implementado (honesto)

* Terminação **TLS** no processo Kof e **ACME** — o Kof não expõe TLS em socket
  cru; ver `12-tls.md`. O hostname/roteamento por `Host` está pronto, mas o
  HTTPS depende de um terminador (`web.app().listenSecure`, JVM) ou externo.
* **DNS** como serviço — o wildcard por conta é configuração externa de DNS
  (ver `11-dns.md`); o KofCloud já emite o hostname correto.
* **WebSocket** público, **HTTP/2/3**, **domínio próprio automatizado**,
  **deploy com artefato transferível**, **painel web completo** (só o slice 1
  existe), **daemon: tray + auto-update** (slices 2–3 da Fase 13).
* **Múltiplos nós** de control plane/edge (sem consenso; registro único em
  memória + arquivo).
* **TLS/HTTPS na borda (Fase 9)** — slice 1 parcial: control plane HTTPS
  via `serve --tls-cert/--tls-key` (`listenSecure` 3-arg, JVM) + HSTS (vem
  do runtime do Kof) + **fluxo real da CLI provado sobre HTTPS**
  (`control-tls-cli`) + **aviso de PEM expirando** (`openssl -checkend`
  30d, aviso nunca bloqueio); o resto (301, SNI por hostname, Edge HTTPS)
  segue pendente — Edge esbarra em L2 (`net` sem TLS) e L16
  (handler `String`);
  ACME automático: ES256/JWS pousou no lab do Kof, ainda fora do jar
  pinado. **Ressalva:** o cliente `kof.http` é trust-all — cifra, mas não
  autentica o servidor (ver `12-tls.md` e `17-limitations.md`).
* **WAF + proteção da aplicação (Fase 10)** — slices: assinaturas mínimas
  (detectar antes de bloquear), teto de corpo/timeout por rota, allowlist
  por serviço. Regra: nunca logar conteúdo, nunca quebrar o legítimo.
* **PQC no túnel (Fase 11)** — híbrido ML-KEM+X25519 com fallback clássico;
  bloqueado no primitivo do stdlib (só X25519/Ed25519 hoje).
* **Gestão web (Fase 12)** — painel somente-leitura → gestão completa →
  onboarding guiado; consome só a API `/v1` pública, nunca mostra `kfs_`.
* **Desktop client (Fase 13)** — instalador por SO + tray com estado +
  sincronização do manifesto + auto-update com assinatura Ed25519.
* **Baixa latência (Fase 14)** — medir (p50/p99) → relay regional →
  caminho de bytes O(n); sem número antes/depois, sem otimização.
* **E2EE real (Fase 15)** — slice 1 (modelo de visibilidade em
  `docs/20-visibility.md`: quem vê o quê hoje vs alvo corpo-opaco) ✅;
  corpo opaco (slice 2) e SNI mínimo (slice 3) pendentes; tensão honesta
  com WAF (por serviço: um ou outro).
* **Arquitetura escalável (Fase 16)** — estado plugável → Edge horizontal
  → control ativo-passivo → células; mesma API, fail-closed, rollback.
* **Resposta a incidentes (Fase 17)** — detecção com severidade → `freeze`
  reversível → status page + webhook assinado + post-mortem; game-day.
* **Integrações de mercado (Fase 18)** — DNS/cert, OIDC/SSO+SCIM, CI/CD
  com `--plan`, OTel/billing, segredos externos; sem lock-in, dry-run.
* **Segurança contínua (Fase 19)** — SBOM + `security.txt`, scan no CI,
  rotação assistida + kill-switch **documentado** (`SECURITY.md` ##
  Contenção), audit log WORM rumo a SOC2.

Ver `16-plan.md` para as fases e `17-limitations.md` para as lacunas.
