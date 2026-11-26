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
reconexão com backoff (passo `kofcloud.limits.backoffBaseMs/MaxMs`, default
500/30000 ms, recuo seguro nas puras `backoffBaseLimitFrom`/
`backoffMaxLimitFrom` — base não-positiva recai no default, teto abaixo da
base recai na base).

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
(dispositivos ativos por conta, default 5, **zero/inválido recaem no
default via `deviceLimitFrom` pura — com o valor cru `count >= 0`
congelava TODO cadastro novo em TODA conta**), `kofcloud.limits.services` (por dispositivo, default 10, **mesmo recuo via `serviceLimitFrom` pura**) e
`kofcloud.limits.streams` (por túnel, default 128, **zero/inválido recaem no
default via `streamLimitFrom` pura — 0 recusaria todo OPEN**, com
`429`/`RESET LIMIT_EXCEEDED`).
Prova: `tests/limits_test.kf` (11 testes: limites + recuo seguro do passo
da varredura de credenciais e do pusher de SERVICES) + `tests/e2e_multiservice_test.kf`
(3 serviços no mesmo túnel, com isolamento de falha).

**Presença visível no `device list` (parte do contrato de `03-device.md`).**
O doc dizia que "sem heartbeat por 3× o control plane marca o dispositivo
`OFFLINE`" — esse estado não existe: `status` é o ciclo de vida
(`ACTIVE`/`REVOKED`/`SUSPENDED`) e presença é um eixo efêmero decidido pelo
Edge. O `device list` mostrava só o cadastro, então responder "este notebook
está no ar?" exigia cruzar dois comandos. Agora a linha traz
`tunel: ONLINE|OFFLINE|desconhecido (motivo)` via `/v1/presence`, com a mesma
regra do `status`/`doctor`; falha na consulta nunca vira `OFFLINE` inventado.
Prova: `tests/device_presence_test.kf` (5).

**Teto de conexões públicas simultâneas (parte do contrato de `10-edge.md`).**
A tabela prometia "1024 conexões públicas simultâneas → `503` + fecha", mas o
`serveHttp` spawnava uma task por `accept` sem teto: uma rajada de conexões
abertas e ociosas (slowloris) consumia uma task e um socket por conexão até
exaurir a máquina. Agora `EdgeState.tryAcquirePublicConn` reserva a vaga no
`accept` (antes de ler o pedido) e `handlePublicConnection` libera em
`finally` — sem isso, uma exceção no relay vazaria a vaga e o Edge acabaria
recusando todo mundo. Excedeu → `503 KCF117` + métrica
`kofcloud_edge_http_connections_refused_total`. O teto tem recuo seguro via
`publicConnLimitFrom` pura (zero/inválido recaem no default 1024 — com 0 a
primeira conexão já seria recusada). Prova:
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
de verdade. **2xx = `ONLINE`**, qualquer outro código (3xx/4xx/5xx ou
inválido) ou sem resposta no prazo = `DEGRADED`, recusa = `OFFLINE` — a
regra antiga "status < 500 = ONLINE" aprovava 404/403/301: um serviço com a
rota de health removida ficava ONLINE e o Edge roteava para ele (a decisão
é a pura `healthStatusFor`, coberta pelo teste). O prazo
(`kofcloud.limits.healthTimeoutMs`, 2 s) usa o único mecanismo do Kof para
`receive` sem timeout: uma task fecha a conexão e o receive em voo lança
(medido). A regra também fica travada na prosa: os cabeçalhos de
`Health.kf`/`Deploy.kf` e deste teste dizem 2xx, e a trava quebra se alguém
restaurar "< 500". Prova: `tests/health_test.kf` (5).

## Fase 5 — autenticação

`PUBLIC` / `AUTHENTICATED` / `PRIVATE` no modelo e no Edge. `PRIVATE` já existe
desde a Fase 2 (sem rota); `AUTHENTICATED` adiciona a checagem de token antes do
túnel.

**Estado: ✅ implementado.** Credencial de serviço `kfs_` emitida pelo control
plane (`POST /v1/services/token`) e validada no Edge; o token da conta não
autoriza acesso público; o Edge remove a credencial antes de repassar ao
serviço local; `401` com `WWW-Authenticate: Bearer`. CLI: `kof cloud service
token <nome>` (revogacao por token ou id curto: `kof cloud service token-revoke <nome> <kfs_|id-curto>` → `POST /v1/services/token/revoke`, 404 opaco se alheio, 409 `KCF063` em ambiguidade; inventario: `kof cloud service token-list <nome>` → `POST /v1/services/tokens`, so id curto + emissao, nunca o valor; expiracao opcional `--ttl <horas>`, expirado vira 401 e some do inventario; o `service status` resume o inventario (`tokenInventoryLine`) contando as perto do fim com o MESMO predicado do token-list; o `deploy --check`, o `deploy --plan` e o `doctor` AVISAM as kfs_ perto do fim dos servicos AUTHENTICATED do manifesto (`check: aviso:` / `plan: aviso:` / `AVISO` — mesmo predicado, nunca problema/FALHA). Prova: `tests/service_token_test.kf` (15 testes) +
`tests/e2e_authenticated_test.kf`.

**Endurecimento (padrão kof-share R6): ✅ implementado.** Erros estáveis
`KCF001`–`KCF106` (`model/Errors.kf`): nunca removidos nem renumerados; 404
opaco entre contas (existente-alheio indistinguível de inexistente);
owner-gate em `POST /v1/services/status` (só a conta dona altera); CLI com
`Hint:` humana (`printFail`); validação de `publish` devolve o HTTP certo por
família (400 validação, 404 dono/recurso, 429 limite). Prova:
`tests/errors_test.kf` (11 testes: códigos do control plane, 404 opaco,
limites/owner-gate no registry, códigos do Edge, Hint + trava de regressao
100% + revoke/TTL + nome|id + fail-fast de flags).

## Fase 6 — domínio próprio

**Estado (slice 1): ✅ implementado.** Registro PENDING com instrução CNAME
(`POST /v1/domains` → `CNAME api.example.com -> site.mel.kofcloud.io`),
verificação por `net.resolve` contra os destinos esperados do Edge (config
`kofcloud.edgeHosts`, comparação exata, NXDOMAIN/falha vira `KCF124`
indistinguível de "resolveu para outro lugar"), rota ativa só depois de
VERIFIED, poda junto com serviço/dispositivo. A zona reservada cobre
subdomínios **e o apex** (`kofcloud.io` também é recusado com `KCF120`). Dono verificado antes do DNS
(sem trabalho de rede para conta alheia). CLI: `domain add --service`,
`domain verify [--tls-cert <pem>]`, `domain remove`; erros `KCF120`–`KCF124` e `KCF164`
com `Hint:`. O `--tls-cert` julga o PEM localmente antes da rede (cobre o
hostname? expira em 30 dias?) — ver Fase 9 slice 3.
Prova: `tests/domains_test.kf` (18 testes: validação, registro/idempotência,
verificação sem rede, rota/poda, API com service preenchido, Edge via túnel real, Hint + serializadores, certificado × hostname puro e com openssl real, webhook DNS no fio e dry-run).

**Fatia: DNS fora do Edge recusa (KCF164).** A verificação aprovava quando
ALGUM endereço era o Edge: o hostname com CNAME para o Edge que também
responde num IPv4 de terceiro (registro antigo, CDN/provedor no meio)
virava VERIFIED e parte dos clientes ia para fora da plataforma — medido
por sonda (`outcome='' status=VERIFIED` com `198.51.100.7` na resposta).
Agora, achado o Edge, `domainResolvesOnlyToEdge` confere os demais IPv4
literais; sobra de terceiro = PENDING + `409 KCF164` (distinto do `KCF124`
de "nada chega"). Hostname CNAME-only (sem endereço próprio) e texto de
CNAME devolvido pelo resolver continuam aprovando. Prova:
`tests/domains_test.kf` (fatia `domains-verify`: dois testes novos + expectativa antiga
ajustada).

**Fatia: AAAA de terceiro recusa (KCF164).** A mesma sonda mostrou que um
registro AAAA de terceiro na resposta era ignorado — o cliente que prefere
IPv6 caía fora da plataforma mesmo com o IPv4 certo. O IPv6 agora é
julgado pelo valor: `ipv6Groups` expande `::` e a forma mista
(`1:2:3:4:5:6:1.2.3.4`) em 8 grupos numéricos, então `2001:db8::10` e
`2001:db8:0:0:0:0:0:10` são o mesmo host; o IPv4 mapeado
(`::ffff:203.0.113.10`) casa com o IPv4 declarado em `kofcloud.edgeHosts`,
e o Registry aceita o AAAA do Edge pela mesma comparação. Saída: PENDING +
`409 KCF164`. Prova: `tests/domains_test.kf` (fatia `domains-verify`: dois
testes novos; três mutações — AAAA ignorado, IPv6 sempre recusando, Registry
sem IPv6 — quebram).

**Domínios próprios no `status` ✅.** O `status` listava os serviços sem
dizer quais hostnames externos levam a eles — o operador que mexeu no DNS
de `api.example.com` não sabia qual serviço, nem onde ele roda. Agora, com
serviços publicados, o status busca `/v1/domains` (best-effort) e imprime
uma seção "Domínios: N apontando para esta maquina" com `<servico> <-
custom <host> <PENDING|VERIFIED>`: `domainsPointingToServices` (pura —
casa pelo nome do serviço alvo, custom sem serviço e de outra máquina
ficam fora) e `statusDomainLine` (pura). Prova: `tests/registry_test.kf`
(`status-domains` + `status-domains-e2e` com control real).

Automação DNS específica de provedor ✅ (Cloudflare/Route53/qualquer CLI):
`domain add --dns-webhook` (receptor HTTP do operador) e `--dns-cmd`
(CLI do provedor via `sh -c`, sem SDK), ver Fase 18 slice 1.

Falta (fatiado de propósito): TLS por SNI no Edge (o Kof não expõe — ver
`12-tls.md`). Sem isso, o hostname próprio serve HTTP pela borda; o HTTPS
continua dependendo de terminador externo.

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
address validados, sem default silencioso). **Fatia: nome único no manifesto.**
`parseManifest` recusava tudo menos a repetição: dois itens com o mesmo
`name` passavam e o publish — idempotente por nome — reescrevia o primeiro
em silêncio. Agora nome repetido falha posicional antes de build/publish.
**Fatia: campo repetido na mesma entrada.** `manifestField` lê pela primeira
ocorrência, então `"exposure":"PRIVATE","exposure":"PUBLIC"` valia `PRIVATE`
e a segunda cópia era ignorada sem aviso (medido) — mesmo risco do nome
repetido. `duplicateManifestField` recusa posicional antes do build/publish.
Prova: `tests/deploy_test.kf` (`deploy-err` estendido). **Fatia: nome é único
na CONTA, não só no manifesto.** Medido: publicar `site` no device B devolvia
`200` com o `site` do device A (porta do A) e nada era criado — a conta ficava
com um serviço só e, como o hostname é `<nome>.<handle>`, o roteamento ia
sempre para o A. A idempotência do publish passou a ser por (conta, device,
nome): mesmo device devolve o existente; outro device dá `409 KCF139`. Prova:
`tests/deploy_test.kf` (`deploy-e2e`) e `tests/errors_test.kf`
(`errors-hint-all`, 59 construtores). **Fatia: nome ambíguo na CLI é recusado.** O
resolvedor devolvia o primeiro serviço que casava, então `service status site`
com "site" em dois devices agia no dispositivo que a ordem da lista decidisse.
Agora `resolveServiceName` (pura) distingue inexistente de ambíguo e os
comandos por nome recusam listando os devices. Prova:
`tests/hosts_test.kf` (`service-name-match` estendido). **Fatia: `--check`/`doctor` conferem o
dispositivo.** O manifesto descreve a máquina local, mas `verifyDeployment`
casava só o nome: um serviço homônimo publicado noutro device contava como
"publicado" e o check dava verde para um deploy que nunca chegou ali —
medido. `verifyDeploymentFor` recebe o `deviceId` esperado e acusa
"publicado no dispositivo X, não neste"; `cmdDeploy` e `doctor` passam o
device da sessão; a sobrecarga antiga (sem escopo) fica para chamadores sem
sessão. Prova: `tests/deploy_test.kf` (`deploy-check` estendido: sem escopo
aprova, com escopo acusa). **Fatia: `--plan` também tem escopo.** Medido:
serviço do manifesto publicado noutro device aparecia como "= igual" (o
deploy falharia com 409) e serviço de outro device aparecia como "fora do
manifesto; use down --prune" (o prune nunca o alcança). Agora
`diffDeploymentFor` marca `! conflito` para o nome tomado noutro device e
omite extras de outro device; `deploy --plan` passa o device da sessão.
Prova: `tests/deploy_test.kf` (`deploy-plan` estendido). **Fatia: `down` diz quando o serviço vive noutro device.** Antes, um serviço homônimo publicado noutro device imprimia "ausente (ok)" — o operador achava que não havia nada e ele seguia no ar com o mesmo hostname. `serviceIdOnDevice` e `serviceDeviceElsewhere` (puras) separam os dois casos; a mensagem `downElsewhereLine` nomeia serviço, device e a orientação de rodar o down lá. Prova: `tests/deploy_test.kf` (`deploy-down-scope`). **Fatia: sumário honesto do `down`.** O resumo final contava
`services.size` (o manifesto) em vez do que foi removido: um manifesto de 3
serviços nunca publicados dizia "3 removidos" — medido. `downSummary` (pura)
diz "nada a remover (N no manifesto, M noutro dispositivo)" quando nada saiu,
mantém a contagem real quando removeu e "down com N falha(s)" quando falhou.
Prova: `tests/deploy_test.kf` (`deploy-down-summary`). **Fatia: `down` exige `--yes`
(guarda de destruição).** O `down` REMOVE serviços e a remoção é
irreversível — sem confirmação explícita ele agora aborta. O caso medido: um
`down` que removeu parte e abortou por uma falha, mais um serviço publicado
de novo entre os dois comandos, fazia o `down` seguinte apagar um serviço
que o operador não pediu naquele momento (re-criado por entropia). A
contagem pura `downOrphanCount` (só o device da sessão, o mesmo predicado do
`--prune`) e `downConfirmationRequired` alimentam a mensagem: nomeia os
declarados, diz quantos órfãos o `--prune` levaria e aponta o `down --yes`.
Prova: `tests/deploy_test.kf` (`deploy-down-guard`: pura + `--yes` repetido
barra no uso) + `deploy_test.kf` (`valueflags-unit`: `down --prune --yes`
passa, `--yes --yes` barra).
**Fatia: `device revoke` exige `--yes`.** O `down` ganhou guarda de
destruição (acima); o `device revoke` ficou de fora e é o comando MAIS
destrutivo da CLI: revogar é irreversível e leva junto os serviços
publicados e as credenciais vivas do dispositivo. Sem confirmação, um typo
ou uma linha colada de script derrubava a máquina inteira. Agora resolvido
o alvo, sem `--yes` o comando só avisa (`deviceRevokeConfirmationRequired`,
pura — nomeia o dispositivo e aponta o comando de confirmação) e não toca
o control plane; `--yes` repetido barra no uso (mesma regra do `down`).
Prova: `tests/value_flags_test.kf` (`device-revoke-guard`: pura + `--yes`
repetido) e `tests/webhook_test.kf` (sem `--yes` o device continua vivo;
com `--yes` revoga).

