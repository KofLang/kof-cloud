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

**Presença visível no `device list` (parte do contrato de `03-device.md`).**
O doc dizia que "sem heartbeat por 3× o control plane marca o dispositivo
`OFFLINE`" — esse estado não existe: `status` é o ciclo de vida
(`ACTIVE`/`REVOKED`/`SUSPENDED`) e presença é um eixo efêmero decidido pelo
Edge. O `device list` mostrava só o cadastro, então responder "este notebook
está no ar?" exigia cruzar dois comandos. Agora a linha traz
`tunel: ONLINE|OFFLINE|desconhecido (motivo)` via `/v1/presence`, com a mesma
regra do `status`/`doctor`; falha na consulta nunca vira `OFFLINE` inventado.
Prova: `tests/device_presence_test.kf` (2).

**Teto de conexões públicas simultâneas (parte do contrato de `10-edge.md`).**
A tabela prometia "1024 conexões públicas simultâneas → `503` + fecha", mas o
`serveHttp` spawnava uma task por `accept` sem teto: uma rajada de conexões
abertas e ociosas (slowloris) consumia uma task e um socket por conexão até
exaurir a máquina. Agora `EdgeState.tryAcquirePublicConn` reserva a vaga no
`accept` (antes de ler o pedido) e `handlePublicConnection` libera em
`finally` — sem isso, uma exceção no relay vazaria a vaga e o Edge acabaria
recusando todo mundo. Excedeu → `503 KCF117` + métrica
`kofcloud_edge_http_connections_refused_total`. Prova:
`tests/conn_limit_test.kf` (2).

**Profundidade de fila por stream (parte do contrato de `08-multiplexing.md`).**
O desenho prometia "fila cheia → `RESET LIMIT_EXCEEDED` em vez de bloquear a
conexão inteira", mas os canais do Kof são ilimitados (medido: `Channel` só
tem `send`/`receive`) — um stream cujo consumidor estava lento acumulava
memória sem teto, e um serviço local que devolvesse gigabytes derrubava o
Agent inteiro. Agora `TunnelSession` conta os itens pendentes por stream
(`kofcloud.limits.queueDepth`, 64) e `routeData` é o ponto único da política:
fila cheia → `RESET LIMIT_EXCEEDED` naquele stream + inbox fechada, e o
leitor do túnel segue para o próximo frame. Marcadores de controle não contam
para o teto (são o que desbloqueia o consumidor). Prova:
`tests/queue_depth_test.kf` (3).

**Health check HTTP com prazo (parte do contrato de `09-flows.md` §6 e `04-service.md`).**
O check só fazia um connect TCP e `DEGRADED` nunca era emitido: um serviço que
respondia `500` continuava `ONLINE` no Edge. Agora `agent/Health.kf` faz `GET`
de verdade (status < 500 = `ONLINE`, ≥ 500 ou sem resposta no prazo =
`DEGRADED`, recusa = `OFFLINE`); o prazo (`kofcloud.limits.healthTimeoutMs`,
2 s) usa o único mecanismo do Kof para `receive` sem timeout: uma task fecha
a conexão e o receive em voo lança (medido). Prova: `tests/health_test.kf` (2).

## Fase 5 — autenticação

`PUBLIC` / `AUTHENTICATED` / `PRIVATE` no modelo e no Edge. `PRIVATE` já existe
desde a Fase 2 (sem rota); `AUTHENTICATED` adiciona a checagem de token antes do
túnel.

**Estado: ✅ implementado.** Credencial de serviço `kfs_` emitida pelo control
plane (`POST /v1/services/token`) e validada no Edge; o token da conta não
autoriza acesso público; o Edge remove a credencial antes de repassar ao
serviço local; `401` com `WWW-Authenticate: Bearer`. CLI: `kof cloud service
token <nome>` (revogacao por token ou id curto: `kof cloud service token-revoke <nome> <kfs_|id-curto>` → `POST /v1/services/token/revoke`, 404 opaco se alheio, 409 `KCF063` em ambiguidade; inventario: `kof cloud service token-list <nome>` → `POST /v1/services/tokens`, so id curto + emissao, nunca o valor; expiracao opcional `--ttl <horas>`, expirado vira 401 e some do inventario). Prova: `tests/service_token_test.kf` (13 testes) +
`tests/e2e_authenticated_test.kf`.

**Endurecimento (padrão kof-share R6): ✅ implementado.** Erros estáveis
`KCF001`–`KCF106` (`model/Errors.kf`): nunca removidos nem renumerados; 404
opaco entre contas (existente-alheio indistinguível de inexistente);
owner-gate em `POST /v1/services/status` (só a conta dona altera); CLI com
`Hint:` humana (`printFail`); validação de `publish` devolve o HTTP certo por
família (400 validação, 404 dono/recurso, 429 limite). Prova:
`tests/errors_test.kf` (10 testes: códigos do control plane, 404 opaco,
limites/owner-gate no registry, códigos do Edge, Hint + trava de regressao
100% + revoke/TTL + nome|id + fail-fast de flags).

## Fase 6 — domínio próprio

**Estado (slice 1): ✅ implementado.** Registro PENDING com instrução CNAME
(`POST /v1/domains` → `CNAME api.example.com -> site.mel.kofcloud.io`),
verificação por `net.resolve` contra os destinos esperados do Edge (config
`kofcloud.edgeHosts`, comparação exata, NXDOMAIN/falha vira `KCF124`
indistinguível de "resolveu para outro lugar"), rota ativa só depois de
VERIFIED, poda junto com serviço/dispositivo. Dono verificado antes do DNS
(sem trabalho de rede para conta alheia). CLI: `domain add --service`,
`domain verify [--tls-cert <pem>]`, `domain remove`; erros `KCF120`–`KCF124`
com `Hint:`. O `--tls-cert` julga o PEM localmente antes da rede (cobre o
hostname? expira em 30 dias?) — ver Fase 9 slice 3.
Prova: `tests/domains_test.kf` (11 testes: validação, registro/idempotência,
verificação sem rede, rota/poda, API com service preenchido, Edge via túnel real, Hint + serializadores, certificado × hostname puro e com openssl real).

Falta (fatiado de propósito): TLS por SNI no Edge (o Kof não expõe — ver
`12-tls.md`) e automação de DNS por provedor (Cloudflare/Route53). Sem isso,
o hostname próprio serve HTTP pela borda; o HTTPS continua dependendo de
terminador externo.

## Fase 7 — deploy

**Estado (slice 1 — deploy declarativo + slice 5 parcial): ✅ implementado.**
`kofcloud.json` (`--file` para outro caminho) declara `services[]` (nome,
porta numero ou string, address/exposure/protocol/build/artifact opcionais);
`kof cloud deploy` roda o `build` de cada serviço (quando declarado; falhou
= não publica aquele), confere o `artifact` declarado (ausente = não publica
aquele, fail-fast antes do publish) e publica em lote
(idempotente, sem remover o que nao declarou) e `kof cloud down` remove so
os do manifesto (`--prune` poda órfãos do device). Sem artefato transferível
— o build roda local, só o publish atravessa a rede. Parser manual com erro posicional
(Map<String, String> nao aceita numero — ClassCast medido; exposure/protocol/
address validados, sem default silencioso). Prova:
`tests/deploy_test.kf` (9 testes). **Fatia: artefato tem de ser arquivo
legível** (`artifactProblem`): `File().exists()` aceita diretório, então
`"artifact": "dist"` passava a checagem e o deploy publicava um serviço cujo
"artefato" era a pasta. Agora exige conteúdo legível (`readText() != null` —
em diretório volta null); arquivo vazio ainda é artefato. Prova:
`tests/deploy_test.kf` (`deploy-build` estendido).
**Fatia: `build` roda no shell.** A documentação prometia "comando de shell"
(`"build": "npm run build"`), mas `process.run(buildCmd)` recebe o texto
inteiro como NOME de executável — medido, `"echo hi"` falhava com exit -1
(mesma lição do `daemonSystemctl`). Agora `process.run("sh", "-c", buildCmd)`:
argumentos, pipes e `exit` do shell funcionam, e o código de saída do shell é
o resultado. Prova: `tests/deploy_test.kf` (`deploy-build` estendido:
"echo hi", `exit 3`, pipe, comando inexistente). `deploy --check` verifica publicado +
respondendo + artefato (aviso `artefato ausente`, nunca bloqueio; sem
rollback); `deploy --plan` mostra o diff sem aplicar (com `(build)`/
`(artifact)` inclusive no igual).

**Estado (slice 3 — daemon user-level): ✅ implementado.** `kof cloud daemon
install|status|uninstall`: gera `kofcloud-agent.service` (mesmo `connect`,
`Restart=always`, `NoNewPrivileges`, enable manual de proposito), erro
honesto sem systemd; artefato por SO na Fase 13 slice 1 (systemd continua o
padrao no Linux). Prova:
`tests/daemon_test.kf` (8 testes + 5 da Fase 13 slice 1 — ver U28).

**Estado (hot-reload — debito quitado): ✅ implementado.** O Agent re-busca
`services[]` via `/v1/connect` a cada `6×healthMs` e troca a tabela DA SESSAO
(objeto compartilhado — Map nao atravessa `spawn` por valor); streams em voo
nao sao tocados; remocao sai da tabela. Junto, handshake por stream: o Edge
espera OPENED antes de DATA, e OPEN recusado vira 502 em vez de travar o
relay para sempre (buraco real: publicar-depois-do-connect pendurava a vaga).
O ciclo de health lê a TABELA DA SESSÃO a cada rodada (nunca a lista capturada
no connect): serviço removido para de ser sondado e a memória poda o nome;
serviço novo passa a ser sondado no ciclo seguinte (antes, STATUS preso no
UNKNOWN e removido reportado para sempre).
Prova: `tests/reload_test.kf` + OPENED em `tests/protocol_test.kf` + `health-reload`/`health-follows-reload` em `tests/health_test.kf`.

