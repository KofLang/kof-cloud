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
string, address/exposure/protocol/build opcionais); `kof cloud deploy` roda o
`build` de cada serviço (quando declarado; falhou = não publica aquele) e
publica em lote
(idempotente, sem remover o que nao declarou) e `kof cloud down` remove so
os do manifesto (`--prune` poda órfãos do device). Sem artefato transferível
— o build roda local, só o publish atravessa a rede. Parser manual com erro posicional
(Map<String, String> nao aceita numero — ClassCast medido; exposure/protocol/
address validados, sem default silencioso). Prova:
`tests/deploy_test.kf` (8 testes). `deploy --check` verifica publicado +
respondendo (sem rollback); `deploy --plan` mostra o diff sem aplicar.

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

Falta fatiado (Fase 7): build/artefato (passo 1 ✅: campo `build` opcional por
serviço no manifesto, executado antes do publish, falha não publica;
falta: artefato transferível, cache, build remoto), auto-update. `kof cloud doctor` (somente-leitura: login, control plane,
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

## Fase 12 — gestão da nuvem (web client / painel)

**Motivação:** hoje tudo é CLI + JSON. Operar conta/dispositivos/serviços/
domínios/tokens pelo terminal afasta quem não é técnico — e gestão visual
é o que transforma "túnel" em "nuvem" na cabeça do usuário.

**Estado: ⏳ não iniciado.** Desenho fatiado (cada slice é um incremento
testável; o painel consome só a API pública `/v1` — nenhum endpoint novo
privilegiado, mesma auth `X-KofCloud-Token`):

* **Slice 1 — painel somente-leitura.** `serve` ganha `--panel` que expõe
  página estática (status da conta, devices online via `/v1/presence`,
  serviços + hostnames, domínios VERIFIED/PENDING). Sem escritas, sem
  login novo (cola o `kfc_` existente). Prova: teste serve a página e os
  dados batem com a API.
* **Slice 2 — gestão completa.** Criar conta, registrar device, publish/
  unpublish, add/verify/remove domínio, emitir/revogar/listar `kfs_`,
  suspend/unsuspend — tudo pelo painel, chamando os mesmos endpoints da
  CLI. Cada ação pede confirmação quando destrutiva (revoke/remove).
  Prova: e2e via HTTP do painel (publica e remove de verdade).
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

**Estado: ⏳ não iniciado (base pronta: daemon user-level + doctor).**
Desenho fatiado:

* **Slice 1 — instalador por SO.** Linux: usa a unit systemd atual
  (empacotar + `enable` com consentimento); Windows: serviço via
  `sc.exe`/NSSM chamando o mesmo `connect`; macOS: `launchd` user agent.
  Cada um atrás de `kof cloud daemon install` (detecção de SO, não flag).
  Prova: teste gera o artefato certo por SO (puro, sem instalar).
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

**Estado: ⏳ não iniciado.** Desenho fatiado (cada slice com benchmark
antes/depois, mesmo protocolo, sem mudar formato público):

* **Slice 1 — medir de verdade.** Benchmark contínuo no Edge: latência
  p50/p99 por hostname (agregado, sem dado de cliente), tempo de relay
  por tamanho, custo do chunk. Sem medir, otimizar é chute. Prova: nova
  seção no `/v1/metrics` + teste de formato.
* **Slice 2 — relay regional.** `connect` passa a receber a lista de Edges
  (`edgeHost` vira ranking por latência medida no handshake); o Agent
  escolhe o mais próximo e reavalia a cada N minutos. Sem consenso
  distribuído no MVP (cada Edge continua independente; o control plane
  só publica a lista). Prova: e2e com 2 Edges, Agent migra para o mais
  rápido.
* **Slice 3 — caminho de bytes O(n).** Tirar `bytesToString`/`base64` via
  String do caminho quente: chunks maiores (64 KiB) + cópia direta de
  bytes quando o stdlib permitir; handshake comprime `services[]`
  (hoje vai cheio a cada `connect`). Prova: benchmark mostra queda
  linear no tempo por MiB.

**Regra:** nenhuma otimização sem número antes/depois no `STATUS.md`;
latência de cauda (p99) vale mais que média.

## Fase 15 — E2EE real (nem a plataforma lê)

**Motivação:** hoje o túnel cifra Agent↔Edge (`sessionKey` por sessão),
mas o Edge **decifra para rotear** — ele vê o Host e o corpo em claro.
E2EE real significa: a plataforma nunca tem a chave do conteúdo, mesmo
comprometida. É o que protege a máquina física e o site hospedado de
verdade (inclusive da própria operadora da nuvem).

**Estado: ⏳ não iniciado.** Modelo honesto, em camadas (cada slice reduz
o que a plataforma vê, sem quebrar o anterior):

* **Slice 1 — declarar o modelo.** Documentar o que cada parte vê hoje
  (cliente vê tudo; Edge vê Host+corpo; Agent/serviço veem tudo) vs o
  alvo (Edge vê só o necessário para rotear). Sem código — alinhamento.
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

**Estado: ⏳ não iniciado.** Desenho fatiado:

* **Slice 1 — separar estado do processo.** Registry atrás de interface
  (hoje a classe já concentra tudo): backend arquivo (atual) → backend
  SQLite/Postgres por config. Migração com `kof cloud migrate --dry-run`
  (lê, valida, escreve, compara contagem) antes de trocar. Prova: suíte
  roda igual nos dois backends.
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

**Estado: ⏳ não iniciado.** Desenho fatiado (cada slice exercitado em
game-day: simula o incidente de propósito e mede o tempo de resposta):

* **Slice 1 — detecção e severidade.** Sinais com nível
  (`INFO/WARN/CRIT`): pico de 401/429, \u200btúnel caindo em massa, revoke em lote,
  expiração em massa de `kfs_`, Edge sem heartbeat. `GET /v1/incidents`
  lista os abertos (filtrado por conta quando for dado de cliente, global
  quando for plataforma). Prova: teste injeta o sinal, incidente abre.
* **Slice 2 — contenção em um comando.** `kof cloud freeze <alvo>`
  (conta|device|service|edge): congela tudo do alvo (suspend + revoga
  tokens vivos + derruba sessões) e gera ID de incidente; `unfreeze`
  reverte passo a passo com confirmação. Congelar é reversível por
  desenho — destruir nunca é automático. Prova: e2e congela e
  descongela sem perder configuração.
* **Slice 3 — comunicação e post-mortem.** Status page mínima (lê os
  incidentes globais; sem login), notificação por webhook configurável
  (conta cadastra URL + segredo; payload assinado Ed25519), e template
  de post-mortem no repo (`docs/incidents/AAAA-MM-DD-titulo.md`: causa,
  impacto, timeline, correção, ação preventiva com dono e prazo).
  Prova: game-day completo fecha com post-mortem publicado.

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

**Estado: ⏳ não iniciado (ritmo, não projeto).** Desenho:

* **Slice 1 — baseline e SBOM.** `kof cloud audit --sbom` gera inventário
  (versão do CLI, Runtime Kof, deps, hashes); `docs/SECURITY.md`
  (política: o que é suportado, como reportar, SLA de resposta);
  `security.txt` no Edge (`/.well-known/security.txt`).
* **Slice 2 — varredura contínua.** `kof cloud scan` (dependências,
  segredos vazados em `kofcloud.json`/env, PEM expirando, `kfs_` sem
  TTL em serviço crítico); roda no CI a cada push + game-day mensal.
  Achado vira issue com severidade, nunca silêncio.
* **Slice 3 — resposta e rotação programada.** Rotação assistida
  (`kof cloud rotate --all`: re-emite `kfs_` com overlap, revoga os
  antigos depois do TTL; PEMs com 30 dias de antecedência); kill-switch
  documentado (o `freeze` da Fase 17 em escala de plataforma).
* **Slice 4 — auditoria externa.** Trilha imutável (audit log WORM:
  append-only com hash encadeado, verificável por terceiro); relatório
  SOC2-tipo-II como meta (controles mapeados 1:1 com o que já existe:
  R6, 404 opaco, E2EE, TTL, auditoria).

**Regras:** segurança por default em todo slice novo (a pergunta "o que
pode vazar aqui?" entra no template de teste); nenhum dado de cliente
em log/métrica/audit — sem exceção, sem "só dessa vez".

## Ordem de execução no MVP

```
Fase 1 → Fase 2 → Fase 3
```

validando cada etapa. Nada de Fase 4+ antes de a Fase 3 estar estável.