**Fatia: nome de serviço com caixa errada ganha diagnóstico.** O nome de
serviço é chave EXATA (manifesto, rota, hostname) — "Site" não resolve
"site", e adivinhar mudaria a chave em silêncio. Mas os comandos por nome
diziam só "não encontrado", e o operador conferia o `service list` e
achava o nome certo. Agora `serviceCaseHint` (pura, comparação própria sem
`toLowerCase` — a caixa é lacuna medida do runtime, L20) nomeia o serviço
existente; `printServiceNotFound` é o ponto único das mensagens (o aviso
nunca aparece sem o "não encontrado"). Wire em unpublish, status, token,
token-list, token-revoke, rotate, rotation e todo comando que passa pelo
`resolveServiceOrReport` (suspend, unsuspend, tokens). Prova:
`tests/registry_test.kf` (`service-case-hint`) + `tests/scan_extra_test.kf`
(`service-case-e2e`: `unpublish Site` recusa e nada é removido).
**Mesmo diagnóstico para o dispositivo ✅:** `device revoke Notebook` /
`suspend Notebook` diziam só "dispositivo nao encontrado" — agora
`deviceCaseHint`/`printDeviceNotFound` (puras, irmãs do serviço) nomeiam
o device existente; `resolveDeviceByNameOrId` devolve a lista já buscada
para o aviso não custar uma segunda viagem ao control plane. Wire em
revoke/suspend/unsuspend/freeze/publish --device. Prova:
`tests/registry_test.kf` (`device-case-hint`).
**Fatia: artefato tem de ser arquivo
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
"echo hi", `exit 3`, pipe, comando inexistente). Agora o detalhe da falha inclui a última linha de
stderr (ou stdout): `build falhou (exit N) — <linha>`; só o código obrigava o
operador a reproduzir o comando à mão. Uma linha só, nunca o comando — o
detalhe vai ao terminal do usuário. `deploy --check` verifica publicado +
respondendo + artefato (aviso `artefato ausente`, nunca bloqueio; sem
rollback) e AVISA credenciais `kfs_` perto do fim nos serviços
AUTHENTICATED do manifesto (`kfsExpiryCheckLine`, mesmo predicado do
`token-list`; `check: aviso:` nunca muda o exit — rotação é rotina,
não outage); `deploy --plan` mostra o diff sem aplicar (com `(build)`/
`(artifact)` inclusive no igual).

**Estado (slice 3 — daemon user-level): ✅ implementado.** `kof cloud daemon
install|status|uninstall`: gera `kofcloud-agent.service` (mesmo `connect`,
`Restart=always`, `NoNewPrivileges`, enable manual de proposito), erro
honesto sem systemd; artefato por SO na Fase 13 slice 1 (systemd continua o
padrao no Linux). Prova:
`tests/daemon_test.kf` (14 testes: 9 do daemon + 5 da Fase 13 slice 1 — ver U28).

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