**Estado (presenca real do tunel): ✅ implementado.** O Edge espelha
bind/unbind num mapa do Registry (nao persistido); `GET /v1/presence`
(autenticado, filtrado por conta) lista devices com sessao viva; `kof cloud
status` mostra `Tunel: ONLINE/OFFLINE` confirmado pelo Edge (nao inferido do
processo local) e o `doctor` ganhou a camada do tunel. Prova:
`tests/presence_test.kf` (vivo listado, morto some, 401 sem token, filtro
por conta).

Falta fatiado (Fase 7): build/artefato (passos locais ✅: campo `build`
opcional executado antes do publish, campo `artifact` opcional conferido no
deploy (fail-fast) e no `--check`/doctor (aviso); plano marca `(build)`/
`(artifact)`. Falta: artefato transferível, cache, build remoto),
auto-update. `kof cloud doctor` (somente-leitura: login, control plane,
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
consome túnel). Chave = hostname (o Edge TCP não expõe IP de origem) **já
normalizado** por `hostOnly` — apara, remove a porta, abaixa a caixa e tira
o ponto final de FQDN. Sem isso, `Host: Site...` dava 404 (DNS é
case-insensitive) e cada variação de caixa abria cota própria (100 pedidos
em `site.` + 100 em `SITE.` passavam do 429). Métrica `_rate_limited`,
`Hint:` sem dado de cliente. Prova:
`tests/ratelimit_test.kf` (2 testes: 4o pedido → 429 com/sem túnel, cota
isolada por hostname, Retry-After, métrica, Hint). **Janela em segundos:**
`security.rateLimit` recebe segundos, mas a config é em ms — a conversão vive
num ponto só (`Limits.httpWindowSeconds`, teto no arredondamento, mínimo 1 s,
recuo para o default em valor inválido) e o `Retry-After` deriva dela. Antes
a janela ia em ms direto: o default de 60 s virava 60.000 s (~16,7 h) de
bloqueio por hostname, e 101 pedidos bastavam para tirar um site do ar pelo
resto do dia.

**Estado (slice 2 — suspensão + auditoria): ✅ implementado.**
`SUSPENDED` reversível para serviço (sai da rota, opaco como PRIVATE) e
dispositivo (túnel recusado com `DEVICE_SUSPENDED`, fatal — sem reconexão);
health do Agent e status via API **não** dessuspendem (só `unsuspend`
libera); enforcement imediato no Edge (request derruba a sessão) e contínuo
via PING (cai em até um heartbeat); trilha de auditoria em anel (fonte unica
`limitAuditEvents()`, `kofcloud.limits.auditEvents`, default 100 eventos, só
metadados) com `GET /v1/audit` filtrado por conta + `kof cloud
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

**Estado: slice 1 parcial ✅ (control plane) + resto ⏳.** Desenho fatiado
(cada slice é um incremento testável, na ordem):

* **Slice 1 — HTTPS com PEM do operador ✅ (control plane).** `serve`
  ganha `--tls-cert/--tls-key` (par obrigatório, PEM lido do disco com
  fail-fast); sem flags, HTTP (comportamento atual); com o par,
  `ControlPlane` usa `app.listenSecure(port, cert, key)` (JVM) e o banner
  anuncia `https://`. A CLI fala com o control via `http.*` (trust-all no
  Kof), então `login --control https://…` funciona sem config de trust.
  Escopo honesto: só o control plane (JSON/Strings, sem binário) — o Edge
  público segue HTTP porque o handler do `web.app()` corrompe bytes altos
  (L16 segue valendo). Prova: `tests/control_test.kf` (`control-tls`:
  openssl gera o par, `enableTls` liga, `http.get` lê o `/v1/health` via
  HTTPS) + `tests/hosts_test.kf` (`serve-tls`: paridade, erro de metade,
  esquema do banner, guardas fail-fast).
* **Slice 2 — HSTS + redirecionamento HTTP→HTTPS ✅ (parcial: HSTS).**
  O runtime do Kof já emite `Strict-Transport-Security` em toda resposta
  `web.app()` sob TLS (`Policy.headers=true` por default, gated em
  `req.secure` do SSLSocket — medido em `JvmRuntimeWebDispatch` linha 95 e
  `JvmWebSecurityRuntime` linha 37). Sem código nosso: com o par TLS, o
  control plane já anuncia HSTS; sem TLS, nada é anunciado (anunciar em
  claro seria mentir). Prova: `tests/control_test.kf` (`control-tls`:
  resposta crua via `openssl s_client` tem `max-age=`; via HTTP o header é
  ausente). Tentar `app.use` com `headerSet` seria redundante — e o
  redirecionamento 301 HTTP→HTTPS fica fora (o `serve` sobe um control
  por porta: com TLS não há listener HTTP para redirecionar).
* **Slice 3 — SNI por hostname + renovação assistida (parcial: expiração visível ✅).**
  O `serve` com `--tls-cert/--tls-key` roda `openssl x509 -checkend 30d`
  e avisa (nunca bloqueia) quando o PEM expira em menos de 30 dias;
  openssl ausente/erro = sem aviso. Pura `tlsCertExpiringSoon` (exitCode
  → Bool). Prova: `tests/hosts_test.kf` (`serve-tls` estendido).
  O mesmo julgamento ficou visível fora do `serve`: `kof cloud scan
  --tls-cert` (Fase 19 slice 2) e, agora, `kof cloud doctor --tls-cert`
  — camada `tls` distingue ausente / PEM inválido / expira em 30 dias /
  válido, mostra só o caminho (nunca o conteúdo) e falha o diagnóstico
  quando o certificado não serve (pura `doctorTlsLine`). Prova:
  `tests/doctor_test.kf` (`doctor-tls`, com `openssl` real e pula sem ele).
  **`domain verify` aceita o PEM ✅** (`--tls-cert <pem>`): antes de qualquer
  rede, confere SAN DNS (fallback CN) contra o hostname — curinga de um
  rótulo (RFC 6125), nunca o domínio base — e recusa com `FALHA tls` quando
  o certificado não cobre (DNS certo com cert errado = HTTPS quebrado
  descoberto tarde); expirando em 30 dias vira aviso na linha `ok`, e
  openssl ausente é aviso (nunca recusa inventada). Só caminho e nomes
  aparecem, nunca o conteúdo do PEM. Provas: `tests/domains_test.kf`
  (`domains-cert`: puro + `openssl` real, pula sem ele).
  **`doctor --tls-cert` confere cobertura do host ✅.** A camada TLS só
  olhava formato e validade: um PEM válido de **outro domínio** passava e o
  HTTPS quebrava no handshake, em produção. Agora `doctorTlsLineFor` recebe
  SAN/subject (`openssl x509`) e os hostnames do manifesto
  (`tlsRequiredHosts` = `<servico>.<handle>.kofcloud.io`) e aplica a mesma
  regra de cobertura do `domain verify --tls-cert` (`certHostVerdict`, uma
  regra só): PEM que não cobre vira `FALHA tls`, curinga da conta cobre, e
  sem SAN consultado (openssl ausente) não se inventa falha. Prova:
  `tests/doctor_test.kf` (`doctor-tls`, 3 testes a mais, com PEM real).
  Pendente: SNI por hostname (mapa em memória, recarregado sem derrubar o
  Edge — depende do runtime), renovação automática, e expiração visível
  também no `service status`.
  ACME de verdade (DNS-01) depende de ES256 no stdlib — sem isso, emissão
  continua externa (ver `12-tls.md`).

**Bloqueios honestos:** sem TLS em socket cru (L2) — por isso `listenSecure`
(JVM-only); sem ACME e sem ES256 (só Ed25519) — por isso sem Let's Encrypt
automático ainda. Ver `12-tls.md` e `13-kof-apis.md` L2.

**Edge HTTPS segue bloqueado por medição, não por falta de plano:** o
handler do `web.app()` devolve `String` e corrompe bytes altos (UTF-8:
`0xC8 0xFF` → `C3 88 C3 BF` — `13-kof-apis.md` L16). Um proxy sobre
`web.app()` destruiria respostas binárias. Para destravar o Edge: TLS em
socket cru (capacidade do Kof) ou API de resposta binária no `web.app()`.
Enquanto isso, TLS no Edge segue no terminador externo (`12-tls.md`) —
o que o slice parcial acima resolve é o control plane (login/token).

## Fase 10 — WAF + proteção da aplicação (cloud robusta)

**Motivação:** hoje o Edge roteia sem inspecionar — SQLi/XSS/RCE passam
direto para o serviço local. Cloud robusta filtra o óbvio na borda sem
quebrar o legítimo (fail-open com log vs fail-closed configurável).

**Estado: Fase 10 completa ✅ (slices 1, 2 e 3).**

