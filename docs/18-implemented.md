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
| domínio próprio (slice 1) | `model/CustomDomain.kf` + registro/verificação/roda no registry, `POST /v1/domains[/verify][/remove]`, CLI `domain add/verify/remove/list` (list mostra PENDING/VERIFIED no custom) + `service status` mostra os customs vinculados (linha `Dominio: custom <host> <status>`) | `domains_test.kf` |
| métricas do Edge (slice 1) | `edge/Metrics.kf`: túneis/recusas, sessões, ok/negadas por motivo, bytes, latência via `observability.*` | `metrics_test.kf` |
| métricas do control plane (slice 2) | `control/Metrics.kf`: lifecycle no Registry + `GET /v1/metrics` sem auth | `metrics_test.kf` |
| métricas do Agent (slice 3) | `agent/Metrics.kf`: connects, reconnects, fatais por motivo, health, streams, erros locais | `agent_metrics_test.kf` |
| rate limit por hostname (slice 1) | `security.rateLimit` no Edge público: 429 `KCF107` + `Retry-After` antes de rotear | `ratelimit_test.kf` |
| suspensão + auditoria (slice 2) | `SUSPENDED` reversível (serviço/dispositivo), enforcement em request+PING, anel de auditoria + `GET /v1/audit` + `kof cloud audit` | `abuse_test.kf` |
| frame DATA fatiado + teto de linha | `Payload.encodeChunks` (blocos de 16 KiB) + `LineCodec` fail-closed + `maxTunnelLineBytes` | `frames_test.kf` |
| conexoes simultaneas por servico | contador por `serviceId` no Edge + `429 KCF108` antes do OPEN | `conns_test.kf` |
| deploy declarativo (slice 1) | `kofcloud.json` + `deploy`/`down` sobre publish em lote (idempotente, sem tocar fora do manifesto; `down --prune` poda órfãos do device atual) | `deploy_test.kf` |
| diagnostico doctor | somente-leitura em camadas (login, control, device, servicos, tunel) + `--file` | `doctor_test.kf` |
| daemon user-level (slice 3) | `cli/Daemon.kf`: unit systemd `--user` + install/status/uninstall | `daemon_test.kf` |
| presenca real do tunel | espelho bind/unbind no Registry + `GET /v1/presence` filtrado por conta + `status` ONLINE/OFFLINE confirmado | `presence_test.kf` |
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
* **Domínio próprio (Fase 6, slice 1)**: hostname externo registrado como
  PENDING com instrução CNAME, verificado por `net.resolve` contra os destinos
  esperados do Edge (comparação exata; NXDOMAIN/falha = `KCF124` opaco), rota
  ativa só após VERIFIED; conta alheia recebe o mesmo `404` opaco; poda junto
  com serviço/dispositivo; persistido no estado (`domains_test.kf`, 8 testes).
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
  `_rate_limited` (`ratelimit_test.kf`).
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
* **Daemon user-level (Fase 7, slice 3)**: `cli/Daemon.kf` gera
  `kofcloud-agent.service` (mesmo `connect`, `Restart=always`,
  `NoNewPrivileges`, enable manual); `status`/`uninstall`; erro honesto sem
  systemd; só Linux (`daemon_test.kf`, 8 testes).
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
  **deploy com build/artefato**, **daemon Windows/macOS**, **painel web**.
* **Múltiplos nós** de control plane/edge (sem consenso; registro único em
  memória + arquivo).
* **TLS/HTTPS na borda (Fase 9)** — slices: PEM do operador via
  `listenSecure`, HSTS+301, SNI por hostname; ACME automático bloqueado
  (sem ES256/ACME no stdlib).
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
* **E2EE real (Fase 15)** — corpo opaco para a plataforma (Edge roteia
  sem decifrar); tensão honesta com WAF (por serviço: um ou outro).
* **Arquitetura escalável (Fase 16)** — estado plugável → Edge horizontal
  → control ativo-passivo → células; mesma API, fail-closed, rollback.
* **Resposta a incidentes (Fase 17)** — detecção com severidade → `freeze`
  reversível → status page + webhook assinado + post-mortem; game-day.
* **Integrações de mercado (Fase 18)** — DNS/cert, OIDC/SSO+SCIM, CI/CD
  com `--plan`, OTel/billing, segredos externos; sem lock-in, dry-run.
* **Segurança contínua (Fase 19)** — SBOM + `security.txt`, scan no CI,
  rotação assistida + kill-switch, audit log WORM rumo a SOC2.

Ver `16-plan.md` para as fases e `17-limitations.md` para as lacunas.