**Listagens em JSON para CI (`--json`).** `kof cloud service list --json` e
`device list --json` imprimem a MESMA serialização do fio (`serviceJson`)
e, no device, o eixo de presença (`tunnel`: ONLINE/OFFLINE/"desconhecido
(motivo)") derivado do `/v1/presence` — nenhum script precisa parsear a
tabela de texto nem chamar `curl`. A presença vive numa pura única
(`presenceOf`) usada pelo texto E pelo JSON (as duas não divergem); lista
vazia é `[]`, nunca o texto de "nenhum serviço" (que quebraria o parser).
Prova: `tests/json_output_test.kf` (2: puras + e2e).

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
**Camada da auditoria (Fase 19 slice 4):** `doctorAuditLine` pura sobre o
corpo do `/v1/audit/verify` — cadeia quebrada ou espelho comprometido
viram FALHA com orientação (`kof cloud audit --verify`); control plane
antigo sem os campos do espelho vira ok honesto "espelho nao reportado",
nunca FALHA nem mentira. Prova: `tests/doctor_test.kf` (`doctor-audit`,
2 testes: puro em 6 casos + e2e que trunca o espelho no disco).

## Débitos de limites (antes do deploy)

**Estado (frame DATA fatiado + teto de linha): ✅ implementado.** Envio em
blocos de 16 KiB (`kofcloud.limits.sendChunkBytes`) já codificados, um frame
DATA por bloco; recepção recusa linha acima de `maxTunnelLineBytes()`
(2×frameBytes + margem, fail-closed com `LINE_OVERFLOW` no Edge). Chunk
adaptativo (encolhe até caber — o base64 do stdlib passa por UTF-8 e expande
bytes altos). O teto tem recuo seguro via `frameLimitFrom` pura
(zero/inválido recaem no default 1 MiB — com 0 o remetente seguiria
mandando blocos de `sendChunkBytes` enquanto o receptor recusaria linha
acima de 4096, quebrando o relay no meio em silêncio); o BLOCO DE ENVIO
tem a sua (`sendChunkBytesFrom` pura — mesmo recuo do receive, 0/negativo
recaem no default 16 KiB porque sem bloco nenhum o relay pararia em
silêncio, + clamp ao teto do frame). Prova:
`tests/frames_test.kf` (4 testes: fatiamento +
recomposição byte-idêntica, codec fail-closed, HELLO gigante fecha sem
WELCOME, 1.5 MiB fim a fim — mais os asserts do recuo do bloco de leitura
na mesma suite).

**Estado (idle timeout): ✅ implementado.** `lastActivityMs` por sessão
(renovado só por frame útil — PING/PONG não contam) + varredura a cada 5 s
nos dois lados: túnel ocioso cai (`kofcloud.limits.tunnelIdleMs`, default
120 s; métrica `_idle_timeout` no Edge) e stream preso leva `RESET
IDLE_TIMEOUT` (`kofcloud.limits.streamIdleMs`, default 30 s) sem derrubar a
sessão. EOF limpo virou throw (reconecta com backoff), exceto com stopFile
ou GOAWAY — antes, queda silenciosa parava o Agent. Prova:
`tests/idle_test.kf` (5 testes).

**Estado (conexões simultâneas por serviço — fecha os débitos): ✅
implementado.** Contador por `serviceId` no Edge
(`kofcloud.limits.serviceConns`, default 64, **zero/inválido recaem no
default via `serviceConnLimitFrom` pura — 0 recusaria todo o relay com 429**);
excedeu → `429 KCF108` +
`Retry-After` antes do OPEN (sem consumir stream nem tocar no Agent).
Aproximado sob corrida (documentado); métrica `_service_overloaded`.
Prova: `tests/conns_test.kf` (2 testes: concorrência real com servidor
lento, liberação, isolamento, métrica, Hint; recuo do teto com mutação).

## Abuse prevention (parcial, antes do deploy)

**Estado (slice 1 — rate limit por hostname): ✅ implementado.**
`security.rateLimit` (fixed-window, medido) fiado no Edge público:
`kofcloud.limits.httpRequests` por hostname (default 100, **zero/inválido
recaem no default via `httpRequestLimitFrom` pura — cota 0 recusaria o
primeiro pedido de todo hostname, a borda inteira em 429**) e janela
(`kofcloud.limits.httpWindowMs`, default 60 s); excedeu → `429 KCF107`
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
audit`. Prova: `tests/abuse_test.kf` (5 testes: rota/health/API, device no
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
Prova: `tests/metrics_test.kf` (13 testes: formato, recusa por motivo,
Edge fim a fim, privacidade, endpoint + contadores via HTTP e via registry).

**Implementado (leitura pela CLI — `kof cloud metrics`):** o export existia
no fio e nenhum comando o consumia — o operador precisava de `curl`. Agora
`kof cloud metrics [--grep <texto>] [--control <url>]` imprime as linhas do
export (filtro puro `metricsLines`: substring na linha, vazio = tudo, linhas
vazias fora; sem casamento = mensagem e exit 0, nunca erro). Não exige sessão
(métricas são agregados sem dado de conta, como no `/v1/health`): usa
`--control`, senão o control da sessão, senão o default; control fora do ar
é exit 1. Prova: `tests/metrics_cmd_test.kf` (2: filtro puro + help, e2e
com contador real — sem filtro, com filtro, filtro sem casamento e control
fora do ar).

**Estado (slice 3 — fecha): ✅ implementado.** `agent/Metrics.kf` conta
connects, reconnects (backoff), recusas fatais por motivo, health reports,
streams servidos e erros locais — agregados, sem nome de serviço, endereço,
porta, conteúdo, token ou chave. O Agent roda em processo separado, então
seus contadores vivem no próprio processo (sem protocolo novo no MVP).
Prova: `tests/agent_metrics_test.kf` (3 testes: contadores puros, fluxo
real via túnel, recusa fatal por motivo).

**Estado (eventos legíveis do Edge — mesmos fatos, formato humano): ✅
implementado.** O Edge contava recusas e quedas em métrica/anel, mas não
tinha PALAVRA nenhuma: o `serve` imprime só o banner e o operador da
plataforma não via POR QUE um túnel foi recusado. Agora `edge/Events.kf`
emite uma linha (stdout via `log.*` **e** `~/.kofcloud/edge-runtime.log`):
túnel recusado (com o MOTIVO do protocolo — DEVICE_UNKNOWN, AUTH_FAILED,
DEVICE_REVOKED/SUSPENDED, VERSION_UNSUPPORTED, LINE_OVERFLOW), túnel
aceito (com contagem de sessões vivas), sessão derrubada por ociosidade
(com o teto em ms), teto de conexões públicas atingido (com o limite —
503), parada limpa (GOAWAY SHUTDOWN) e agora a **família upstream do
caminho público**: os 8 desfechos de relay (`upstream_error` + teto de
streams) deixam palavra — resposta acima do teto (com o número), tempo
esgotado (com o prazo; corpo curto quando há parcial), teto de streams,
rota que o agente ainda não conhece, falha no túnel, resposta vazia e
resposta que não é HTTP. Palavra fixa, nunca o conteúdo nem o erro do
relay (o `e` do catch fica fora do arquivo). Mesmo contrato do agent-runtime
(teto `kofcloud.limits.logBytes`, corta o início; falha de IO nunca derruba
o Edge): motivo do protocolo e contagens, **nunca** hostname, device,
credencial ou corpo. Leitura: `kof cloud logs --edge` (mutuamente exclusivo
com `--runtime` — mundos distintos: Agent local × Edge da plataforma).
Prova: `tests/edge_events_test.kf` (6: linha com nível, teto que corta o
início com carga real, contrato sem segredo/hostname, e2e — HELLO inválido
num Edge real deixa "tunel recusado: DEVICE_UNKNOWN" no arquivo, HELLO
gigante acima do teto fecha SEM responder e deixa "tunel recusado:
LINE_OVERFLOW" sem vazar o conteúdo, a parada limpa deixa o GOAWAY — e
caminhos separados dos outros dois logs). Paridade do fail-closed
mid-sessão: sessão ACEITA que mandar linha acima do teto fecha com a
palavra "sessao fechada: linha gigante acima do teto (LINE_OVERFLOW)" +
métrica no balde `_line_overflow` — antes era o único desfecho de sessão
em silêncio; decisão do Edge não alimenta a sonda de tuneis perdidos.

**Estado (eventos legíveis do AGENT — o outro lado conta a sua verdade):
✅ implementado.** O inventário de palavras do Edge expôs o espelho mudo: o
sweeper de ociosidade do AGENT derrubava a sessão sem palavra nenhuma e o
fim do `connectAndServe` jogava "edge closed connection" — acusando o peer
pelo que foi decisão daqui. Agora o sweeper emite `agentIdleDropLine`
("sessao ociosa derrubada localmente (Nms sem frame util)") e marca
`idleDropped` na sessão (somente em memória — nunca no fio nem no estado)
ANTES de fechar; o fim do `connectAndServe` lê a marca e joga "sessao
derrubada localmente por ociosidade", preservando "edge closed connection"
para queda real do peer. Reset de streams presos é contado em lote, uma
palavra por varredura ("N stream(s) preso(s) resetado(s) por ociosidade") —
contagem, nunca conteúdo. Prova: `tests/idle_test.kf` (`idle-reset`
estendido: formato das linhas puras, sem credencial, marca nasce falsa,
travas com ÂNCORA de indentação no wiring do sweeper e no throw honesto —
técnica do c330b8c; 3 mutações — throw, palavra de derrubada, palavra de
reset — ficam vermelhas e foram revertidas).

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

  **Kof 0.8 — HSTS implementado pelo KofCloud ✅.** O runtime deixou de
  emitir o header (§823 do Kof: sem `app.security()`, nada é enviado). O
  control plane o emite por middleware no listener TLS (e só nele):
  `Strict-Transport-Security: max-age=31536000`, valor em fonte única
  (`hstsHeaderValue`) e sem `includeSubDomains` — os subdomínios da conta
  são servidos pelo Edge (outro certificado e escopo), e prometer cobertura
  que não se tem impediria o operador de servir HTTP lá. A trava
  `control-tls` confere o header no fio e a ausência dele em HTTP. Prova:
  `tests/control_test.kf` (`control-tls`).
* **Slice 3 — SNI por hostname + renovação assistida (parcial: expiração visível ✅).**
  O `serve` com `--tls-cert/--tls-key` roda `openssl x509 -checkend 30d`
  e avisa (nunca bloqueia) quando o PEM expira em menos de 30 dias;
  openssl ausente/erro = sem aviso. Pura `tlsCertExpiringSoon` (exitCode
  → Bool). **Vencido × expirando ✅:** `-checkend` sai 1 para os dois; agora
  `tlsCertVerdict` + `certExpiredAt` (puras) usam a data real
  (`openssl -enddate`, formato C e ISO) e o serve diz "JA VENCIDO" quando é.
  Prova: `tests/serve_tls_expired_test.kf` (3) + `tests/hosts_test.kf` (`serve-tls`).
  A MESMA regra chega aos outros três comandos que julgam um PEM —
  `domain verify --tls-cert`, `service status --tls-cert` e `doctor
  --tls-cert` (`domainCertLineWith`, `doctorTlsLineWith`): vencido vira
  `FALHA tls` ali também ("aviso" só para a janela dos 30 dias).
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
  **`service status` aceita o PEM ✅** (`--tls-cert <pem>`): o veredito
  visível fecha a lista dos comandos que olham um serviço de relance —
  openssl roda UMA vez (`serviceCertLines`) e cada hostname impresso no
  status (default da conta + customs via `customDomainsFor`) ganha a MESMA
  linha do `domain verify --tls-cert` (`domainCertLine`/`certHostVerdict`,
  uma regra só): expirando em 30 dias é aviso na linha `ok`, PEM
  ausente/inválido/que não cobre é `FALHA tls` e sai exit 1; flags soltas/
  repetidas barram antes de qualquer rede. Prova:
  `tests/status_cert_test.kf` (`svcdevice-unit` + `status-cert`, com PEM
  real e pula sem openssl; mutação na expiração quebra o aviso e foi
  revertida).
  Pendente: SNI por hostname (mapa em memória, recarregado sem derrubar o
  Edge — depende do runtime) e renovação automática.
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

**Dica (Hint) obrigatória para todo código KCF ✅.** A `Hint:` é o caminho do
operador quando um erro chega à CLI, mas nada garantia que todo código
devolvido pelo control plane/Edge tivesse uma: um construtor novo sem
`errHint` aparecia como erro cru. A trava `hint-coverage` percorre os
construtores de `model/Errors.kf` (fonte única) e exige Hint para **todos**
(≥40 códigos), proíbe qualquer Hint que **ecoе um valor** de `kfs_`/`kfc_`
ou senha — só o prefixo como nome de argumento (`<kfs_...>`) ou a palavra
"kfs_ completo" passam (`hasCredentialValue`, pura e estritamente testada).
Prova: `tests/hosts_test.kf` (`hint-coverage` ×2).

**Migração ao Kof 0.8.1 ✅ (restabilização).** O toolchain do host mudou e
o repositório inteiro parou de compilar/rodar (`PARSE087`, `split` literal,
`JSN006` estrito, TLS verificado por padrão, HSTS não automático — ver L21
em `13-kof-apis.md`). Mudanças: todas as asserções não literais viraram
`need(cond, "literal")`; o fio do control plane manda `localPort`/
`createdAt`/`lastSeenAt` como **número JSON** (objecto montado à mão —
`mapOf()` é homogêneo) e o decoder lê explicitamente; a CLI/Agent aplicam
`applyHttpTlsPolicy` (opt-out só com control em loopback) e os e2e chamam
`trustLocalhostTls()`; o HSTS deixa de ser promessa do runtime (trava do
`control-tls` verifica o fio). `split("\\+")` da trava `u-counts` virou
`split("+")`. Prova: `kof test --target jvm --timeout 120` verde.

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
  novo vale nos três. **Barra invertida ✅:** `..\` cru e a forma JSON-escapada
  `..\\` (que contém a crua como prefixo) casam em path, corpo e header — sem
  normalizar a string (O(n²) em Kof); o pedido repassado segue byte a byte.
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
  (default 1 MiB, **zero/inválido recaem no default via `httpBodyLimitFrom`
  pura — com 0 todo corpo viraria 413**); o Edge recusa antes do relay quando o `Content-Length`
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
  `tests/framing_test.kf` (15 testes; os 3 do teto de corpo — ver U54: unit, e2e 400/413, relay com corpo mentiroso).
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
  (medido; mesma lição do separador `;` em `Registry.expiryProbes`). A prosa
  fora da tabela viajava defasada do mesmo jeito: README e STATUS diziam
  "56 suites" com a suite real em 57. Corrigido e a trava derivou os números
  em prosa AO VIVO — `Total: N testes` = soma real dos arquivos (não o
  declarado na tabela), `# N suites` do STATUS, `(N suites)` do README e o
  run total `N passed, 0 failed` do STATUS = linhas reconhecidas (o resumo
  final da CLI conta suites, não testes; o run total viajava em 56 com
  57 suites reais). Sanidade exige suites > 50.
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

  **Implementado (teto de corpo POR SERVIÇO ✅):** `publish` aceita
  `maxBodyBytes` (API `POST /v1/services`, CLI `--max-body <bytes>` e
  manifesto `"maxBodyBytes"`) — texto decimal no record (`""` = herda o
  global), a mesma forma de `localPort`: um campo numérico nulável custou
  VerifyError e decode quebrado (medido) e, de texto, um estado antigo
  decodifica `null` (tratado como "herda"). Forma inválida (0, negativo,
  lixo) → **400 KCF142** com Hint apontando o global; o Edge aplica o teto
  do serviço antes do relay (`serviceMaxBodyCap` pura, recuo seguro para o
  global) — acima do da rota → `413 KCF110`, o mesmo código do teto global.
  O teto entra na config mutável (`serviceConfigEquals`) — republicar com
  outro valor é drift no `--plan`/`--check` — e `serviceJson` o expõe. Prova:
  `tests/route_cap_test.kf` (2: `route-cap-unit` — pura, recuos, igualdade,
  validador; `route-cap-e2e` com túnel real: corpo de 16 bytes → **413** no
  serviço com teto 8 e **200** no que herda o global, mesmo corpo e mesmo
  túnel; teto 0 recusado com KCF142).
* **Slice 3 — allowlist por serviço ✅.** `allowMethods`/`allowPaths` (CSV)
  opcionais no `POST /v1/services`; persistidos no `Service` e aplicados no
  Edge **antes do relay**: fora da lista → `403 KCF111` +
  `X-KofCloud-WAF: blocked` + métrica `_waf_allow_denied` (sem stream, sem
  Agent). Fail-closed só para quem declarou — lista vazia = sem restrição
  (serviço legado passa). Política malformada não publica: `400 KCF113`
  (método: só verbos exatos em maiúsculas) / `400 KCF114` (path: absoluto
  `/...`, sem espaço). Prova: `tests/waf_allow_test.kf` (5 testes:
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
  importa aqui. O cadastro implícito da CLI (`loginOrCreate`) respeita o
  mesmo teto: a falha limitada (`KCF134`) sobe intacta, em vez de cair no
  cadastro — que criava conta para handle bloqueado — ou ser mascarada como
  credencial inválida. Prova: `tests/login_limit_test.kf` (3).

**Estado: slices 1, 2 e 3 ✅ implementados.** Desenho fatiado (cada slice é
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
  listagem; o token da conta continua só na memória da aba. **Webhook ✅
  (F17 s3 pela API):** o último recurso da conta sem formulário na página —
  a seção chama a MESMA rota `POST /v1/webhook` do CLI (`action`
  set/show/remove/test), mostra só a URL e o prefixo do segredo
  (`secretHint`; o valor nunca volta pela API), o segredo digitado sai do
  campo após configurar, remover pede `confirm()`, e o carregamento é
  separado do Carregar porque `show` sem nada é `404 KCF130` por desenho
  (vira placeholder, nunca erro que impediria as listas). Prova:
  `tests/panel_actions_test.kf` (22: o HTML expõe cada ação, o e2e
  publica/suspende/emite/remove pelas rotas que o painel usa, o contrato do
  "Adicionar domínio", o "Criar conta"/registrar device pela mesma rota da
  CLI, o webhook com entrega real num receptor, o Diagnóstico, o banner, o
  fluxo guiado do slice 3 e o rodapé derivado de fonte única anunciando
  exatamente as 21 rotas que a página usa).

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
  **Bug corrigido (vocabulário do domínio).** A tabela de domínios comparava
  com `'ACTIVE'` — vocabulário que a API nunca devolve (`CustomDomain` é
  PENDING/VERIFIED) — então todo domínio VERIFIED aparecia amarelo e o
  operador não distinguia verificado de pendente. Agora VERIFIED é verde,
  PENDING é amarelo e entradas sem status (zona wildcard, hostname de
  serviço) são `-` sem cor inventada, nas duas renderizações (`carregar` e
  `carregarListas`). Junto: o `select` de dispositivos era reconstruído DUAS
  vezes por carregamento (chamada duplicada na mesma função). Prova:
  `tests/panel_actions_test.kf` (trava do vocabulário no HTML + e2e da
  listagem PENDING; mutação revertendo a comparação quebra).

  **Gap fechado (criar conta pelo painel).** O escopo do slice 2 prometia
  "criar conta ... tudo pelo painel", mas o painel só tinha o campo de colar
  token — criar conta exigia terminal. Agora a seção Conta tem o formulário
  (usuário/email/senha) chamando a MESMA rota pública `POST /v1/accounts`
  da CLI, com guarda local (vazio nem chama a rota) e o token devolvido
  aparecendo UMA vez (para colar no campo acima) e nunca gravado (sem
  `localStorage`, sem URL — a regra já valia para o `kfs_` na emissão).
  O painel segue sem endpoint privilegiado. Prova:
  `tests/panel_actions_test.kf` (trava do formulário no HTML + e2e do corpo
  exato, do token que autentica, da senha curta 400 e do handle repetido
  409; mutação da rota quebra).

  **Gap fechado (registrar device pelo painel).** O mesmo escopo prometia
  "registrar device ... tudo pelo painel", mas a seção Dispositivos só
  suspendia/reativava/revogava — registrar exigia terminal. Agora o botão
  "Registrar" reserva o NOME pela MESMA rota pública `POST /v1/devices` da
  CLI, com guarda local (vazio nem chama a rota). A chave pública verdadeira
  nasce na máquina — o corpo do fio exige uma, então vai o marcador honesto
  `aguardando-maquina` (nunca verificado no handshake; nem painel nem CLI o
  exibem). A máquina assume o dispositivo com `login --device <nome>`: o
  dedup por nome entre ativos devolve o MESMO id, então o publish feito pelo
  painel já espera a máquina certa. O painel segue sem endpoint
  privilegiado. Prova: `tests/panel_actions_test.kf` (trava do formulário no
  HTML + e2e do contrato de adoção — re-registro com a chave da máquina
  devolve o mesmo id, publish no pré-registrado funciona, nome vazio é 400;
  mutação da rota quebra).
**Recusa de alvo inexistente ✅.** Na leitura do JS do painel, os botões que
resolvem o alvo por NOME (suspender/reativar serviço, emitir e revogar
`kfs_`, remover serviço, suspender/reativar/revogar dispositivo) mandavam
`id: idPorNome(...)`, que é `""` quando o nome digitado não bate com a
lista — o control plane respondia 404/500 com corpo opaco e o operador não
sabia qual campo estava errado. Agora o botão recusa localmente com
`alvoAusente()` ("alvo de suspender nao encontrado: informe o nome do
servico exato (veja a lista acima)") sem chamar a API; as puras
`panelTargetName`/`panelTargetRefusal` guardam o veredito fora do DOM.
Prova: `tests/panel_actions_test.kf` (`panel-gate`: puras + trava de fonte
de cada chamada protegida).

* **Slice 3 — onboarding guiado.** Criar conta → baixar desktop client →
  publish do primeiro serviço em 3 telas, com diagnóstico embutido
  (reaproveita a lógica do `doctor`, camada por camada). Prova: fluxo
  completo num control plane de teste.

  **Primeiro passo ✅ (diagnóstico embutido no painel).** A seção
  Diagnóstico roda, pela API pública `/v1` (nenhum endpoint novo), as
  camadas do `doctor` que a página enxerga: ciclo de vida do dispositivo
  (REVOKED = FALHA orientando re-registro; SUSPENDED = FALHA apontando o
  `device unsuspend`), túnel confirmado pelo Edge (`/v1/presence` sem
  sessão = FALHA com "rode: kof cloud connect") e cadeia de auditoria
  (`/v1/audit/verify`: cadeia quebrada e espelho comprometido = FALHA,
  integra = ok com contagem de eventos) — mesmo vocabulário ok/FALHA da
  CLI. Limite honesto na página: o que depende da máquina do usuário
  (portas locais, DNS da sua rede) segue no `kof cloud doctor`. **Quarta
  camada (credencial da conta): o Diagnóstico julga o MESMO sinal do fio
  que o `doctor` e o `token list` (campo aditivo `expiring` do `/v1/tokens`,
  inventário sem valores — só a contagem vira texto, nunca o `kfc_`) e vira
  FALHA com `token rotate --ttl` antes de a sessão morrer em dias; sem TTL,
  nenhuma entrada marca e a camada é ok.** Prova:
  `tests/panel_actions_test.kf` (trava do HTML + e2e das quatro rotas no
  estado que o painel renderiza, incluindo rotate com TTL 24h que marca só
  a entrada nova; mutação da rota de auditoria quebra, mutação que tira o
  `/v1/tokens` do `diagnostico()` quebra).

  **Segundo passo ✅ (banner do próximo passo, com presença confirmada).**
  O painel agora aponta o que falta na conta, derivado do que a página JÁ
  buscou (devices, serviços e presença — nenhuma consulta nova, nenhum
  estado inventado): sem dispositivo = passo 1 (botão Registrar, ou
  `kof cloud login --device <nome>` na máquina); dispositivo no cadastro
  com NENHUM no `/v1/presence` = **máquina desconectada** — aponta o
  `kof cloud connect` **mesmo com serviço publicado**, porque a rota
  pública não responde e o banner antigo sumia dando a conta por no ar
  (o cadastro nunca é prova de túnel; a presença do Edge é); device no ar
  sem serviço = passo 2 (publicar); com serviço e túnel vivo a conta está
  no ar e o banner some. O banner se atualiza no carregamento inicial e
  após cada ação (idempotente: `innerHTML` sobrescreve, nada acumula; as
  duas chamadas passam a lista de presença que o mesmo carregamento já
  buscou). Prova: `tests/panel_actions_test.kf` (trava do banner e das
  duas chamadas + e2e que percorre os estados — `[]` → device sem sessão
  → serviço publicado com túnel fora; nenhum agente conecta, a presença
  nunca contém o device; mutação removendo o `online` da chamada do
  carregamento quebra).

  **Fluxo guiado ✅ (as 3 telas, opt-in).** A seção "Começar" liga o modo
  passo a passo: Tela 1 (conta — criar ou colar token e Carregar), Tela 2
  (máquina — registrar + adotar com `kof cloud login --device <nome>` e
  `kof cloud connect` rodando; o painel não instala nada: o cliente é o
  próprio kof) e Tela 3 (primeiro serviço — publicar com o serviço
  rodando na máquina; o diagnóstico confirma o túnel). A tela atual é
  derivada do estado real que a página já tem — `pronto` só vira
  verdadeiro no sucesso do Carregar, depois devices, depois serviços —
  sem estado inventado, sem endpoint novo e sem formulário duplicado:
  cada tela reusa as seções e funções existentes, com `Verificar`
  (`carregarListas`) e `diagnostico()` embutidos por camada. O Sair
  reseta o fluxo (o token da próxima sessão revalida). Prova:
  `tests/panel_actions_test.kf` (trava da derivação `if(!STATE.pronto)`
  + contagem das 3 re-renderizações; mutação do `pronto` quebra; e2e
  percorre o fluxo completo num control plane de teste).

  **Higiene travada (fonte única da renderização).** As três tabelas eram
  renderizadas por código duplicado (carregamento inicial × recarregar
  após ação) — e o custo já foi pago: o vocabulário do domínio precisou ser
  corrigido nas duas cópias e o select era reconstruído duas vezes pelo
  mesmo motivo. Agora `renderListas` dita o HTML das tabelas uma única vez
  e os dois caminhos a chamam; a trava existe: a definição + 2 chamadas e
  o ramo `'VERIFIED'` exatamente UMA vez no HTML (duplicar de volta quebra).

  **Sair encerra a sessão no servidor.** O token colado na página vivia para
  sempre: fechar a aba não revoga nada, e só sobrevivia o `logout` da CLI
  porque ela auto-revoga — o painel não tinha equivalente. O botão Sair
  chama a MESMA rota pública `POST /v1/tokens/revoke` com o próprio token
  antes de limpar a memória da aba (tabelas, select, banner, diagnóstico e
  segredos exibidos); token já morto (`KCF004`) também encerra; qualquer
  outra falha avisa "o token segue válido" e aponta `kof cloud token revoke`
  sem limpar — limpar faria o operador acreditar numa revogação que não
  aconteceu. Sem endpoint novo. Prova: `tests/panel_actions_test.kf`
  (travas do HTML + e2e: auto-revoga 200 → a credencial morre 401; mutação
  na rota quebra).

  **Contenção ✅ (F17 s2 pela API):** a última ação destrutiva que exigia
  terminal era o `freeze` — o painel de gestão completa prometia "tudo pelo
  painel". A seção Contenção tem `<select frztarget>` (serviços e devices
  por id resolvido das listas que a página já buscou + "conta inteira",
  opções via DOM como o select de devices) e chama as MESMAS rotas
  `/v1/freeze` e `/v1/unfreeze` da CLI (nenhum endpoint privilegiado); o
  freeze pede `confirm()` porque REVOGA as credenciais vivas (o unfreeze
  restaura o status, nunca as credenciais — a página diz "emita novas com
  `service rotate`"); o rodapé (`panelRoutes`) ganha as duas rotas e a
  trava bidirecional as exige. Prova: `panel-freeze` em
  `tests/panel_actions_test.kf` (22: freeze 200 com incidente, inventário
  de kfs_ fica `[]`, serviço SUSPENDED fora da rota, unfreeze restaura com
  `restored:"1"`, segundo unfreeze 409 KCF141, incidente listado na
  `/v1/incidents`).
**Regras:** o painel nunca mostra `kfs_` depois da emissão (mesma regra da
CLI); nunca loga corpo/token; é estático + `fetch`, sem framework pesado
(o Kof serve arquivo + JSON; o resto é HTML/JS simples).

## Fase 13 — desktop client (daemon que sincroniza com a nuvem)

**Motivação:** hoje o "client" é `kof cloud connect` em foreground ou uma
unit systemd manual no Linux. Usuário real quer instalar, logar uma vez e
esquecer — com tray, reconexão visível e update assistido.

**Estado: slice 1 ✅; slice 2 (tray + auto-update) e slice 3 (sincronização ✅ via `deploy --watch`, auto-update pendente) parciais.**
Desenho fatiado:

* **Slice 1 — instalador por SO ✅.** `daemonKind()` (config
  `kofcloud.daemonKind` ou detecção do SO) escolhe o artefato:
  `daemonUnitText` (systemd), `daemonLaunchdPlist` (launchd, `KeepAlive` +
  `RunAtLoad`) e `daemonWindowsScript` (`sc create ... start= auto`). O
  `daemon install` grava no caminho do SO (`daemonArtifactPath`) e imprime a
  ativação (`daemonActivateHint`) — **nunca** ativa sozinho; no Windows o
  script pede Administrador. Prova: `tests/daemon_test.kf` (`daemon-os` ×4: testes puros, sem instalar nada).
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

  **Implementado (sincronização — `deploy --watch`; auto-update pendente):**
  o manifesto local é a fonte — `kof cloud deploy --watch` re-lê o arquivo a
  cada passo (`kofcloud.limits.deployWatchMs`, default 2 s, recuo seguro
  `deployWatchMsFrom` em `util/Limits.kf`, junto dos outros passos de laço — 0/negativo viraria busy-loop lendo/parseando) e,
  quando o TEXTO muda, aplica com a MESMA semantica do deploy (build →
  artifact → publish idempotente), serviço a serviço com falha isolada; o
  primeiro apply é imediato. O que o watch NÃO faz, por desenho: **nunca
  remove** (remoção é destrutiva e um editor pode apagar por acidente — o
  `down --prune` continua sendo o comando explícito) e não aplica arquivo
  vazio/ilegível (editor gravando pela metade não vira estado). Stop:
  sentinela própria `~/.kofcloud/deploy-watch-stop` (separada do stopFile
  do Agent — encerrar o watch não pode parar o `connect` de alguém), mesma
  técnica de polling (L3: sem watch do SO). Decisão pura `watchDecision`
  (stop vence, vazio espera, igual não aplica, mudou aplica). **Junto,
  fatoração:** `loadManifestFile` delega na nova `loadManifestText` (o parse
  é um só; o watch parseia o texto que já leu). `--watch` não combina com
  `--check`/`--plan` (o watch aplica; os outros só olham). Auto-update
  assinado segue pendente (Ed25519 utilizável continua preso no stdlib, L17).
  Prova: `tests/deploy_watch_test.kf` (4: decisão pura, recuo do passo,
  travas do help, e2e com tunel real — publica, re-aplica idempotente,
  publica o novo SEM remover o velho).

**Regras:** enable/start sempre com consentimento explícito (como hoje);
nunca auto-update silencioso sem assinatura válida; segredos ficam no
keychain do SO quando houver (fallback: `~/.kofcloud` com permissão
best-effort, como hoje).

## Fase 14 — baixa latência (cloud rápida de verdade)

**Motivação:** hoje há UM Edge (`serve` único) e relay em chunks de 16 KiB
(antes com `bytesToString` O(n²) no caminho; desde o slice 3 o base64 é
linear em bytes). Funciona, mas latência é distância + custo por byte —
cloud rápida precisa dos dois endereçados com medição.

**Estado: slice 1 ✅ (latência + tamanho); slice 2 ✅ (ranking de Edges); slice 3 parcial (base64 e leitura linear ✅, handshake comprimido pendente).**
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
  (`metrics-latency`). **p50/p95/p99 AGREGADOS ✅ (F14 s1, sem
  cardinalidade):** o histograma do Edge já cai em faixas; `percentileFromBuckets`
  (nearest-rank puro, `edge/Metrics.kf`) converte as contagens em um **teto de
  faixa** (`<=250 ms`, `>+2000 ms`) — nunca um valor exato inventado (o
  histograma não guarda amostras) e `sem amostras` quando não há. A CLI lê o
  fio já baixado (`expositionCounter`/`latencyBucketCounts` puros; a stdlib
  **só exporta faixas não-zero** — medido —, então ausente = 0) e
  `kof cloud metrics --percentiles` imprime o bloco. Prova:
  `tests/percentile_test.kf` (3: pura nas bordas/excedente/vazio, parser com
  exposição real medida, e2e com relay real — 5 pedidos nas faixas, p50 é um
  teto e o bloco não diz "sem amostras"). Por hostname continua deliberado
  fora (privacidade).

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
  (`edgeHost`/`edgePort`) continua. **Reavaliação ✅ (parte observável):**
  o `agentRefresher` já chamava `fetchEdge` a cada ciclo, que sonda e
  ranqueia todos os Edges — e descartava o ranking. Agora `EdgeEndpoint`
  carrega o vencedor (`best`) e o refresher emite um evento `info` no
  runtime log quando o ranking aponta outro Edge que o da sessão
  (`edgeDriftLine`, pura; override do operador nunca é drift). Migrar
  sessão significa matar o túnel, então NÃO trocamos sozinhos: o operador
  vê em `logs --runtime` e reinicia o `connect`. Pendente: troca
  automática de sessão e o Edge dedicado por conta.
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
  acima e abaixo do real). **Pendente:** o handshake comprimido.

  **Implementado (base64 linear em bytes — o ganho O(n) de verdade).** O
  codificador deixou de ser `bytesToString` (concatenação `out = out + c`,
  O(n²)) + `encoding.base64Encode` do stdlib. `protocol/Bytes.kf` agora tem
  tabela RFC 4648 própria, mapeando cada byte direto da tabela e
  concatenando em blocos de ~3 KiB por `String.concat` (no JVM, o
  `java.lang.String.concat` nativo) — custo linear no número de bytes. O
  decodificador também é próprio (tolerante, sem passar por String). Prova
  com número antes/depois medido no mesmo processo: **100 KiB 1137 ms →
  46 ms (~25×)**; 512 KiB em 331 ms; 9000 bytes → 12000 chars (sem a
  expansão UTF-8 que o stdlib fazia). Testes: `tests/protocol_test.kf`
  (`bytes: base64 e byte-exato`, `base64 decodifica tolerante`),
  `tests/frames_test.kf` (1.5 MiB em chunks continua recompondo byte a
  byte). O bloco de envio (16 KiB) agora existe por latência/memória, não
  para escapar de O(n²).

  **Implementado (leitura do pedido e conversão bytes→String lineares).** Dois
  quadráticos do caminho público, medidos antes/depois:
  1. **Acumulação do pedido.** `readRequestBytes` fazia `acc = bytesConcat(acc,
     chunk)` a cada bloco lido — custo quadrático no número de blocos, e o
     teto de leitura (16 MiB) é justamente quem deixa o número crescer.
     Agora anexa num buffer com folga (`Bytes.bytesGrown`, dobra quando enche,
     devolve o exato no fim via `bytesSlice`). Medido em bench: **1 MiB em
     blocos de 8 KiB 4815 ms → 16 ms (~300×)**.
  2. **Corpo inteiro convertido para o WAF.** `bytesToString(request.body)`
     era chamado e só DEPOIS o teto de 64 KiB descartava o excedente — e a
     conversão era `out = out + c` por caractere, O(n²). Agora `wafBodyPrefix`
     corta no teto antes de converter e `bytesToString` concatena em blocos de
     ~3 KiB (linear). Medido fim a fim (upload de 512 KiB pelo túnel):
     **~22,4 s → ~0,7 s**; a leitura em si já entregava os blocos em ~50 ms.
  Junto, bug do `concat` final: quando o total era múltiplo exato do bloco, o
  `concat("")` do fim devolvia o buffer interno do concat com folga (medido:
  200 KiB viravam 400 KiB). Agora o bloco vazio não é concatenado — o mesmo
  guard vale no base64.
  Sem mudança de protocolo, de estado ou de semântica do WAF (os bytes além
  do teto nunca eram varridos). Testes: `tests/protocol_test.kf`
  (`bytesGrown` pura e `bytesToString` linear em 200 KiB) e
  `tests/framing_test.kf` (`framing-accum`: 512 KiB em blocos de 16 KiB com
  pausa entre eles chega byte-exato ao serviço local). **Pendente ainda:**
  handshake comprimido e `bytesConcat` na reconstrução de resposta.

  **Implementado (SERVICES empurrado — caminho hot do hot-reload):** o
  hot-reload pagava uma requisição HTTP ao control plane a cada ciclo
  (`agentRefresher` → `POST /v1/connect`, a cada 6×healthMs), mesmo sem
  mudança nenhuma. Agora o Edge compara a lista do device com a última
  enviada (serialização única `serviceListJson`) e manda SERVICES pelo
  PRÓPRIO túnel quando muda (`tunnelServicesPusher`, passo
  `kofcloud.limits.servicesPushMs`, default 2 s, recuo seguro
  `servicesPushMsFrom` — 0/negativo viraria busy-loop serializando a lista
  de cada device sem parar); o Agent
  aplica com `applyServicesFrame` (pura; lista ilegível é fail-open para o
  próximo push, nunca derruba o túnel; **não renova o idle** — frame de
  controle não é tráfego, a mesma mentira que o PING já evita). Publish,
  unpublish, suspensão e mudança de config aparecem no Agent em até um
  ciclo do pusher, sem o Agent falar com o control plane. O refresher HTTP
  continua como **fallback** (control plane antigo, Edge sem pusher) — os
  dois caminhos escrevem na MESMA tabela da sessão (`setServices`), então a
  ordem não importa. **Junto, bug real do refresher:** dormia o gap inteiro
  (6×healthMs) num único `time.sleep` e mantinha a task (e a JVM dos testes)
  viva minutos após a sessão morrer; agora dorme em passos
  (`kofcloud.limits.refreshPollMs`, default 1 s, recuo seguro) checando
  `session.open`, com o fetch no ritmo do gap. Prova:
  `tests/services_push_test.kf` (2 testes: aplicação pura + e2e com
  healthMs=60000 — o refresher não roda na janela do teste, então o
  publish/remoção que roteia veio pelo push) e `limits-push` em
  `tests/limits_test.kf` (11 testes).

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
  **Implementado (readiness + fail-closed — o nó não mente sobre perda de
  dados):** `/v1/health` respondia `200 ok` mesmo com o arquivo ilegível —
  o balanceador e a status page davam verde sobre a perda. Agora a sonda é
  **pura** (`readinessFor(loadStatus)`, testável sem processo) e o fio
  devolve **503** com `ok:false`/`ready:false`/`state:corrupt` (estado
  `absent` — primeiro boot — e `ok` seguem 200 com `ready:true`); o
  `serve` **recusa subir** com o estado ilegível (`serveCorruptRefusal`,
  exit 1: cita o arquivo, garante que ele **não foi tocado** e aponta o
  `state-backup`) — antes subia VAZIO e publicar criava conta nova por
  cima da perda. Nenhum corpo carrega conta/credencial. O estado corrupto
  pós-partida (arquivo trocado em disco com o processo vivo) já tinha a
  status page; agora o health também conta. **Divergência no disco ✅
  (s3-cont):** outro processo com o mesmo `--state` (nó antigo ainda vivo,
  editor) faz o próximo `save()` sobrescrever o que lá estava —
  `Registry.stateChangedOnDisk` (texto carregado × arquivo) e
  `readinessForWith` respondem **503 `state:drifted`** (degrau para fora da
  rota, melhor que derrubar o que servia; corrupção tem precedência) e o
  `doctorControlLine` diz "outro processo, NÃO faça backup" — o 503 de
  divergência e o de corrupção pedem ações opostas. Prova:
  `tests/readiness_test.kf` (4: sonda pura nos três estados + recusa do
  texto, e2e com o Registry carregando arquivo ilegível — 503 pelo fio e
  `cmdServe` exit 1 com o arquivo intacto —, sonda de divergência pura com
  a linha do doctor e e2e com escritor externo → 503 drifted). **Doctor ✅:** o `http.get` lança
  em ≥500, então o 503 do estado ilegível virava "control plane inacessível
  ou fora do ar" — rede quando o problema era o ARQUIVO. A camada agora é
  pura (`doctorControlLine`): 503 nomeia o estado ilegível e aponta o
  `state-backup`; falha de rede segue "inacessível"; corpo sem `ok` é
  "resposta inesperada" (nunca verde). Prova: `doctor-control` em
  `tests/doctor_test.kf` (puro em 4 casos + e2e com control real corrupto).

  **Implementado (`serve --backup-dir` — a parte sem banco do backup
  automático):** o estado em arquivo é o ponto único de falha do MVP
  (17-limitations); atrasar a assistência por causa do banco deixava o
  operador sem nada depois de uma subida que tirou o control plane do ar.
  Agora, com `--backup-dir <dir>`, cada subida do `serve` grava uma cópia
  do estado em disco no nome canônico `<estado>-<stamp>.json` (stamp de 16
  dígitos: ordem lexicográfica = temporal) e **poda acima do teto**
  (`kofcloud.limits.stateBackups`, default 5 — disco cheio derrubaria o
  `save()` do estado), com prefixo terminado em `-` para os backups de
  `control-state` nunca se misturarem aos de `control-state-old`; e só com
  o arquivo **estável** (idade ≥ `kofcloud.limits.backupStateMs`, default
  30 s — o Kof não expõe mtime, `stat -c %Y` decide; `stat` indisponível =
  backup segue). Estado ilegível não é copiado (o fail-closed já recusou) e
  falha de escrita vira **AVISO** no banner, nunca derruba o serve. A linha
  do banner é pura (`stateBackupLine`). Prova: `tests/backup_dir_test.kf`
  (3: poda sobre nomes com outro estado sem colisão e recuo seguro dos
  limites; banner nos três estados; subida real sem estado → nada, estado
  recente barrado pela idade, cópia do estado em disco, teto respeitado e
  arquivo ilegível sem backup).
  **Implementado (retry offline — a máquina sem rede não acorda a cada
  segundo):** notebook no avião, Wi-Fi trocando, VPN caindo — tudo virava
  "control plane inalcançável" e o laço de reconexão batia na base do
  backoff (500 ms, crescendo) contra uma rede que não existe (medido: o
  retry continuava no ritmo do backoff normal por todo o período offline).
  Agora o `runAgent` sonda a rede a cada volta (`agentNetworkUp`: nome do
  Edge fixo, senão o host do control plane): sem rede, a espera é o passo
  fixo de `offlineRetryMs` (30 s default, `kofcloud.limits.offlineRetryMs`,
  recuo seguro) em vez do backoff crescente, a linha do evento diz "sem
  rede local" em vez de culpar o Edge, e na volta a rede avisa "rede local
  voltou" e o backoff recomeça do zero. Prova: `tests/backoff_test.kf`
  (6: passo fixo nas voltas offline, backoff normal online, recuo do
  passo, linhas, sonda real com `.invalid`, dica de credencial recusada; 3 mutações — sem passo fixo,
  sem recuo, sem delegação do `webhookUrlHost` — quebram).
  `urlHost` em `util/Text.kf` é a extração genérica de host (o
  `webhookUrlHost` delega nela — mesma regra, sem duas cópias).

  **Implementado (credencial do control recusada não vira "rede"):** a
  `kfc_` revogada/vencida volta 401 `KCF004` do control plane, e o laço de
  reconexão do Agent tratava como falha transitória — martelava o control
  em silêncio e o operador só via backoff, nunca "faça login". Agora
  `isControlAuthError`/`systemHint` (puras) reconhecem o erro e o laço emite
  um evento que diz o que fazer (`kof cloud login <handle>`); NÃO é fatal —
  o login resolve e o próximo ciclo reconecta sozinho. Prova:
  `tests/backoff_test.kf` (`backoff-authhint`).

  **Implementado (falha de bind visível — a borda não some em silêncio):**
  os três listeners do `serve` (control, túnel, público) abrem DENTRO de
  `spawn`; se uma porta estivesse ocupada, `net.listen` lançaria e a exceção
  **morreria na task** — o `serve` seguiria anunciando as portas como no ar,
  com a borda inexistente (medido: `cannot listen on port … Endereço já em
  uso`). Agora as três portas são testadas ANTES de qualquer anúncio/spawn
  (`prebindServeListener`: abre e fecha na hora, o listener real reusa o
  bind); a falha vira `serveBindRefusal` (papel + porta + motivo + "nenhuma
  porta foi anunciada" + a flag para trocar) e sai **exit 1** — nunca deixa
  metade da plataforma no ar. Pura na decisão (o bind real é o único IO).
  Prova: `tests/serve_bind_test.kf` (3: rótulos+recusa puros, pre-bind
  detecta ocupada/libera quando solta, e2e com a porta do túnel segurada —
  `cmdServe` exit 1 e o control plane NÃO sobe). **Pendente:** backend
  SQLite/Postgres atrás de interface (o `kof.db` existe, mas o desenho pede
  a interface primeiro).
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
  misturar contas).

  **Implementado (varredura periódica):** o detector herdava a janela da
  poda preguiçosa — numa conta ociosa ninguém valida e o vencimento passava
  sem sinal. O `serve` agora roda um laço próprio
  (`Commands.credentialExpirySweeper`, passo `kofcloud.limits.expirySweepMs`,
  default 60 s com recuo seguro `expirySweepMsFrom`; para no stopFile da
  plataforma) sobre o MESMO núcleo de poda (`Registry.sweepExpiredCredentials`
  → `sweepServiceTokensExpired`, fonte única dos dois caminhos): `kfs_`
  vencidas saem com a métrica e a sonda da conta dona, `kfc_` entra na poda
  de sempre, nada vira "revoked". Prova: `incident-expiry-idle` (laco real
  em processo: poda sem validação, mantém válidas, abre o incidente da dona,
  idempotente, para no stopFile; mutação laco-sem-varrer quebra) e
  `limits-expiry-sweep` (recuo seguro do passo). **Pendente
  (deliberadamente):** pico de 429 por hostname
  — o rate limit já conta por hostname, mas abrir incidente por hostname
  violaria a regra de não ter cardinalidade por cliente.

  **Implementado (platô do sinal):** o `raiseIncident` deduplicava enquanto
  o incidente estava ABERTO, mas qualquer evento com ele FECHADO reabria na
  hora — um ataque contínuo virava um incidente por evento (o `freeze` de
  conta abre um a cada serviço do alvo, todos no mesmo instante) e afogava
  o anel de 100 e o webhook. Agora cada (sinal, conta) tem um embasamento
  (`plateaus` no estado, **persistido**: um restart no meio do ataque não
  zera o sinal) e a decisão é pura (`model/Incident.kf`
  `signalOutcomeWith`): aberto e ainda disparando → nada; aberto e a guarda
  venceu → volta a disparar sem abrir; fechado dentro da janela
  (`kofcloud.limits.incidentPlateauWindowMs`, default 1 h, recuo seguro) →
  platô, só conta a reincidência; fechado e fora da janela → reabre como
  `<kind>_reopened` com severidade **CRIT** (reincidência não é ruído
  novo). O fio continua compatível (`kind` intacto; campo aditivo
  `baseKind`); a CLI explica a reincidência na linha do incidente. O
  embasamento é guarda de ruído, não dado de cliente (só sinal, escopo e
  instantes) e tem teto higiênico (poda os mais antigos, nunca recusa um
  incidente). Prova: `incidents_test.kf` (`incident-plateau` ×5: pura nos
  quatro estados + guarda vencida + embasamento ilegível, reincidência
  externa CRIT com restart, freeze de conta abre UM incidente e não um por
  serviço; `incident-unit` atualizado: resolver fecha e o mesmo sinal
  dentro do platô não reabre).

  **Implementado (suspensão em lote):** os detectores cobriam revoke em
  lote, mas a SUSPENSÃO não revoga nada — um token de conta comprometido
  podia suspender todos os serviços (cada site sai do ar com 404 opaco) sem
  incidente nenhum. `detectBulkSuspensions` (mesmo padrão do
  `detectBulkRevokes`: anel de auditoria, janela
  `kofcloud.limits.bulkRevokeWindowMs`, limiar `bulkRevokeThreshold` = 3)
  soma `service.suspended` + `device.suspended` da conta e abre
  `bulk_suspensions` (WARN, escopo da conta) no limiar; ligado em
  `suspendService` e `suspendDevice`; idempotente como os irmãos (um
  incidente aberto por tipo+conta e o mesmo sinal até resolvido); o freeze
  já tem incidente próprio e não passa pelo detector. Prova:
  `incident-susp` em `tests/incidents_test.kf`.

  **Implementado (túnel caindo em massa):** o `handleTunnelConnection` via
  o fim do socket como silêncio — sem GOAWAY do Agent, sem fechamento
  próprio do Edge, nada contava. Agora o fim abrupto (`EOF` do peer ou
  leitura que lança: reset violento ou close local do writer) alimenta uma
  quarta sonda (`tunnelLossProbe`, separada): ≥5 perdas em 300 s abrem
  `edge_tunnel_losses` (WARN, global). A marca `edgeClosed` na sessão (só em
  memória, como a `idleDropped` do Agent) distingue decisão do Edge de perda
  de verdade: sweeper de ociosidade e `goawayAll` marcam ANTES de fechar, e o
  destino compartilha um helper único (`tunnelDiedAbruptly`) — a palavra
  ("tunel perdido: EOF do agent (N sessao(oes) vivas)", só contagem, sem
  device/hostname/token) vive num lugar só. De brinde, o `try/catch` no laço
  de leitura fecha um buraco de liveness: leitura que lançava matava a task
  sem `unbind` — sessão presa em ONLINE com rota quebrada. Prova:
  `incident-loss` (puro: limiar, isolada não abre, sondas independentes,
  janela vencida, marca nasce falsa + travas ancoradas do helper/chamadas) e
  `incident-loss-e2e` (Edge real: HELLO válido + close abrupto conta e abre,
  volta a sonda, parada limpa via stopFile não conta nem duplica).

* **Slice 2 — contenção em um comando.** `kof cloud freeze <alvo>`
  (conta|device|service|edge): congela tudo do alvo (suspend + revoga
  tokens vivos + derruba sessões) e gera ID de incidente; `unfreeze`
  reverte passo a passo com confirmação. Congelar é reversível por
  desenho — destruir nunca é automático. Prova: e2e congela e
  descongela sem perder configuração.

  **Implementado (service|device):** `Registry.freezeService/freezeDevice`
  suspendem o alvo, revogam as credenciais vivas e abrem o incidente
  (`freeze_service`/`freeze_device`, WARN); o status anterior fica nas marcas
  (`freezeMarks`, **persistidas no estado com teto higiênico
  (`limitFreezeMarks`, default 1000, afinável por
  `kofcloud.limits.freezeMarks`) — acima do teto o freeze seguinte poda só
  as marcas inertes (alvo removido ou device REVOKED); marcas vivas nunca
  são podadas e o containment nunca falha por limite** — o restart não perde
  a reversibilidade) e `unfreezeService/unfreezeDevice` restauram exatamente
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
  mesmo guarda foi aplicado a `freezeService`/`freezeDevice`). **Freeze
  nunca toca na sessão:** o containment revoga as `kfs_` do alvo mas as
  sessões `kfc_` seguem válidas — o operador não se tranca para fora no
  meio do incidente (travado no teste de conta: `accountByToken` da conta
  congelada segue não-nulo). Prova:
  `tests/freeze_test.kf` (`freeze-unit` ×14 + HTTP). **Pendente:** alvo
  `edge` (exige coordenação multi-Edge, Fase 16).
* **Slice 3 — comunicação e post-mortem.** Status page mínima (lê os
  incidentes globais; sem login), notificação por webhook configurável
  (conta cadastra URL + segredo; payload assinado HMAC-SHA256 — o desenho
  pedia Ed25519, mas o stdlib não expõe par utilizável, SECN014), e template
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
  `tests/status_page_test.kf` (6). **Implementado (template):**
  `docs/incidents/README.md` — contrato do post-mortem (causa, impacto,
  timeline, correção, ação preventiva com dono e prazo; sem dado de
  cliente), com trava de regressão (`postmortem-template`).

  **Implementado (webhook):** `model/Webhook.kf` + `Registry.setWebhook/
  removeWebhook/deliverWebhook` + `POST /v1/webhook` (set|show|remove|test) e
  CLI `kof cloud webhook`. Quando um incidente **da conta** abre, o control
  plane faz POST do evento com `X-KofCloud-Signature: sha256=<hex>` (HMAC
  SHA-256 do corpo com o segredo compartilhado, comparado em tempo constante
  no receptor). URL https (http só em loopback, sem credencial na URL) e
  segredo ≥16 chars validados; o segredo **nunca** volta pela API/CLI (só o
  prefixo). Falha de entrega não impede o incidente de existir — e agora tem
  **prazo explícito** (`Limits.limitWebhookTimeoutSec`, 5 s, recuo seguro
  `webhookTimeoutSecFrom` — 0/negativo quebraria TODA entrega e o `catch`
  do postWebhook viraria o caminho comum): a
  entrega roda inline no caminho do incidente e o default do stdlib
  (`http.timeout`, 15 s por pedido) deixaria um receptor que aceita e não
  responde pendurar o sweeper e handlers por 3x mais que o necessário; o
  e2e usa o mesmo fixture de socket cru do timeout de upstream e separa os
  dois mundos por uma borda de 12 s. `webhook set`/`remove` também deixam
  **rastro de auditoria** (`webhook.set`/`webhook.removed`, ator e conta —
  o segredo nunca entra no evento, no anel nem no espelho WORM; remove
  real registra, o 404 de inexistente não). **Divergência registrada:** o
  desenho pedia Ed25519, mas o stdlib não expõe par Ed25519
  utilizável (SECN014, `13-kof-apis.md` L17). Prova:
  `tests/webhook_test.kf` (12: + `webhook-host`, `urlHost` genérico
  com `webhookUrlHost` delegando).
**Regras:** contenção sempre reversível; comunicação sempre antes do
fix secreto (avisa o impacto, depois corrige); todo incidente vira teste
novo (se não tem regressão, não fechou).

## Fase 18 — integrações com o mercado (plugar sem prender)

**Motivação:** ninguém vive isolado — DNS, certificados, alertas, CI/CD,
identidade e billing moram em ferramentas que o operador já usa. Cloud
robusta pluga por padrão aberto, nunca por SDK proprietário.

**Estado: 🟡 parcial (DNS ✅ s1 — webhook e `--dns-cmd`; billing ✅ s4 — export e push por webhook; segredos ✅ s5; CI/CD ✅ s3; falta OTel e SSO).** Princípio único: **a KofCloud fala o protocolo
do mercado, não o contrário.** Fatiado por conector (cada um: config por
env, credencial nunca em log, teste com fake local + e2e opcional real):

* **Slice 1 — DNS e certificados.** Provedores DNS (Cloudflare, Route53,
  outros via webhook genérico) para criar o CNAME do domínio próprio
  (`domain add --dns=cloudflare`); ACME DNS-01 quando ES256 existir
  (Fase 9 slice 3). Credencial do provedor em env, escopo mínimo
  (só zona do domínio).
  **Implementado (webhook genérico):** `domain add --dns-webhook <url>`
  fala o protocolo do mercado (POST HTTP com `{"action":"create_cname",...}`
  e `Authorization: Bearer` de `KOF_KOFCLOUD_DNS_WEBHOOK_TOKEN` quando
  setado) para um receptor hospedado pelo operador — o CNAME nasce no
  provedor que o operador quiser, sem SDK proprietário e sem lock-in.
  Mesma regra de URL do webhook de incidentes (`webhookUrlError`: https ou
  http de loopback, sem credencial na URL), mesmo prazo de entrega de 5 s;
  falha de entrega vira AVISO na CLI (exit 0) e o domínio segue PENDING —
  o caminho manual (`domain verify`) continua sendo o caminho principal.
  O receptor e o body são genéricos (hostname/service/alvo); o token NUNCA
  aparece no corpo, na saída ou em log. `--dry-run` (regra dos conectores):
  mostra o MESMO POST com o valor mascarado e não registra nem envia — e o
  caminho pegou o NPE real do `config.env` null com a var ausente (`envOrNull`
  fonte única; antes a CLI quebrava DEPOIS de registrar o domínio). Prova:
  `tests/domains_test.kf` (`domains-dnswebhook`, `domains-dnsdryrun`).

  **Implementado (provedor por comando — Cloudflare/Route53 sem SDK):**
  `domain add --dns-cmd '<cmd>'` roda o CLI que o operador escolher via
  `sh -c`, com placeholders `{hostname}`/`{target}`/`{service}` expandidos
  antes (`dnsCmdExpand`, pura) — Cloudflare (`wrangler`/`cloudflare-cli`),
  Route53 (`aws route53 ...`) e qualquer outro entram pelo protocolo aberto
  do shell, **sem SDK e sem lock-in**. Mesmo contrato do webhook: uma fonte
  por vez (`--dns-webhook` × `--dns-cmd` = erro posicional), `--dry-run`
  mostra o comando expandido e não executa, e falha do provedor vira
  **AVISO** (o domínio já ficou PENDING e o caminho manual `domain verify`
  continua) — nunca aborta nem esconde o motivo. **O alvo do CNAME é o
  hostname do serviço**, nunca o host do control plane: a execução usa o
  `target` da instrução devolvida pelo control (`dnsTargetFromInstruction`)
  e o dry-run deriva o mesmo alvo localmente (`serviceHostname`) — antes os
  dois usavam `webhookUrlHost(controlUrl)` e o provedor criava o registro
  errado (o `verify` nunca sairia de PENDING). A credencial do provedor
  fica no ambiente do operador (o comando só referencia, ex.
  `CF_API_TOKEN`), nunca no corpo/saída/log; o stdout é descartado (pode
  conter dado do provedor), só o exit code decide. Prova:
  `tests/dns_cmd_test.kf` (6: fonte única, expansão+previa, alvo do CNAME
  = host, execução real com falha honesta — inclusive o exit 127 medido de
  comando inexistente, help, e2e com efeito observável).
* **Slice 2 — identidade e SSO.** OIDC genérico (qualquer IdP:
  Google, GitHub, Keycloak): `login` abre browser, troca code por conta
  vinculada (nunca substitui `kfc_` — vira segundo fator de emissão).
  SCIM mínimo para times (provisionar/revogar conta em lote).
* **Slice 3 — CI/CD e GitOps.** Action oficial (`kofcloud/deploy@v1`):
  lê `kofcloud.json`, publica, roda `--check`, falha o pipeline se
  quebrar; `kof cloud deploy --plan` mostra o diff sem aplicar (como
  `terraform plan`). Prova: pipeline de exemplo no repo.
  **Implementado (exemplo executável):** `docs/examples/ci-pipeline.yml` —
  login com device fixo `ci` (o re-registro dedupa no device vivo: rodar o
  pipeline não gera devices novos nem estoura o teto), `deploy --plan`
  revisável, `deploy --check` falhando o pipeline; senha só via segredo
  mascarado pelo Actions → env do passo (nunca inline, nunca no log) e
  pré-requisito declarado no topo: runner **self-hosted** na máquina do
  `connect` (num runner de nuvem o `--check` não teria o que sondar).
  Trava de realidade (`deploy-ci`): o teste lê o exemplo do disco e exige
  que todo `kof cloud <cmd>` citado exista no help, as 3 fases do fluxo
  estejam lá e nenhuma senha apareça literal.
  **Implementado (action de exemplo):** `docs/examples/action.yml` — a
  action composite que o desenho chama de `kofcloud/deploy@v1`, pronta para
  copiar para um repositório próprio e referenciar com `uses:`: inputs
  (`handle`/`password`/`control-url`/`device` com default `ci`), três passos
  (login por env do passo, `--plan`, `--check`) e o MESMO pré-requisito
  honesto no topo (vale para o workflow que a usa: self-hosted na máquina
  do `connect`). A senha entra como input alimentado de `secrets.*`
  (mascarado pelo Actions) e chega ao comando por env, nunca literal.
  Trava de realidade (`deploy-action`): o teste lê a action do disco e exige
  `using: 'composite'`, que todo `kof cloud <cmd>` citado exista no help, as
  3 fases, o default `ci` e nenhuma senha literal — o laço de extração é
  fonte única do `deploy-ci` (`assertCitedCommands`). Exemplo cobre o fluxo;
  a publicação num repositório oficial segue externa (infra, não código).
* **Slice 4 — observabilidade e billing.** Export OpenTelemetry
  (traces do relay + logs estruturados) para o backend que o operador
  já tem (Grafana/Datadog); webhook de uso (requests/bytes por conta)
  para billing externo; Stripe como referência (nunca único).

  **Implementado (uso por conta — a matéria-prima do billing):** o Edge
  acumula `requests`/`bytesIn`/`bytesOut` por `accountId` no Registry
  (`noteUsage`, contadores em String por L11) **só no caminho roteado** —
  a conta vem do serviço (o Edge a conhece pelo roteamento, nunca do
  cliente) e recusas de plataforma (404/503/429) não são uso a precificar.
  Persistido no ControlState (`List<UsageEntry>` nulável — estado antigo
  carrega, mesmo padrão do freezeMarks; o `noteUsage` NÃO grava disco por
  request — o uso vai junto no próximo `save()` de mutação). Leitura:
  `GET /v1/usage` escopado pelo token (a conta alheia não existe para o
  fio; 401 sem token) e `kof cloud usage`. A camada de **export** veio em
  dois conectores: o fechamento sob demanda (`POST /v1/usage/export`) e o
  **push por webhook** (`usagePushSweeper`) — o contador já era o contrato.
  **Implementado (ciclo de billing — o export):** `POST /v1/usage/export`
  devolve a janela acumulada com `periodEnd` e **zera** o contador
  (`Registry.usageSnapshot`): cada export cobre exatamente uma janela, sem
  sobreposição nem buraco; export repetido é zeros (idempotente — nunca
  re-fatura) e o Edge continua contando em cima do zero. O fechamento de
  janela deixa rastro (`usage.export` na trilha, vocabulário fechado).
  CLI: `kof cloud usage --export`. O reset do export PERSISTE NA HORA
  (padrão do `setWebhook`/`removeWebhook`): o export é mutação de contrato,
  então o `usageSnapshot` grava o estado ao zerar — um restart recarrega a
  janela fechada e o export repetido segue zeros (não re-fatura; trava no
  reload do `usage-unit`). Resta a honestidade do ACÚMULO: um crash entre
  os `noteUsage` e o próximo `save()` de mutação perde o intervalo sem
  mutação — nunca duplica. Prova: `tests/usage_test.kf` (6:
  contadores+escopo+persistência, snapshot le-e-zera, e2e export com
  auditoria, help).

  **Implementado (push do uso — o webhook de billing):** o `serve` roda um
  sweeper (`usagePushSweeper`, passo `kofcloud.limits.usagePushMs`, default
  1 min, recuo seguro `usagePushMsFrom` — 0/negativo viraria busy-loop de
  snapshot+POST) que, para cada conta com webhook configurado e janela com
  uso, faz snapshot (`Registry.pushUsage`) e entrega o evento
  `usage.window` (`X-KofCloud-Event`) com a MESMA assinatura HMAC-SHA256
  dos incidentes (`X-KofCloud-Signature`) — o operador NÃO gerencia uma
  segunda URL/segredo, e o receptor roteia por tipo. Sem webhook ou sem uso
  = no-op (o contador não zera por push vazio). O snapshot acontece ANTES
  da entrega: falha de rede não perde a janela (já fechada; o consumidor
  deduplica por `periodEnd`, como no export). Payload: requests/bytesIn/
  bytesOut/periodEnd — o mesmo vocabulário do GET/export, sem credencial.
  **Freeze × push (decisão de desenho, medida):** uma conta congelada NÃO
  acumula uso novo (serviço SUSPENDED sai da rota antes do relay — a recusa
  não é uso), mas a janela pendente de ANTES do freeze continua sendo
  pushada: o webhook é o canal do próprio operador e o freeze entrega o
  incidente pelo MESMO receptor — cortar o push calaria o operador do
  próprio freeze. Conta e webhook vivem no mesmo Registry e não há remoção
  de conta, logo não existe webhook órfão. **Export × push (mesma fonte,
  travado):** os dois fecham a janela pelo mesmo `usageSnapshot`
  (read-and-reset persistido) — o push seguinte a um export manual só leva
  o delta da janela nova, nunca re-fatura o que o export já fechou
  (mutação com o push lendo sem zerar quebra no assert exato).
  Prova: `tests/usage_push_test.kf` (4: payload, push só com webhook e uso,
  recuo do passo com mutação, e2e — receptor real recebe `usage.window` e a
  assinatura confere contra o segredo; o caso do freeze trava o push pós-
  freeze com serviço real suspenso — mutação pulando a conta congelada
  quebra no assert exato). O export OTel (traces) segue fora
  (sem OTLP no stdlib, 13-kof-apis).
* **Slice 5 — segredos e chaves.** Provedores de segredo (Vault, AWS SM,
  1Password Connect) como fonte do `tunnelSecret`/PEMs: a plataforma lê
  na partida, nunca persiste em disco; rotação por SIGHUP/reload.

  **Implementado (fonte externa por arquivo/comando):** `serve` ganha
  `--tunnel-secret-file <arq>`/`--tunnel-secret-cmd <cmd>` e
  `--tls-cert-cmd <cmd> --tls-key-cmd <cmd>` (exclusivos com
  `--tls-cert/--tls-key`). A plataforma fala o **protocolo aberto do shell**
  (`sh -c`) — qualquer provedor (Vault, AWS SM, 1Password Connect expõem um
  CLI) vira fonte sem SDK; nada de lock-in. O valor é lido **uma vez na
  partida**, mantido só em memória, **nunca impresso** (o banner diz só a
  ORIGEM — arquivo ou "comando", jamais a linha do comando, que pode
  carregar token) e **nunca gravado** (os PEMs vão direto ao `enableTls`,
  sem arquivo temporário). Duas fontes para o mesmo segredo = erro
  posicional (nunca escolha silenciosa); comando que falha ou devolve vazio
  = erro explícito, nunca segredo vazio indo para o handler. **Divergência
  honesta:** rotação é por reiniciar o `serve` — SIGHUP não existe (L3, sem
  sinais), então o "reload" do desenho original não tem mecanismo; o
  operador rotaciona no provedor e reinicia. **Fatia: falha da fonte é
  recusa na partida, não task morta.** O tunnel secret ia por
  `readSecretSource(tunnelCmd)` e o comando só rodava DENTRO do `spawn` do
  túnel: provedor caído (Vault, token expirado) derrubava a task e o `serve`
  anunciava a borda com a porta muda — o oposto do "erro explícito" que a
  fatia promete. Agora as fontes do tunnel secret (e os dois PEMs por
  comando) são lidas ANTES de anunciar/subir: falha = linha de recusa +
  exit 1, nada no ar. Os três comandos de segredo passam por um único
  `runSecretSource` (um exec, um contrato de falha `"<label>: ..."`, mesmo
  trim, erro nunca ecoa o valor), e com arquivo/comando o env/config não
  vence a fonte que o operador escolheu. Prova:
  `tests/secrets_test.kf` (6: `secret-unit` ×3 — leitura de comando/arquivo
  com falha honesta, combinação de fontes, exec único —, `secret-usage` e
  `secret-e2e` ×2 — `ControlPlane.enableTls` com PEMs lidos por comando sobe
  HTTPS de verdade e o `serve` com `--tunnel-secret-cmd` falho recusa antes
  de subir).

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
  imprime o inventário local (49 fontes + SHA-256, sem login, sem rede).
  Prova: `tests/security_test.kf` (`security-unit` + `security` +
  `security-policy`) + `tests/hosts_test.kf` (`audit-sbom`). Pendente:
  o resto do ritmo (`rotate --all` ✅ no slice 3, `scan` ✅ no slice 2,
  PEM ✅ — aviso de expiração em 30 dias no `doctor --tls-cert` (Fase 9 s3)
  e no `scan --tls-cert`; kill-switch ✅ documentado; WORM ✅ parcial no
  slice 4 — espelho append-only + verify; storage separado pendente).
  **Relatório de controles ✅ (meta SOC2-tipo-II):** `docs/CONTROLS.md` mapeia
  cada controle (CC6.1–CC8.1, A1.2/A1.3) 1:1 com o mecanismo real (arquivo)
  e a prova (suíte/tag) e, em TODA seção, o limite honesto (**Não cobre**:
  WORM não imutável, nó único, sem MFA, HMAC em vez de Ed25519, sem artefato
  transferível). Não é certificação — é o retrato que a auditoria externa
  leria, sem checklist aspiracional. Prova: `tests/controls_doc_test.kf`
  (`docs-controls` ×4: citações de suíte existentes, toda seção com o
  limite e a contagem de seções fixa, âncoras de mecanismo e os limites
  espelhados do 17-limitations, e cada identificador em crase do mapa —
  `EdgeState`, `wafAllowCheck`, `doctor` — resolvido a uma declaração do
  `src/` ou a um comando do help: nome por parecença não vira mecanismo).
* **Slice 2 — varredura contínua (parcial: manifesto + ambiente + PEM ✅; dependências pendente).**
  `kof cloud scan [--file]` lê o manifesto e acusa `kfs_`/`kfc_`/password
  literal (exit 1 com achado, 0 limpo) — sem login, sem rede, para rodar
  no CI a cada push. Pura `scanManifestSecrets` (nomeia o serviço sem
  vazar o valor). Prova: `tests/deploy_test.kf` (`deploy-scan`) + trava
  da ajuda. **Dependências ✅ (offline por desenho):** resolver `kofdeps`
  exige rede e o CI isolado não tem — mas o que se sabe sem rede é dito:
  `scanDependencyPins` (pura) lê `kof.toml`/`package.json`/`Cargo.toml`/
  `go.mod`/`requirements.txt` no diretório do manifesto e acusa pin aberto
  (`latest`/`*`/`^`/`~`/faixa/sem versão — build não reproduzível) e pin
  exato antigo quando o comentário da linha traz a data (`# pinned
  YYYY-MM-DD`, limite `kofcloud.limits.depPinMaxAgeDays`, 365). Nunca diz
  CVE nem versão atual: isso exige rede, e um scan offline que inventasse
  "vulnerável" seria pior que nenhum. Prova:
  `tests/scan_extra_test.kf` (`scan-deps`). **`--summary` ✅ (CI legível):** o `scan` imprime uma linha
  ("scan: N achado(s) — detalhe: rode de novo sem --summary") mantendo o exit
  code; `--summary` repetido barra no uso. Prova: `scan-summary`.
  O item "`kfs_` sem TTL em serviço crítico" pousou como Higiene de
  credenciais abaixo (`kof cloud service tokens` / `POST
  /v1/services/token-hygiene`).

  **Complemento ✅:** o `scan` agora também olha o **ambiente** (lista curada
  de nomes sensíveis — o Kof não enumera env, `config.env(nome)` lê um por
  vez — mais qualquer valor com `kfs_`/`kfc_`/`password`) — e a lista inclui
  as credenciais da **própria plataforma** (`KOF_KOFCLOUD_DNS_WEBHOOK_TOKEN`,
  o token do webhook DNS da Fase 18 s1, que escapava da varredura porque não
  estava na lista — mutação removendo-o quebra no assert exato) — e o **PEM do
  operador** (`--tls-cert`: expirado ou expirando em 30 dias, via
  `openssl x509 -checkend`). Nunca imprime valor, só o nome. Prova:
  `tests/scan_extra_test.kf` (6, inclui `scan-deps`). Dependências: a parte
  offline está acima; CVE/versão atual continua fora (rede). O "`kfs_` sem TTL em serviço crítico" está fechado pela
  Higiene de credenciais abaixo — quem lista é o `service tokens`, não o
  `scan` (o scan continua local: sem conta, sem rede).

  **Higiene de credenciais ✅:** `kof cloud service tokens` (e
  `POST /v1/services/token-hygiene`) lista os serviços da conta com `kfs_`
  **válidos e sem expiração** — o item "kfs_ sem TTL em serviço crítico" do
  plano. Só leitura: a ação recomendada é `service rotate <nome> --ttl 24`.
  Escopo por conta; sem token → 401. Prova: `tests/token_hygiene_test.kf`
  (5: o caso do rotate liga as lanes — rotate sem TTL deixa o serviço no
  relatório (pendente) e rotate com TTL NÃO limpa o legado sem TTL em
  silêncio; mutação revogando o legado quebra no assert exato).
* **Slice 3 — resposta e rotação programada (✅ + `rotate --all`).**
  Rotação assistida (por serviço): `kof cloud service rotate
  (<nome>|--all) [--ttl H] [--prune]` emite credencial nova e **só revoga
  as anteriores depois do TTL** (overlap sem downtime); `--all` itera os
  serviços da conta com falha isolada por serviço (resumo ok/falhas, exit
  1 se alguma falhar); `service rotation <nome>` mostra quantas ainda
  valem. Credencial **sem** expiração nunca é revogada em silêncio — fica
  contada como pendente até ser revogada à mão. Prova:
  `tests/rotate_test.kf` (6: `rotate-all` com dois serviços independentes).
  **Credencial da CONTA (`kfc_`) com ciclo de vida ✅.** O `kfc_` é o token
  mais poderoso do sistema (publica, revoga device, congela conta) e era o
  único sem ciclo de vida: `logout` apagava só o arquivo local e o valor
  seguia válido no servidor para sempre. Agora `TokenEntry.expiresAt`
  (nulável — estado antigo decodifica como "sem expiração", JSN004 medido),
  expiração preguiçosa na validação, `GET /v1/tokens` (inventário sem
  valores), `POST /v1/tokens/revoke` (valor ou id curto; 409 `KCF063`
  ambíguo, 404 `KCF064` opaco) e `POST /v1/tokens/rotate` com overlap
  (antigas só caem depois do TTL; sem TTL nada é revogado em silêncio).
  CLI: `kof cloud token list|revoke|rotate|rotation|prune`; o `rotate` grava a
  nova credencial na sessão local antes de imprimir. O `status` mostra o
  aviso de expiracao do mesmo sinal `expiring` do fio (pura
  `statusCredentialWarning`; era o comando de rotina cego — o operador
  descobria a perda da sessão nos 401). **Logout × sessão vencida
  (travado):** token vencido não autoriza os endpoints (revoke/prune
  devolvem 401 KCF004 porque passam pela mesma validação preguiçosa), então
  o `logoutDisposition` cai em `dead` — nunca em `warn` com a mentira
  "ela segue valida". Prova:
  `tests/account_token_test.kf` (11: TTL/poda/legado, revoke por valor e id
  curto com escopo por conta, ambiguidade, overlap, aviso no `status`,
  HTTP fim a fim).
  **TTL longo (30 dias) não estourava por acaso:** `ttlHours * 3600000` era
  aritmética de `Int` (32 bits), que estoura acima de 596 h — `--ttl 720`
  nascia vencido (401 imediato). O cálculo agora é `Long` e fonte única
  (`tokenExpiryAt`, usada por kfc_ e kfs_); medido com o CLI 0.6.3
  (`720 * 3600000` em Int = `-1702967296`).
  **Acúmulo de logins ✅.** Cada `login` emite credencial nova e não revoga a
  anterior — com o TTL opcional, o inventário cresce a cada entrada e uma
  credencial vazada de um login antigo vale para sempre. `POST
  /v1/tokens/prune` + `kof cloud token prune` mantêm apenas a da sessão atual
  e revogam o resto (escopo por conta; sem token → 401; idempotente).
  **Prune × `kfs_` (travado):** a poda só toca o inventário da conta — as
  `kfs_` válidas de serviços AUTHENTICATED sobrevivem (mutação limpando as
  `kfs_` no prune quebra no assert exato; um prune de rotina que fechasse o
  acesso público a todos os serviços passaria em silêncio). Prova:
  `tests/account_token_test.kf` (11). PEM com 30 dias de antecedência ✅
  (`doctor --tls-cert`, Fase 9 s3, e `scan --tls-cert` no slice 2);
  kill-switch ✅ documentado (`docs/SECURITY.md` ## Contenção) e
  automatizado pelo `freeze` de um comando da Fase 17 slice 2.
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
  adulteração parcial, não imutabilidade. Prova: `tests/audit_chain_test.kf` (`chain-unit`
  + `chain-filter`).

  **Implementado (espelho append-only, parcial):** cada `Registry.audit`
  grava UMA linha JSON no `<estado>.worm.jsonl` (mesma serialização da API,
  pura `auditEventJson` — o fio e o arquivo nunca divergem), à época sem cap
  nem poda: sobrevive à evicção do anel (cap 100) e à rescrita inteira do
  `control-state.json` (o `save` não o toca). Best-effort por desenho: falha
  de IO jamais quebra a requisição (o anel + o estado são a fonte; o espelho
  é trilha redundante) e adulterá-lo não afeta a API. **Honestidade:** o
  append é read+write (o stdlib não expõe append atômico — ver 17-
  limitations), então quem tem o disco pode reescrever o arquivo inteiro;
  o que o espelho compra é **sobrevivência à poda e à rescrita do estado**,
  não imutabilidade física. Prova: `tests/audit_chain_test.kf` (`worm-unit`
  + `worm`).

  **Implementado (verify do espelho):** `/v1/audit/verify` e
  `kof cloud audit --verify` cobrem também o espelho — campos ADITIVOS
  (`mirrorOk`/`mirrorBrokenAt`/`mirrorEvents`; cliente antigo continua
  decodificando os três originais). A cadeia do arquivo tolera RAIZ
  DESCONHECIDA (a 1ª ocorrência de cada conta pode apontar para evento
  anterior ao espelho — deploy antigo não vira falso "quebrada"), mas pega
  adulteração de campo, remoção do meio e linha ilegível (`parse`). A CLI
  sai 1 com o espelho comprometido (trilha redundante) e diz
  "desconhecido" contra control plane antigo, sem os campos. Prova:
  `tests/audit_chain_test.kf` (`worm-unit` + `worm`).

  **Implementado (teto do espelho — "rotação" sem rename):** o espelho tem
  teto em bytes (`kofcloud.limits.wormMirrorBytes`, default 8 MiB — recuo
  seguro da família `wormMirrorBytesFrom`): acima dele o append poda a
  CABEÇA do arquivo em linhas inteiras, sempre mantendo o evento mais novo
  (pura `wormMirrorNext`; `appendLineCapped` guarda o contrato never-throw).
  Sob o teto o append é byte a byte o anterior — nada muda para deploy
  existente. As raízes desconhecidas que a poda produz são toleradas pelo
  verify de sufixo por desenho; a poda remove eventos antigos do espelho
  (indistinguível de remoção por quem tem o disco — 17-limitations).
  Prova: `tests/audit_chain_test.kf` (`worm-unit` ×6 + `worm` ×3).
  Pendente: WORM real em storage separado (objeto
  com lock de retenção) e o relatório SOC2-tipo-II.

  **Implementado (vocabulário fechado):** toda ação de audit vive nos
  construtores de `model/Audit.kf` — o último literal cru do código
  (`service.token_issued`, único sem construtor) virou
  `auditServiceTokenIssued()` e a trava de fonte (`chain-unit`) quebra se
  um literal de ação reaparecer fora de Audit.kf ou se a emissão deixar de
  delegar: o `audit --action` casa exato, então literal cru escapava do
  vocabulário (typo viraria ação invisível no filtro). **Privacidade do
  anel** (trava do worker estendida, `worm-privacy`): com emissão e
  revogação real de `kfs_`, nem o arquivo nem o FIO `/v1/audit` contêm
  `kfs_`/`kfc_`/senha — o anel guarda só metadados e o fio é a mesma
  serialização do espelho; se um dia vazasse, o `scan` do CI pega. Prova:
  `tests/audit_chain_test.kf` (21).
**Regras:** segurança por default em todo slice novo (a pergunta "o que
pode vazar aqui?" entra no template de teste); nenhum dado de cliente
em log/métrica/audit — sem exceção, sem "só dessa vez".

## Ordem de execução no MVP

```
Fase 1 → Fase 2 → Fase 3
```

validando cada etapa. Nada de Fase 4+ antes de a Fase 3 estar estável.