* **Slice 1-detectar — assinaturas mínimas em modo detectar ✅.**
  `wafDetectSignature` pura (método+path, minúsculas; `../`/`..%2f`/`..%5c`,
  `%00`, `UNION SELECT`, `OR 1=1`, `<script`/`%3cscript`) → nome do padrão;
  o Edge (`HttpFront`) conta por padrão (`_traversal`, `_nullbyte`,
  `_sqli_union`, `_sqli_tautology`, `_xss_script`, `_other`) e segue com o
  relay (nunca bloqueia). `KCF109` existe com Hint (o enforce futuro usa
  esse código). **Corpo pequeno ✅:** `wafDetectBodySignature` inspeciona até
  `wafBodyInspectBytes()` (64 KiB) do corpo — o desenho prometia "corpo
  pequeno" e só método+path era visto; JSON escapado não engana
  (`"q":"x' OR 1=1--"` casa `sqli-tautology`), corpo acima do teto não é
  varrido. Teto fonte-única: `wafBodyInspectBytes()` delega em
  `limitWafBodyInspectBytes()` (`kofcloud.limits.wafBodyInspectBytes`, default
  64 KiB, travado com mutação). **Headers ✅:** `wafDetectHeadersSignature`
  varre os **valores** dos headers do cliente (User-Agent/Referer/Cookie/X-*)
  até o mesmo teto; `wafHeaderIgnored` exclui o que é da plataforma
  (`Authorization` é credencial e nunca é lida, `Host` é rota,
  `Content-Length`/`Connection`/`X-Forwarded-For` são do protocolo). Path,
  corpo e header casam pela **mesma** lista (`wafSignatureOf`), então um padrão
  novo vale nos três. Honestidade: a barra invertida do JSON não é normalizada
  (traversal JSON-escapado não casa no corpo nem no header).
  **Percent/form-encoded ✅ (uma passada):** quando o literal não casa e o
  texto tem `%` ou `+`, `wafSignatureDecoded` compara o **mesmo vocabulário**
  contra o texto decodificado — `%2e%2e%2f`, `..%5C`, `union+select`,
  `%3Cscript`, tautologia form-encoded passam a casar. O matcher usa um
  buffer circular do tamanho da agulha (`wafDecodedContains`): nada de
  reconstruir a string, que em Kof é O(n²) e custaria segundos por corpo de
  64 KiB. `+` vira espaço só em query/corpo (form-encoding real); no valor de
  header só `%NN`. Decodifica **uma** vez de propósito: `%252e` vira `%2e`,
  não `.` — cadeia até estabilizar vira falso positivo e esconde o custo.
  Prova: `tests/waf_detect_test.kf` (`wafencoded-unit` + e2e `wafenforce`
  estendido: traversal/`union+select` encoded dão 403 antes do relay, query
  legítima com `+` e UTF-8 percent seguem 200) + `wafdetect-unit`,
  `wafdetect`, `wafbody-unit`, `wafdetect-body`, `wafheader-unit`.
* **Slice 2a — teto de corpo por requisição ✅.** `kofcloud.limits.httpBodyBytes`
  (default 1 MiB); o Edge recusa antes do relay quando o `Content-Length`
  declarado passa → `413 KCF110` + métrica `_body_too_large`. Não consome
  stream nem toca no Agent; corpo exatamente no teto passa. Corpo sem
  `Content-Length` (transfer-encoding de requisição) é **recusado** com
  `400 KCF118` — ver o slice de enquadramento abaixo, que fechou o buraco.
  Prova: `tests/waf_body_test.kf` (2 testes).
* **Slice 2c — enquadramento do corpo ✅.** O `Content-Length` é a fronteira
  do corpo que o Edge repassa, mas era lido com `parseIntSafe` (Int): valores
  acima de 2^31 davam wrap (`4294967306` → 10, smuggling de 30 bytes) ou
  negativo (`999999999999`, corpo descartado em silêncio). Além disso,
  duplicado divergente e `Transfer-Encoding` passavam adiante. Agora
  `parseContentLength` devolve **Long** + flag `invalid` (duplicado
  divergente, valor não numérico/vazio/com sinal, nome fora da forma canônica,
  `Transfer-Encoding`), o Edge responde `400 KCF118` antes de rotear e o corpo
  relayado é fatiado **exatamente** no declarado. Prova:
  `tests/framing_test.kf` (3: unit, e2e 400/413, relay com corpo mentiroso).
* **Slice 2d — `Host` não ambíguo ✅.** Fechado o outro lado da mesma classe:
  o Edge roteia pelo `Host` e reconstrói o pedido para o serviço local, mas
  com **dois** `Host` ele roteava pelo primeiro e repassava os dois — o
  serviço local via o segundo (`Host: evil.example.com` chegava à aplicação),
  host-header injection. Medido com `parseRequest`/`serializeRequest`.
  `hostIsUnambiguous` (pura) exige exatamente um `Host`, não vazio, sem
  caractere de controle/espaço; senão `400 KCF119` antes de rotear (mesma
  ordem do enquadramento: rate limit primeiro). Prova:
  `tests/framing_test.kf` (`framing-host-unit` + e2e: duplicado/ausente →
  400, aplicação vê um único `Host`; mutação removendo a checagem quebra).
* **Slice 2e — cabeçalhos hop-by-hop ✅.** O Edge reconstruía o pedido com
  todos os headers do cliente: `Keep-Alive`, `Upgrade`, `TE`, `Trailer`,
  `Proxy-Authenticate` e — o caso caro — `Proxy-Authorization` (credencial de
  proxy) chegavam em claro ao serviço local; um token nomeado num
  `Connection: keep-alive, X-Private` também atravessava. Medido com
  `serializeRequest`. Agora `headerIsForwarded` (pura) descarta os
  hop-by-hop do RFC 9110 §7.6.1 (+ `proxy-connection` legado) e os tokens
  nomeados em `Connection` (`connectionTokens`, normalizados); o Edge escreve
  o seu `Connection: close` e `X-Forwarded-For`. Prova:
  `tests/framing_test.kf` (`framing-hop-unit` + e2e no relay: o serviço local
  não vê nenhum deles; mutação reversa quebra os dois).
* **Slice 2f — resposta sanitizada ✅.** O lado simétrico: o relay repassava
  os bytes do serviço local sem tocar, então `Connection: keep-alive`,
  `Keep-Alive`, `Proxy-Authenticate` e tokens nomeados no `Connection` do
  upstream chegavam ao cliente — que era convidado a reusar um socket que o
  Edge fecha (a próxima requisição se perde). Agora `sanitizeResponse` (pura)
  remove os hop-by-hop e os tokens nomeados, escreve o `Connection: close` do
  Edge e preserva status, headers fim-a-fim e corpo byte a byte; resposta sem
  `\r\n\r\n` volta intacta (nada é inventado). Vale também para a resposta
  parcial do timeout. Prova: `tests/framing_test.kf` (`framing-resp-unit` +
  e2e: o cliente vê `connection: close` e nenhum `Keep-Alive`).
* **Slice 2g — resposta chunked decodificada ✅.** `Transfer-Encoding` e
  hop-by-hop, então saía no `sanitizeResponse` — mas o corpo chunked ficava
  como veio: medido, o cliente recebia o **enquadramento** (`5\r\nhello\r\n0`)
  como se fosse conteúdo. O Edge agora detecta o header, decodifica
  (`decodeChunked`, parser tolerante a extensões `;k=v`, com teto de iteração
  e flag `valid`) e entrega o corpo simples; `parseHexSafe` reusa a tabela de
  dígitos do WAF. Corpo chunked malformado entrega o que decodificou (o close
  delimita), nunca o enquadramento. Prova: `tests/framing_test.kf`
  (`framing-chunked-unit` + e2e: GET no serviço local responde chunked e o
  cliente lê `hello world`).
* **Slice 2h — status sem corpo ✅.** `1xx`, `204` e `304` não levam corpo
  (RFC 9110 §6.4.1) e `HEAD` também não, mas o Edge repassava os bytes do
  upstream como vinham: medido, um `204`/`304` com corpo fazia o cliente ler
  aqueles bytes (a conexão fecha, então virariam lixo no fim do stream).
  Agora `statusAllowsBody` (pura, por código + método) descarta o corpo e
  `1xx`/`204` perdem também o `Content-Length` (que não vale para eles),
  enquanto `304`/`HEAD` o mantêm — ali ele descreve a representação do GET.
  `statusCodeOf` lê a linha de status sem parsear header. Prova:
  `tests/framing_test.kf` (`framing-status-unit` + e2e: HEAD devolve 204 com
  corpo no upstream e o cliente recebe corpo vazio).
* **Slice 2i — `Content-Length` da resposta reconciliado ✅.** Medido: um
  upstream com `Content-Length: 2` e 7 bytes de corpo entregava os 7 (o
  excedente vira lixo no fim do stream), e um `Content-Length: 100` com 2
  bytes mantinha o header mentiroso. Agora `sanitizeResponse` corta o corpo no
  declarado e reescreve o header para o tamanho **real** — o Edge fecha por
  EOF de qualquer forma, mas parar de mentir é barato. `304`/`HEAD` ficam de
  fora (o CL descreve a representação do GET) e sem CL o Edge não inventa um.
  Corpo chunked decodificado cai na mesma regra (não ganha TE nem CL falso).
  Prova: `tests/framing_test.kf` (`framing-resp-cl-unit` + e2e: PUT responde
  CL 100 com 2 bytes e o cliente recebe `Content-Length: 2`).
* **Slice 2j — resposta não-HTTP recusada ✅.** O Edge repassava ao cliente
  qualquer byte que o serviço local devolvesse: medido, uma resposta sem linha
  de status (`GARBAGE RESPONSE`) chegava ao cliente como corpo, sem erro
  nenhum. Agora `responseIsParseable` (pura) exige `HTTP/<versão>
  <código 100–599>`; senão o cliente recebe `502 KCF131` (+ métrica
  `_invalid_response`) e os bytes/headers do upstream **não** são ecoados.
  Prova: `tests/framing_test.kf` (`framing-badresp-unit` + e2e: DELETE
  responde lixo → 502, corpo sem vazamento, métrica contada).
* **Slice 2k — linha de requisição validada ✅.** O parser aproveitava as
  partes que existiam: medido, `GE T / HTTP/1.1` virava método `GE` e path
  `T`, e uma linha de STATUS (`HTTP/1.1 200 OK`) era aceita como pedido. Agora
  `requestLineIsValid` (pura) exige exatamente três partes, método `token`
  (sem espaço/controle), path absoluto e versão `HTTP/1.0`/`1.1`; linha
  inválida deixa o método vazio e o `HttpFront` responde `400 KCF132`
  (+ métrica `_invalid_request_line`) antes de rotear. Prova:
  `tests/framing_test.kf` (`framing-reqline-unit` + e2e: quatro partes e
  status-line → 400).
* **Slice 2l — path sem caractere de controle ✅.** Mesma classe da linha:
  medido, `GET /a\rb HTTP/1.1` pós-validação ainda passava (o `\r` não é
  espaço), e ao reconstruir o pedido a linha se partia — o resto virava um
  header para o serviço local (injection). `requestLineIsValid` agora rejeita
  qualquer caractere ≤ 32 ou DEL no path, além de exigir `/` inicial; URI
  absoluta (`http://…`) segue recusada por não começar com `/`. Prova:
  `tests/framing_test.kf` (`framing-reqline-unit` estendido + e2e: CR no
  path → 400 KCF132).
* **Slice 2m — cabeçalhos validados ✅.** Fechada a lista: medido, `X-A\rb: v`
  e a linha sem dois pontos atravessavam até o serviço local, onde a linha se
  partia (header injection). `headerLineIsValid` (pura) exige `Nome: valor`
  com nome `token` (sem espaço/controle) e valor sem CR/LF/NUL;
  `headersAreValid` recusa o pedido inteiro no `HttpFront` com `400 KCF133`
  (+ métrica `_invalid_header_line`) antes de rotear. Prova:
  `tests/framing_test.kf` (`framing-hdrline-unit` + e2e: CR no nome e linha
  sem dois pontos → 400).
* **Slice 2n — 1xx intermediário descartado ✅.** Medido: quando o serviço
  local manda `100 Continue` antes da resposta final, o Edge tratava o bloco
  como a resposta — o cliente recebia `100` com corpo vazio e nunca via o
  `200`/`201`. `sanitizeResponse` agora pula cada bloco `1xx` completo
  (headers + terminador, com teto de 8 iterações) e entrega só a final; bloco
  1xx sem terminador volta como veio (incompleto não é inventado). Prova:
  `tests/framing_test.kf` (`framing-interim-unit` + e2e: PATCH responde
  100 + 200 e o cliente vê 200/`ok`).
* **Slice 2o — query fora das decisões da plataforma ✅.** O comentário do
  front prometia que a query "não muda nada", mas o parser mantém o path
  inteiro: medido, `/.well-known/security.txt?x=1` caía no roteamento e
  devolvia 404. Agora `pathWithoutQuery` (pura) é a fonte única do caminho
  para `security.txt` e allowlist, enquanto o pedido repassado mantém a query
  byte a byte (o serviço local continua vendo o que o cliente mandou). O WAF
  segue com o path **completo**: cortar a query ali cegava o detector para
  traversal/`union select` de query string (regressão pega pelo `wafenforce`
  na suíte completa).
  Prova: `tests/framing_test.kf` (`framing-query-unit`) + `security_test.kf`
  (mesmo path com query → 200).
* **Slice 2p — porta sem estouro de `Int` ✅.** `parseIntSafe` acumula em 32
  bits: medido, `--port 4294968080` virava **784** e era aceito (o serviço
  era publicado na porta errada) e `4294967296` virava 0. Agora
  `parsePortSafe` (pura, `util/Text.kf`) acumula em `Long` e falha assim que
  passa de 65535 — não há valor intermediário que "caiba por acaso" — e é a
  fonte única de publish (CLI, manifesto, API) e das três portas do `serve`
  (`portFlagValue`: ausente usa o default, presente e inválida vira -1 e a
  validação recusa, nunca cai no default em silêncio). Prova:
  `tests/hosts_test.kf` (`serve-ports` estendido) + `tests/errors_test.kf`
  (API: `4294968080`/`4294967296` → 400 KCF043).
* **Slice 2q — inteiros de contagem/TTL sem estouro ✅.** `parseIntSafe` faz
  wrap de 32 bits: medido, `--tail 4294967396` virava **100** e um `ttlHours`
  gigante virava 100 horas — ambos aceitos em silêncio. `parseIntBounded`
  (pura) acumula em `Long` e falha acima do teto; cobre `--tail`, TTL da CLI
  (`validTtlHours`, `service token`, `service rotate`, `token rotate`) e da
  API (emitir/rotacionar, conta e serviço) e os contadores `pending`/`count`.
  Prova: `tests/hosts_test.kf` (`logs-tail`, `ttl-local`) +
  `tests/e2e_authenticated_test.kf` (ttlHours com wrap → 400 KCF062).
* **Slice 2r — `frameInt` sem wrap ✅.** O último caminho de `parseIntSafe`
  no fio: medido, um `HELLO` com `v: 4294967297` virava **1** e passava a
  checagem de versão (um cliente podia forjar compatibilidade), e um
  `heartbeatMs: 4294968796` virava 1500 ms, encurtando o laço do Agent.
  `frameInt` agora usa `parseIntBounded`; fora de `[0, Int.MAX]` devolve 0,
  que os chamadores já tratam como ausente. Prova:
  `tests/protocol_test.kf` (`frame-int`).
* **Slice 2s — `parseStatusLine` sem wrap ✅.** Fecha a mesma família na
  sonda de health do Agent: o código da linha de status passa por
  `parseIntBounded` (teto 599) e fora de 100–599 vira 0, que o `checkService`
  já trata como resposta quebrada (DEGRADED). Prova:
  `tests/health_test.kf` (`health-unit` estendido).
* **Slice 2t — auditoria final de `parseIntSafe` ✅.** Fechada a varredura de
  todos os callers restantes. Entrada externa (CLI/API/rede/fio) já passa por
  `parsePortSafe`/`parseIntBounded`. Sobram três famílias internas, cada uma
  agora com veredito escrito no código: `Session.pending` e
  `EdgeState.serviceConns` (contadores em `Map<String,String>` por L11, só
  escritos pelo próprio código — parse falho devolve 0, que fecha a porta),
  `Registry.expiryProbes` (idem, 0 soma só o lote atual) e o contador
  `revoked` do `token prune` (resposta do control plane), que passou a
  `parseIntBounded` como defesa em profundidade. Trava em
  `tests/hosts_test.kf` (`parseIntSafe` aceita/recusa + os dois parsers com
  teto na mesma teste, para quem for adicionar um caller novo ver o padrão).
* **Slice 2u — trava de unicidade dos identificadores U ✅.** A matriz de
  `docs/15-tests.md` usa `| Unn |` como índice (liga cobertura de teste à
  fase); um duplicado já aconteceu (framing nasceu U51 igual ao login-error
  e só um olho humano pegou). `tests/hosts_test.kf` (`u-matrix`) trava: lê o
  doc versionado e quebra se qualquer `| Unn |` aparecer duas vezes. Prova:
  o teste passa verde e quebra com o duplicado reintroduzido (mutação).
* **Slice 2v — totais do STATUS travados por suite ✅.** A tabela "Suite |
  Arquivo | Testes" do `docs/STATUS.md` é o retrato honesto do que existe, e
  estava defasada: `protocol_test` dizia 11 (arquivo com 12) e
  `metrics_test` 9 (com 10) — a soma da tabela dava 307 contra 309 reais.
  Corrigido e travado por `tests/hosts_test.kf` (`status-counts`): cada linha
  com UM arquivo tem de bater com a contagem de `test "` do fonte (linhas com
  dois arquivos ficam fora por desenho; sanidade exige ≥40 linhas
  conferidas). Cells são parseadas com `indexOf`, nunca `split("|")` — o
  `split` do stdlib trata o argumento como regex e `|` divide por caractere
  (medido; mesma lição do separador `;` em `Registry.expiryProbes`).
* **Slice 2b — timeout de upstream ✅.** `kofcloud.limits.tunnelIdleMs`
  (default 120 s) limita a espera de resposta. `receive` bloqueia e o Kof não
  tem recv com prazo, então o mecanismo é o mesmo do sweeper de idle: uma task
  watchdog entrega `RESET IDLE_TIMEOUT` na fila **do stream** quando passa o
  prazo sem DATA, acordando o relay. Nada chegou → `504 KCF112` + métrica
  `_timeout`; já havia resposta parcial → entrega o parcial (corpo curto,
  nunca 502 com bytes no meio). A vaga de conexão por serviço é liberada.
  Prova: `tests/waf_timeout_test.kf` (2 testes).

Desenho fatiado:

* **Slice 1 — regras básicas por padrão ✅ (detectar + enforce + flag).** Assinaturas
  mínimas em método/path/query/corpo pequeno (traversal `../`, null byte,
  `UNION SELECT`/`OR 1=1` canônicos, `<script` em query): casa → conta
  (detectar ✅) e, com enforce ligado (`serve --waf=enforce` ✅, via
  `EdgeState.setWafEnforce`), `403 KCF109` antes do relay (sem stream, sem
  Agent) + header `X-KofCloud-WAF: blocked`
  + métrica `_waf_denied` + log só com padrão+hostname (nunca corpo).
  Default **detectar** (conta e segue, resposta marcada com
  `X-KofCloud-WAF: would-block` ✅).
  Prova do detectar: `tests/waf_detect_test.kf` (unit das assinaturas + e2e:
  ataque 200 contado por padrão e marcado, legítimo 200 sem contar nem
  marcar). Prova do enforce-núcleo: e2e `wafenforce` (ataque 403 KCF109 +
  `blocked` sem chegar ao local, legítimo 200 antes e depois, métricas
  `_traversal` + `_waf_denied`). Prova da flag: `serve-waf`
  (`resolveWafMode` pura + guardas fail-fast no `cmdServe`).
* **Slice 2 — limites de corpo/tempo por rota.** Teto de request body por
  serviço (`kofcloud.limits.httpBodyBytes`, default 1 MiB; excedeu →
  `413 KCF110`) + timeout de upstream (502 após N s, sem pendurar o stream).
* **Slice 3 — allowlist por serviço ✅.** `allowMethods`/`allowPaths` (CSV)
  opcionais no `POST /v1/services`; persistidos no `Service` e aplicados no
  Edge **antes do relay**: fora da lista → `403 KCF111` +
  `X-KofCloud-WAF: blocked` + métrica `_waf_allow_denied` (sem stream, sem
  Agent). Fail-closed só para quem declarou — lista vazia = sem restrição
  (serviço legado passa). Política malformada não publica: `400 KCF113`
  (método: só verbos exatos em maiúsculas) / `400 KCF114` (path: absoluto
  `/...`, sem espaço). Prova: `tests/waf_allow_test.kf` (4 testes:
  normalização de métodos e paths, `splitList`, e2e — 403 por path e por
  método, path permitido 200, serviço sem política não barra). O deploy
  declarativo cobre os campos: `kofcloud.json` aceita `allowMethods`/
  `allowPaths` (mesma normalização da API, malformado falha no parse),
  `cmdDeploy` envia no publish, e `diffDeployment`/`verifyDeployment`
  (`--plan`/`--check`/`doctor`) acusam drift de política.

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

## Fase 12 — gestão da nuvem (web client / painel)

**Motivação:** hoje tudo é CLI + JSON. Operar conta/dispositivos/serviços/
domínios/tokens pelo terminal afasta quem não é técnico — e gestão visual
é o que transforma "túnel" em "nuvem" na cabeça do usuário.

**Rate limit no login ✅.** O Edge limitava requisições por hostname, mas o
  LOGIN acontece no control plane, que não tinha teto nenhum: uma senha podia
  ser testada sem limite por quem alcança o control. Agora `handleLogin`
  conta tentativas **por handle** (`kofcloud.limits.loginAttempts`, default
  10 / `loginWindowMs` 60 s; sonda em memória, efêmera como `edgeOnline`) e,
  acima do teto, responde `429 KCF134` + `retryAfterSec` **antes** de
  verificar a senha (não gasta CPU de hash para o atacante). Um acerto reseta
  a janela, então usuário legítimo que errou a senha não fica travado pelo
  próprio sucesso. A chave é o handle porque o control plane não vê o IP de
  origem em `web.app()` — e força bruta dirigida a uma conta é o ataque que
  importa aqui. Prova: `tests/login_limit_test.kf` (2).

**Estado: slices 1 e 2 ✅; slice 3 pendente.** Desenho fatiado (cada slice é
um incremento testável; o painel consome só a API pública `/v1` — nenhum
endpoint novo privilegiado, mesma auth `X-KofCloud-Token`).

* **Slice 1 — painel somente-leitura ✅.** `serve --panel` liga `GET /panel`
  no control plane (`control/Panel.kf`): uma página estática (HTML + `fetch`,
  sem framework nem `<script src>`), que pede o `kfc_` da conta e renderiza
  devices (com túnel ONLINE/OFFLINE via `/v1/presence`), serviços com
  hostname/exposição/status e domínios com status. O HTML **não contém
  token**, o token vive só na memória da aba (não é persistido) e a API
  continua exigindo auth — sem a flag a rota nem existe (`404`). Prova:
  `tests/panel_test.kf` (3 testes).

* **Slice 2 — gestão completa.** Criar conta, registrar device, publish/
  unpublish, add/verify/remove domínio, emitir/revogar/listar `kfs_`,
  suspend/unsuspend — tudo pelo painel, chamando os mesmos endpoints da
  CLI. Cada ação pede confirmação quando destrutiva (revoke/remove).
  Prova: e2e via HTTP do painel (publica e remove de verdade).

  **Implementado ✅.** O painel (`serve --panel`) ganhou formulários que
  chamam as MESMAS rotas `/v1` da CLI: publicar, remover, suspender/reativar
  serviço, emitir e revogar `kfs_`, adicionar/verificar/remover domínio,
  suspender/reativar/revogar dispositivo. Destrutivas (remover serviço,
  remover domínio, revogar dispositivo, revogar todas as credenciais) pedem
  `confirm()`. O `kfs_` aparece **uma única vez** na emissão e nunca em
  listagem; o token da conta continua só na memória da aba. Prova:
  `tests/panel_actions_test.kf` (3 — o HTML expõe cada ação, o e2e
  publica/suspende/emite/remove pelas rotas que o painel usa, e o contrato do
  "Adicionar domínio").

  **Bug corrigido (device do publish).** O painel publicava sempre no
  **primeiro** dispositivo da lista (`STATE.devs[0]`), sem escolha: com mais
  de uma máquina na conta, o serviço ia para a máquina errada e o operador só
  descobria depois. Agora há um `<select id="svcdev">` populado em
  `preencherDevices` (opções via DOM, porque o nome do device é dado do
  usuário), a escolha é preservada ao recarregar, e publicar sem escolher
  responde `escolha o dispositivo`. Prova: `tests/panel_actions_test.kf`
  (`panel-actions-unit` + `panel-actions`, 2 a mais).

  **Bug corrigido (contrato do domínio).** O botão "Adicionar domínio" enviava
  `serviceId`, mas `POST /v1/domains` lê `service` (o **nome** do serviço) —
  todo domínio adicionado pelo painel devolvia `404 KCF122` e o botão nunca
  funcionava. Agora manda o nome, e o teste trava o contrato nos dois lados
  (HTML e rota). Junto: `carregarListas` passou a recarregar dispositivos e
  presença, então suspender/revogar pelo painel atualiza a tabela em vez de
  deixar a linha velha na tela.
* **Slice 3 — onboarding guiado.** Criar conta → baixar desktop client →
  publish do primeiro serviço em 3 telas, com diagnóstico embutido
  (reaproveita a lógica do `doctor`, camada por camada). Prova: fluxo
  completo num control plane de teste.

**Regras:** o painel nunca mostra `kfs_` depois da emissão (mesma regra da
CLI); nunca loga corpo/token; é estático + `fetch`, sem framework pesado
(o Kof serve arquivo + JSON; o resto é HTML/JS simples).

## Fase 13 — desktop client (daemon que sincroniza com a nuvem)

**Motivação:** hoje o "client" é `kof cloud connect` em foreground ou uma
unit systemd manual no Linux. Usuário real quer instalar, logar uma vez e
esquecer — com tray, reconexão visível e update assistido.

**Estado: slice 1 ✅; slices 2 e 3 pendentes (base: daemon user-level + doctor).**
Desenho fatiado:

* **Slice 1 — instalador por SO ✅.** `daemonKind()` (config
  `kofcloud.daemonKind` ou detecção do SO) escolhe o artefato:
  `daemonUnitText` (systemd), `daemonLaunchdPlist` (launchd, `KeepAlive` +
  `RunAtLoad`) e `daemonWindowsScript` (`sc create ... start= auto`). O
  `daemon install` grava no caminho do SO (`daemonArtifactPath`) e imprime a
  ativação (`daemonActivateHint`) — **nunca** ativa sozinho; no Windows o
  script pede Administrador. Prova: `tests/daemon_test.kf` (`daemon-os`, 4
  testes puros, sem instalar nada).
  **Bug corrigido junto:** `daemonSystemctl` chamava
  `process.run("systemctl --user …")` como string única — o Kof não faz split
  de shell, então a chamada procurava um executável com esse nome e falhava
  sempre (medido: `Exec failed, error 2`). O `daemon-reload` nunca acontecia.
  Agora usa argv variádico (`process.run("systemctl", "--user", sub, arg)`),
  com regressão em `daemon-argv`.
* **Slice 2 — tray + estado visível.** Ícone mostra ONLINE/OFFLINE
  (lê `/v1/presence`, igual ao `status`), serviços publicados, botão
  connect/disconnect, link para o painel. Sem o tray o usuário não sabe
  se está no ar — hoje só o `status` diz.
* **Slice 3 — sincronização + auto-update.** O client observa o manifesto
  local (`kofcloud.json`): mudou → re-publica sozinho (usa a mesma
  idempotência do `deploy`); nova versão do Agent → baixa, verifica
  assinatura Ed25519 (chave embutida, nunca baixada junto) e troca no
  próximo restart. Prova: e2e de update com assinatura válida/inválida.

**Regras:** enable/start sempre com consentimento explícito (como hoje);
nunca auto-update silencioso sem assinatura válida; segredos ficam no
keychain do SO quando houver (fallback: `~/.kofcloud` com permissão
best-effort, como hoje).

## Fase 14 — baixa latência (cloud rápida de verdade)

**Motivação:** hoje há UM Edge (`serve` único) e relay em chunks de 16 KiB
com `bytesToString` O(n²) no caminho. Funciona, mas latência é distância +
custo por byte — cloud rápida precisa dos dois endereçados com medição.

**Estado: slice 1 ✅ (latência + tamanho); slice 2 ✅ (ranking de Edges); slice 3 pendente.**
Desenho fatiado (cada slice com benchmark antes/depois, mesmo protocolo,
sem mudar formato público):

* **Slice 1 — medir de verdade.** Benchmark contínuo no Edge: latência
  p50/p99 por hostname (agregado, sem dado de cliente), tempo de relay
  por tamanho, custo do chunk. Sem medir, otimizar é chute. Prova: nova
  seção no `/v1/metrics` + teste de formato.

  **Implementado (distribuição):** o histograma do stdlib só exporta
  `sum`/`count` (média), e média esconde cauda — que é o que pesa. Cada
  requisição agora também cai em uma faixa monotônica de contadores
  (`le_50`…`le_2000` + `gt_2000`), sem bucket por hostname/cliente. Fonte
  única `latencyBucketLimitsMs()`; o rótulo é **derivado** da lista (mudar
  a lista muda o rótulo, sem dois lugares para manter). Negativo (relógio
  ajustado) cai na primeira faixa. Prova: `tests/metrics_test.kf`
  (`metrics-latency`). **Pendente:** p50/p99 por hostname (hoje só agregado
  global, porque contar por hostname seria cardinalidade por cliente) e o
  tempo de relay por tamanho.

  **Implementado (tamanho da resposta):** faixas em KiB
  (`responseSizeBucketLimitsKb`: 1, 4, 16, 64, 256, 1024 + excedente),
  contadas em `noteHttpFinished` junto dos bytes totais — é o "tempo de relay
  por tamanho" na forma medível: cruzar a faixa de latência com a de tamanho
  diz se o p99 vem de muitas trocas pequenas ou de poucas grandes. Resposta
  zero não entra (não é relay). Rótulo derivado da lista (mesma regra da
  latência). Prova: `tests/metrics_test.kf` (`metrics-size`).
* **Slice 2 — relay regional.** `connect` passa a receber a lista de Edges
  (`edgeHost` vira ranking por latência medida no handshake); o Agent
  escolhe o mais próximo e reavalia a cada N minutos. Sem consenso
  distribuído no MVP (cada Edge continua independente; o control plane
  só publica a lista). Prova: e2e com 2 Edges, Agent migra para o mais
  rápido.

  **Implementado ✅ (ranking, sem consenso).** `POST /v1/connect` passa a
  devolver `edges` (o principal + `kofcloud.edgeHosts`, CSV `host:porta`);
  o Agent mede cada um com connect+close (`probeEdges`) e escolhe o de menor
  latência (`chooseFastestEdge`, pura; quem não respondeu é ignorado e, se
  nenhum responder, fica o principal). Override explícito de host/porta
  (testes/operador) vence a escolha automática. O contrato antigo
  (`edgeHost`/`edgePort`) continua. **Pendente:** reavaliar a cada N minutos
  (hoje a escolha acontece em cada `connect`, que já é o ciclo de reconexão)
  e o Edge dedicado por conta.
* **Slice 3 — caminho de bytes O(n).** Tirar `bytesToString`/`base64` via
  String do caminho quente: chunks maiores (64 KiB) + cópia direta de
  bytes quando o stdlib permitir; handshake comprime `services[]`
  (hoje vai cheio a cada `connect`). Prova: benchmark mostra queda
  linear no tempo por MiB.

  **Implementado (parte sem stdlib nova): resposta em fluxo + teto honesto.**
  Dois buracos reais fechados, ambos medidos com teste que falha sem a
  correção:

  1. **Truncamento silencioso.** Quando a resposta passava do `maxBytes` do
     relay, o Edge apenas parava de acumular e devolvia os bytes parciais
     como se fossem a resposta inteira — o cliente recebia **200 com corpo
     cortado**, sem sinal. Agora o Edge cancela o stream (`RESET
     STREAM_TOO_LARGE`), descarta o parcial e responde **502 KCF115**; a
     vaga de conexão é liberada e a métrica
     `kofcloud_edge_http_upstream_errors_total_response_too_large` conta a
     recusa. Teto próprio (`kofcloud.maxResponseBytes`, default 16 MiB) em
     vez de herdar o teto do corpo da requisição.
  2. **Agent acumulava a resposta inteira antes de enviar.** Sem isso, o
     cancelamento do Edge não tinha efeito: o Agent continuava lendo o
     serviço local e empurrando bytes que seriam descartados. Agora cada
     bloco lido vira DATA na hora e o Agent consulta `isCancelled` a cada
     bloco — o RESET do Edge para a produção de verdade. Erro local depois
     de já haver bytes no fio vira `RESET LOCAL_ERROR` (não dá para trocar
     por uma resposta de erro no meio de um 200).
  3. **Teto de cabeçalho que não existia.** `readRequestBytes(conn,
     maxBytes)` prometia limitar o pedido, mas sem `\r\n\r\n` o bloco
     inteiro virava "cabeçalho" e seguia adiante. Agora é **431 KCF116**
     antes de rotear.
  4. **Teto de leitura que não cobria o corpo.** O laço do corpo só olhava
     o `Content-Length` declarado — um cliente mandava um valor gigante e o
     Edge acumulava tudo antes do 413 de política. Agora `maxBytes` vale
     para o pedido inteiro e a recusa (**413 KCF110**) sai sem drenar o
     corpo; tanto declarado acima do teto quanto bytes reais acima dele
     (declarado mentiroso) param na leitura.

  Prova: `tests/relay_limit_test.kf` (4: código/Hint, 502 KCF115 fim a fim
  com vaga liberada e métrica, 431 KCF116, 413 de leitura com `Content-Length`
  acima e abaixo do real). **Pendente:** o ganho O(n) de verdade
  (base64/slice sobre bytes no stdlib) e o handshake comprimido.

**Teto da sonda de health na fonte unica.** `limitHealthHeadBytes()` era o
ultimo literal numerico do caminho de health (16384 em `agent/Health.kf`);
agora delega em `limitHealthResponseHeadBytes()` (`util/Limits.kf`,
`kofcloud.limits.healthHeadBytes`, recuo seguro). O teto do Edge para
conexao publica continua outro (`maxBytes`) — sao caminhos distintos. O anel
de incidentes seguiu a mesma regra (`limitIncidents` → `limitIncidentRing`,
`kofcloud.limits.incidents`). Prova: `health-unit` e `incident-unit` (valor
+ delegacao, mutacao quebra).

**Regra:** nenhuma otimização sem número antes/depois no `STATUS.md`;
latência de cauda (p99) vale mais que média.

## Fase 15 — E2EE real (nem a plataforma lê)

**Motivação:** hoje o túnel cifra Agent↔Edge (`sessionKey` por sessão),
mas o Edge **decifra para rotear** — ele vê o Host e o corpo em claro.
E2EE real significa: a plataforma nunca tem a chave do conteúdo, mesmo
comprometida. É o que protege a máquina física e o site hospedado de
verdade (inclusive da própria operadora da nuvem).

**Estado: slice 1 ✅; slices 2 e 3 pendentes.** Modelo honesto, em camadas
(cada slice reduz o que a plataforma vê, sem quebrar o anterior):

* **Slice 1 — declarar o modelo ✅.** `docs/20-visibility.md`: tabela
  quem-vê-o-quê (cliente/Edge/túnel/Agent/serviço) medida contra o código
  (`edge/Relay.kf` decifra, `edge/Http.kf` remove credencial, tokens só
  como hash) + definição de E2EE (corpo opaco, Host visível) + tensão
  honesta WAF×E2EE. Sem código — alinhamento.
* **Slice 2 — Host blindado, corpo opaco.** O `Host` continua visível
  (roteamento exige), mas o corpo passa a viajar cifrado de ponta a
  ponta (chave por serviço, distribuída via control plane autenticado;
  o Edge repassa bytes opacos sem decifrar). Prova: e2e onde o Edge
  serve sem conseguir ler o corpo (teste inspeciona memória do Edge).
* **Slice 3 — SNI/hostname mínimo.** Com TLS na borda (Fase 9), o SNI
  vaza o hostname — mitigar com ECH quando o stdlib permitir, ou
  documentar como resíduo aceito. Alvo final: plataforma vê metadados
  de roteamento, nunca conteúdo.

**Limites honestos:** E2EE total (nem o Host visível) exigiria routing
oblivious — fora do escopo; o compromisso é **conteúdo opaco para a
plataforma**, com metadados mínimos documentados. WAF (Fase 10) e E2EE
são tensão direta: inspecionar corpo exige ver corpo — por serviço, o
operador escolhe (WAF **ou** E2EE estrito, nunca os dois fingindo).

## Fase 16 — arquitetura robusta e escalável

**Motivação:** hoje é um nó de control plane + um Edge, estado em JSON
em memória + arquivo. Isso prova o fluxo, mas é ponto único de falha e
teto de escala. Cloud robusta separa o que escala junto do que não
escala — sem reescrever o núcleo (router + túnel + streams continuam
iguais).

**Estado: slice 1 parcial ✅ (carga explícita + migrate --dry-run); resto ⏳.** Desenho fatiado:

* **Slice 1 — separar estado do processo.** Registry atrás de interface
  (hoje a classe já concentra tudo): backend arquivo (atual) → backend
  SQLite/Postgres por config. Migração com `kof cloud migrate --dry-run`
  (lê, valida, escreve, compara contagem) antes de trocar. Prova: suíte
  roda igual nos dois backends.

  **Implementado (parte sem banco):** a carga do estado passou a ser
  **explícita** — `Registry.loadStatus` = `ok` | `absent` | `corrupt`, e o
  `serve` anuncia em voz alta quando o arquivo existe mas não decodifica
  (antes virava control plane vazio em silêncio, com a conta "sumida" sem
  aviso). A validação de forma é explícita (`looksLikeControlState`): o
  `json.decode` aceita chaves extras e devolve listas vazias, então um JSON
  qualquer "decodificava" como estado vazio. `kof cloud migrate --from
  <estado.json> [--to <destino>] [--dry-run]` lê, valida, compara contagens
  por lista (`StateCounts`/`diffCounts`, puros) e só escreve sem `--dry-run`;
  origem ilegível é recusada com exit 1 e nada é escrito. Prova:
  `tests/migrate_test.kf` (5). `kof cloud state-backup [--file <estado>]`
  preserva o arquivo em disco ao lado (`.bak-<ms>`) antes de qualquer troca
  de backend, e recusa origem ilegível/ausente em vez de copiar lixo.
  **Pendente:** backend SQLite/Postgres atrás de
  interface (o `kof.db` existe, mas o desenho pede a interface primeiro).
* **Slice 2 — Edge horizontal.** N Edges atrás de qualquercast/DNS-round-
  robin; presença vira pub/sub entre Edges (o `/v1/presence` agrega, com
  TTL curto por entrada); rate limit passa a janela deslizante em store
  compartilhado (Redis ou o próprio control plane). Sem afinidade:
  qualquer request pode cair em qualquer Edge. Prova: e2e com 2 Edges.
* **Slice 3 — control plane ativo-passivo.** Líder eleito por lease no
  store (quem tem o lease escreve; o outro serve leitura + fila). Split-
  brain é pior que indisponibilidade curta: em dúvida, o passivo recusa
  escrita (fail-closed, 503 + `Retry-After`).
* **Slice 4 — células e blast radius.** Contas distribuídas em células
  (shards por faixa de `accountId`); incidente numa célula não derruba as
  outras; `status`/`doctor` mostram a célula (transparência, não detalhe
  interno). Prova: teste desliga uma célula, a outra serve.

**Regras:** mesma API `/v1` em todos os slices (escala não muda contrato);
estado nunca em memória sem dono (todo mapa efêmero tem TTL + prova de
expiração, como já faz `serviceTokens`/`edgeOnline`); deploy multi-nó
sempre com rollback de um comando.

## Fase 17 — resposta a incidentes (cloud que se defende e se explica)

**Motivação:** segurança real não é só prevenir — é detectar, conter,
comunicar e aprender. Hoje há auditoria em anel (100 eventos) e métricas;
falta o ciclo completo: do sinal ao post-mortem.

**Estado: Fase 17 completa ✅ (slices 1, 2 e 3).** Desenho fatiado (cada slice exercitado em
game-day: simula o incidente de propósito e mede o tempo de resposta):

* **Slice 1 — detecção e severidade.** Sinais com nível
  (`INFO/WARN/CRIT`): pico de 401/429, \u200btúnel caindo em massa, revoke em lote,
  expiração em massa de `kfs_`, Edge sem heartbeat. `GET /v1/incidents`
  lista os abertos (filtrado por conta quando for dado de cliente, global
  quando for plataforma). Prova: teste injeta o sinal, incidente abre.

  **Implementado (parcial em sinais):** `model/Incident.kf` (INFO/WARN/CRIT,
  vocabulário fechado). Detector por conta: ≥3 `device.revoked` ou
  `service.token_revoked` em 60 s (do anel de auditoria) → `CRIT`/`WARN`,
  idempotente por (sinal, conta), reabre depois de resolvido. Detector global
  do Edge: ≥10 recusas de túnel na janela → `edge_refusals` (WARN).
  `POST /v1/incidents` lista (conta + globais; `all` = só globais) e
  `resolveId` fecha (alheio → `404 KCF125`). CLI: `kof cloud incidents
  [--all] [--resolve <id>]`.

  **Implementado (abuse prevention — rajada de 401):** o front já responde
  `401` a credencial ausente/errada em serviço `AUTHENTICATED`; agora esse
  caminho alimenta uma sonda própria (`RateProbe` no `EdgeState`, separada da
  de recusas de túnel) e ≥25 falhas em 60 s abrem `edge_auth_failures`
  (WARN, global). Limiar mais alto de propósito: cliente honesto com `kfs_`
  vencido também gera 401. A sonda **não** guarda hostname, token nem
  caminho — o incidente é agregado. As duas sondas compartilham
  `probeHit` (reset+incremento num lugar só), então uma não pode esquecer a
  janela. Limiar/janela em `util/Limits.kf`
  (`kofcloud.limits.edgeAuthFailureThreshold/WindowMs`, recuo seguro).
  Prova: `tests/incidents_test.kf` (`incident-unit` + `incident-auth` puro +
  `incident-auth-e2e` com Edge real e token errado).

  **Implementado (Edge sem heartbeat — quedas por ociosidade):** o sweeper
  que já derrubava sessão muda (só PING) agora alimenta uma terceira sonda
  (`idleTimeoutProbe`, separada das outras duas): ≥5 quedas por ociosidade
  em 300 s abrem `edge_idle_timeouts` (WARN, global). Uma queda isolada é
  rotina (Agent parado, notebook suspenso); a sequência é o sinal de Agent
  preso em reconexão. O incidente não carrega device nem hostname. Limiar e
  janela em `util/Limits.kf` (`kofcloud.limits.edgeIdleTimeoutThreshold/
  WindowMs`, recuo seguro). Prova: `incident-idle` (puro: limiar, isolada
  não abre, sondas independentes, janela vencida) e `idle-e2e` estendido (o
  sweeper real derruba a sessão e abre o global).

  **Implementado (expiração em massa de `kfs_`):** a varredura preguiçosa de
  `Registry.serviceByServiceToken` (que já podava credencial vencida) passou
  a atribuir cada vencimento à conta dona e a alimentar uma sonda por conta
  (`expiryProbes`, janela deslizante em memória — não persistida, como
  `edgeOnline`): ≥3 vencidas na janela (300 s) abrem `mass_token_expiry`
  (WARN, escopo da conta). Vencimento isolado é o fluxo normal da rotação por
  TTL; vários juntos significam rotação quebrada (TTL curto sem re-emissão)
  ou estado adulterado. A varredura roda **depois** da poda para o incidente
  nunca atrasar a autenticação que veio antes; o resumo não carrega token,
  serviço nem hostname. Limiar/janela em `util/Limits.kf`
  (`kofcloud.limits.massTokenExpiryThreshold/WindowMs`, recuo seguro).
  Prova: `incident-expiry` (puro: limiar, janela vencida, contas isoladas) e
  `incident-expiry-sweep` (varredura real: atribuição à conta certa, sem
  misturar contas). **Pendente (deliberadamente):** pico de 429 por hostname
  — o rate limit já conta por hostname, mas abrir incidente por hostname
  violaria a regra de não ter cardinalidade por cliente.

* **Slice 2 — contenção em um comando.** `kof cloud freeze <alvo>`
  (conta|device|service|edge): congela tudo do alvo (suspend + revoga
  tokens vivos + derruba sessões) e gera ID de incidente; `unfreeze`
  reverte passo a passo com confirmação. Congelar é reversível por
  desenho — destruir nunca é automático. Prova: e2e congela e
  descongela sem perder configuração.

  **Implementado (service|device):** `Registry.freezeService/freezeDevice`
  suspendem o alvo, revogam as credenciais vivas e abrem o incidente
  (`freeze_service`/`freeze_device`, WARN); o status anterior fica em memória
  (`freezeMarks`) e `unfreezeService/unfreezeDevice` restauram exatamente
  ele — configuração (serviços, domínios, allowlist) nunca é tocada.
  `POST /v1/freeze` e `/v1/unfreeze` (`{target,id}`; alvo alheio/inexistente
  → `404 KCF128` opaco; `KCF126`/`KCF127` para entrada inválida). CLI:
  `kof cloud freeze|unfreeze service|device <nome|id>`.

  **Implementado (account):** `freezeAccount`/`unfreezeAccount` congelam
  todos os devices e serviços da conta e revogam todas as `kfs_` dela
  (incidente `freeze_account`, WARN); o alvo é sempre a conta do token
  (`{"target":"account"}`, sem `id` — id de terceiro nunca escolhe conta).
  A marca de status anterior é **por alvo** e só é gravada se ainda não
  existir: um servidor que já estava `SUSPENDED` volta `SUSPENDED`, e um
  segundo freeze não sobrescreve o estado original (reversibilidade; o
  mesmo guarda foi aplicado a `freezeService`/`freezeDevice`). Prova:
  `tests/freeze_test.kf` (`freeze-unit` ×2 + HTTP). **Pendente:** alvo
  `edge` (exige coordenação multi-Edge, Fase 16).
* **Slice 3 — comunicação e post-mortem.** Status page mínima (lê os
  incidentes globais; sem login), notificação por webhook configurável
  (conta cadastra URL + segredo; payload assinado Ed25519), e template
  de post-mortem no repo (`docs/incidents/AAAA-MM-DD-titulo.md`: causa,
  impacto, timeline, correção, ação preventiva com dono e prazo).
  Prova: game-day completo fecha com post-mortem publicado.


  **Implementado (status page):** `control/StatusPage.kf` serve
  `GET /status` (HTML, auto-refresh 30 s) e `GET /status.json` quando
  `serve --status-page` está ligado — **sem login nem token**, e **só com
  incidentes globais de plataforma** (filtro de defesa em profundidade no
  próprio renderer; nada de dado de conta). Estado geral derivado dos abertos
  (`operational` / `partial_outage` / `major_outage`), timestamps ISO-8601
  UTC, `kind` interno não vaza, texto sempre escapado (HTML e JSON). Prova:
  `tests/status_page_test.kf` (5). **Implementado (template):**
  `docs/incidents/README.md` — contrato do post-mortem (causa, impacto,
  timeline, correção, ação preventiva com dono e prazo; sem dado de
  cliente), com trava de regressão (`postmortem-template`). **Pendente:**
  webhook assinado Ed25519.

  **Implementado (webhook):** `model/Webhook.kf` + `Registry.setWebhook/
  removeWebhook/deliverWebhook` + `POST /v1/webhook` (set|show|remove|test) e
  CLI `kof cloud webhook`. Quando um incidente **da conta** abre, o control
  plane faz POST do evento com `X-KofCloud-Signature: sha256=<hex>` (HMAC
  SHA-256 do corpo com o segredo compartilhado, comparado em tempo constante
  no receptor). URL https (http só em loopback, sem credencial na URL) e
  segredo ≥16 chars validados; o segredo **nunca** volta pela API/CLI (só o
  prefixo). Falha de entrega não impede o incidente de existir. **Divergência
  registrada:** o desenho pedia Ed25519, mas o stdlib não expõe par Ed25519
  utilizável (SECN014, `13-kof-apis.md` L17). Prova:
  `tests/webhook_test.kf` (5).
**Regras:** contenção sempre reversível; comunicação sempre antes do
fix secreto (avisa o impacto, depois corrige); todo incidente vira teste
novo (se não tem regressão, não fechou).

## Fase 18 — integrações com o mercado (plugar sem prender)

**Motivação:** ninguém vive isolado — DNS, certificados, alertas, CI/CD,
identidade e billing moram em ferramentas que o operador já usa. Cloud
robusta pluga por padrão aberto, nunca por SDK proprietário.

**Estado: ⏳ não iniciado.** Princípio único: **a KofCloud fala o protocolo
do mercado, não o contrário.** Fatiado por conector (cada um: config por
env, credencial nunca em log, teste com fake local + e2e opcional real):

* **Slice 1 — DNS e certificados.** Provedores DNS (Cloudflare, Route53,
  outros via webhook genérico) para criar o CNAME do domínio próprio
  (`domain add --dns=cloudflare`); ACME DNS-01 quando ES256 existir
  (Fase 9 slice 3). Credencial do provedor em env, escopo mínimo
  (só zona do domínio).
* **Slice 2 — identidade e SSO.** OIDC genérico (qualquer IdP:
  Google, GitHub, Keycloak): `login` abre browser, troca code por conta
  vinculada (nunca substitui `kfc_` — vira segundo fator de emissão).
  SCIM mínimo para times (provisionar/revogar conta em lote).
* **Slice 3 — CI/CD e GitOps.** Action oficial (`kofcloud/deploy@v1`):
  lê `kofcloud.json`, publica, roda `--check`, falha o pipeline se
  quebrar; `kof cloud deploy --plan` mostra o diff sem aplicar (como
  `terraform plan`). Prova: pipeline de exemplo no repo.
* **Slice 4 — observabilidade e billing.** Export OpenTelemetry
  (traces do relay + logs estruturados) para o backend que o operador
  já tem (Grafana/Datadog); webhook de uso (requests/bytes por conta)
  para billing externo; Stripe como referência (nunca único).
* **Slice 5 — segredos e chaves.** Provedores de segredo (Vault, AWS SM,
  1Password Connect) como fonte do `tunnelSecret`/PEMs: a plataforma lê
  na partida, nunca persiste em disco; rotação por SIGHUP/reload.

**Regras:** todo conector é opcional e sem lock-in (o caminho manual
continua funcionando); credencial de terceiro nunca atravessa o túnel
nem aparece em log/métrica/audit; cada conector tem modo `--dry-run`.

## Fase 19 — segurança profunda e contínua (cloud auditável)

**Motivação:** juntar o que as Fases 9–11 e 17 começaram num programa
contínuo: a segurança não é uma fase que "termina", é um ritmo (scan,
revisão, game-day, auditoria). Esta fase é o metrônomo.

**Estado: slices 1–3 ✅ (parciais); slice 4 parcial (cadeia de hash) ✅.** Desenho:

* **Slice 1 — baseline e SBOM (parcial: `security.txt` + `SECURITY.md` + `audit --sbom` ✅).**
  O Edge serve `/.well-known/security.txt` (200 `text/plain`, corpo
  estático Contact + Policy, sem dado operacional) **antes** de rate
  limit/roteamento/auth — sem conta, sem túnel, sem tocar em serviço
  local (scanner legítimo não é contido); `docs/SECURITY.md` declara
  suporte/reporte/SLA/escopo com trava de regressão; `audit --sbom`
  imprime o inventário local (48 fontes + SHA-256, sem login, sem rede).
  Prova: `tests/security_test.kf` (`security-unit` + `security` +
  `security-policy`) + `tests/hosts_test.kf` (`audit-sbom`). Pendente:
  o resto do ritmo (`rotate --all` ✅ no slice 3, `scan` ✅ no slice 2,
  PEM, kill-switch ✅ documentado, WORM).
* **Slice 2 — varredura contínua (parcial: scan do manifesto ✅).**
  `kof cloud scan [--file]` lê o manifesto e acusa `kfs_`/`kfc_`/password
  literal (exit 1 com achado, 0 limpo) — sem login, sem rede, para rodar
  no CI a cada push. Pura `scanManifestSecrets` (nomeia o serviço sem
  vazar o valor). Prova: `tests/deploy_test.kf` (`deploy-scan`) + trava
  da ajuda. Pendente: dependências e `kfs_` sem TTL em serviço crítico
  (exigem rede/conta).

  **Complemento ✅:** o `scan` agora também olha o **ambiente** (lista curada
  de nomes sensíveis — o Kof não enumera env, `config.env(nome)` lê um por
  vez — mais qualquer valor com `kfs_`/`kfc_`/`password`) e o **PEM do
  operador** (`--tls-cert`: expirado ou expirando em 30 dias, via
  `openssl x509 -checkend`). Nunca imprime valor, só o nome. Prova:
  `tests/scan_extra_test.kf` (4). **Pendente:** dependências (exige resolver
  `kofdeps`/rede) e `kfs_` sem TTL em serviço crítico (exige conta).

  **Higiene de credenciais ✅:** `kof cloud service tokens` (e
  `POST /v1/services/token-hygiene`) lista os serviços da conta com `kfs_`
  **válidos e sem expiração** — o item "kfs_ sem TTL em serviço crítico" do
  plano. Só leitura: a ação recomendada é `service rotate <nome> --ttl 24`.
  Escopo por conta; sem token → 401. Prova: `tests/token_hygiene_test.kf`
  (4).
* **Slice 3 — resposta e rotação programada (✅ + `rotate --all`).**
  Rotação assistida (por serviço): `kof cloud service rotate
  (<nome>|--all) [--ttl H] [--prune]` emite credencial nova e **só revoga
  as anteriores depois do TTL** (overlap sem downtime); `--all` itera os
  serviços da conta com falha isolada por serviço (resumo ok/falhas, exit
  1 se alguma falhar); `service rotation <nome>` mostra quantas ainda
  valem. Credencial **sem** expiração nunca é revogada em silêncio — fica
  contada como pendente até ser revogada à mão. Prova:
  `tests/rotate_test.kf` (5: `rotate-all` com dois serviços independentes).
  **Credencial da CONTA (`kfc_`) com ciclo de vida ✅.** O `kfc_` é o token
  mais poderoso do sistema (publica, revoga device, congela conta) e era o
  único sem ciclo de vida: `logout` apagava só o arquivo local e o valor
  seguia válido no servidor para sempre. Agora `TokenEntry.expiresAt`
  (nulável — estado antigo decodifica como "sem expiração", JSN004 medido),
  expiração preguiçosa na validação, `GET /v1/tokens` (inventário sem
  valores), `POST /v1/tokens/revoke` (valor ou id curto; 409 `KCF063`
  ambíguo, 404 `KCF064` opaco) e `POST /v1/tokens/rotate` com overlap
  (antigas só caem depois do TTL; sem TTL nada é revogado em silêncio).
  CLI: `kof cloud token list|revoke|rotate|rotation`; o `rotate` grava a
  nova credencial na sessão local antes de imprimir. Prova:
  `tests/account_token_test.kf` (5: TTL/poda/legado, revoke por valor e id
  curto com escopo por conta, ambiguidade, overlap, HTTP fim a fim).
  **TTL longo (30 dias) não estourava por acaso:** `ttlHours * 3600000` era
  aritmética de `Int` (32 bits), que estoura acima de 596 h — `--ttl 720`
  nascia vencido (401 imediato). O cálculo agora é `Long` e fonte única
  (`tokenExpiryAt`, usada por kfc_ e kfs_); medido com o CLI 0.6.3
  (`720 * 3600000` em Int = `-1702967296`).
  **Acúmulo de logins ✅.** Cada `login` emite credencial nova e não revoga a
  anterior — com o TTL opcional, o inventário cresce a cada entrada e uma
  credencial vazada de um login antigo vale para sempre. `POST
  /v1/tokens/prune` + `kof cloud token prune` mantêm apenas a da sessão atual
  e revogam o resto (escopo por conta; sem token → 401; idempotente). Prova:
  `tests/account_token_test.kf` (7). Pendente: PEMs com 30 dias de antecedência; kill-switch ✅ documentado
  (`docs/SECURITY.md` ## Contenção — procedimento manual reversível com
  o que existe; o `freeze` de um comando da Fase 17 automatiza quando
  existir).
* **Slice 4 — auditoria externa.** Trilha imutável (audit log WORM:
  append-only com hash encadeado, verificável por terceiro); relatório
  SOC2-tipo-II como meta (controles mapeados 1:1 com o que já existe:
  R6, 404 opaco, E2EE, TTL, auditoria).


  **Implementado (cadeia de integridade):** cada `AuditEvent` carrega
  `prevHash`/`hash` (`sha256(prevHash|at|actor|action|target)`), encadeados
  no `Registry.audit`. `GET /v1/audit/verify` e `kof cloud audit --verify`
  reportam `{ok, brokenAt, events}`: alterar um campo ou remover um evento do
  meio quebra a cadeia no índice certo. A listagem `/v1/audit` expõe os
  hashes, então um terceiro pode verificar por conta própria. **Honestidade:**
  não é WORM de verdade — quem tem o arquivo ainda pode reescrevê-lo inteiro
  (inclusive recalculando a cadeia); o que existe é **detecção** de
  adulteração parcial, não imutabilidade. Prova: `tests/audit_chain_test.kf`
  (5). Pendente: WORM real (append-only no filesystem/objeto) e o relatório
  SOC2-tipo-II.
**Regras:** segurança por default em todo slice novo (a pergunta "o que
pode vazar aqui?" entra no template de teste); nenhum dado de cliente
em log/métrica/audit — sem exceção, sem "só dessa vez".

## Ordem de execução no MVP

```
Fase 1 → Fase 2 → Fase 3
```

validando cada etapa. Nada de Fase 4+ antes de a Fase 3 estar estável.
