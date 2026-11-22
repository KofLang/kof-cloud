# Limitações e lacunas honestas

Este documento lista o que **não** funciona ou não está implementado, sem
esconder atrás de workaround. É a contraparte de `13-kof-apis.md`.

## Lacunas do runtime Kof que moldam o produto

| ID | Lacuna | Impacto | Contorno |
|---|---|---|---|
| L1 | `Conn`/`Listener` não são classes: quebram como campo e como retorno de `spawn` | sessões não podem guardar o socket num objeto | `Conn` por parâmetro/captura; `Channel` em `Map` |
| L2 | sem TLS em socket cru; sem ACME | Edge não termina TLS sobre `net` | `web.app().listenSecure` (JVM) ou terminação externa |
| L3 | sem watch, sem locks, sem chmod | permissão de arquivo é best-effort; concorrência por canais | polling; canais |
| L4 | `String.getBytes()` só JVM (`STR003`) | payload binário não é portátil | `new Byte[n]` + atribuição |
| L5 | `main(args)` vazio em Native/JS | CLI é JVM-first | — |
| L6 | WebSocket server-side é JVM-only (`WEB004`) | WebSocket público fora do MVP | — |
| L7 | sem chunked/streaming HTTP | upload/stream longo limitado | fragmentação + idle timeout |
| L8 | `web.app()` reserva `Authorization` para JWT e 401s antes do handler | tokens opacos não podem viajar em `Authorization` no control plane | header próprio `X-KofCloud-Token` |
| L9 | `http.*` devolve o corpo em 4xx (só lança em ≥500) | erro de API passa silencioso | `requireOk(raw)` transforma `{"error":…}` em exceção |
| L10 | sem sinal nem timeout de `accept` | parada de servidor | fechar o `Listener` (sentinela + watcher) |
| L11 | `Map<String, Int>` como campo de classe quebra o verificador JVM (`VerifyError`; `Long` passa) | contadores Int em mapas precisam de outro tipo | `Map<String, String>` + `parseIntSafe`/`intToString` (padrão batido do codebase) |
| L12 | `await` com valor de retorno quebra o verificador JVM (`VerifyError`) | task com retorno não pode ser esperada com valor | `await` só em task sem retorno; passar resultado por canal/arquivo/socket |
| L13 | `web.app()` fecha a conexão sem headers para corpo de ~1.5 MiB | handler com resposta grande não serve | testar payload grande com servidor TCP cru |
| L14 | EOF remoto chega como exceção (`kof.net: ... Socket closed`), não como vazio | queda silenciosa não existe: todo fim de túnel vira throw | throw → backoff + reconexão; saída limpa só com stopFile/GOAWAY |
| L15 | argumento de task (`spawn`) é copiado por valor | `Map` passado para task vira cópia: `put` dentro não aparece fora | estado compartilhado vive em objeto (classe), não em Map solto |
| L16 | o handler do `web.app()` devolve `String`: byte alto passa por UTF-8 e vira dois (`00 C8 FF 0A` → `00 C3 88 C3 BF 0A`) | Edge não pode usar `web.app()` como front (TLS/HTTPS no Edge fica bloqueado) | parser HTTP próprio sobre `net.listen`; `web.app()` só no control plane (JSON) |
| L17 | Ed25519 documentado mas inutilizável (`keyExchange.privateKey("ed25519")` lança `SECN014`) | não há assinatura assimétrica com chave gerada pela aplicação | HMAC-SHA256 no webhook de incidentes; assinatura assimétrica pendente |
| L18 | `toUpperCase`/`toLowerCase` não mudam a caixa no JVM (medido no CLI 0.6.3); `charAt(i)` comparado com literal de char não casa quando o índice vem de variável; `strings.isUpperCase` também não classifica (`"AUTHENTICATED"` → falso); `a ? b : c` não parseia; `assert cond, msg` só imprime "assertion failed" | regra de "isto é MAIÚSCULO?" não pode ser escrita na linguagem | lista fechada (`protocolWireLiterals()`) ou helper com `if`; rótulo de falha por helper que faz `throw` (`assertWith`) |

| L19 | não é lacuna do Kof: o anel de auditoria era encadeado global e verificado por conta (duas contas ativas "quebravam" a cadeia uma da outra) | verificação acusava adulteração inexistente | cadeia por conta na gravação e na verificação (`auditChainError`) |
| L20 | caixa e classe de caractere não funcionam como esperado no JVM: `"X".toUpperCase()` não muda a caixa, `charAt` comparado com literal não casa e `strings.isUpperCase` não classifica; `a ? b : c` não parseia; `assert cond, msg` perde a mensagem | regra de "isto é MAIÚSCULO?" não pode ser escrita na linguagem; falha de teste sem rótulo | lista fechada (`protocolWireLiterals()`) e helper com `if`; rótulo por `throw` (`assertWith`) — medido em `tests/controls_doc_test.kf` |
## Fora do MVP (por decisão)

* HTTP/2 e HTTP/3.
* `Transfer-Encoding` / chunked **no pedido**: recusado com `400 KCF118`
  antes de rotear — o Edge não decodifica corpo chunked de quem entra
  (repassar o header com corpo vazio faria o serviço local esperar para
  sempre). Pedido com corpo precisa de um `Content-Length` único, numérico e
  canônico; é ele que delimita o corpo repassado (bytes além do declarado
  são descartados, nunca viram uma segunda requisição). **Na resposta** o
  oposto vale desde a Fase 10 slice 2g: `Transfer-Encoding: chunked` do
  serviço local é decodificado (`decodeChunked`) e o cliente recebe o corpo
  simples com `Content-Length` real — nada de enquadramento cru
  (`tests/framing_test.kf`, `framing-chunked-unit`).
* QUIC, UDP forwarding, TCP forwarding genérico.
* CDN, edge distribuído, relay regional.
* Domínio próprio: registro + verificação + rota (Fase 6 slice 1) ✅, com
  automação DNS por provedor (`--dns-webhook` / `--dns-cmd`) ✅; sem TLS
  por SNI. A
  verificação recusa DNS que também resolve fora do Edge (`KCF164`), por
  IPv4 e por IPv6 literal (julgado pelo valor, não pela grafia). O texto do
  CNAME devolvido pelo resolver do SO não desqualifica, e o Edge precisa ser
  anunciado em `kofcloud.edgeHosts` na mesma família do endereço (AAAA do
  Edge só passa se houver um IPv6 esperado, ou se for o IPv4 mapeado).
* Deploy com **artefato transferível** — o deploy declarativo (`kof cloud
  deploy`) existe (Fase 7), mas o build roda na máquina do usuário e só o
  publish atravessa a rede.
* Daemon: **artefato por SO gerado** (systemd/launchd/Windows, Fase 13 s1) e
  ativação sempre manual; o core não depende deles. Tray/estado visível e
  auto-update assinado seguem pendentes (slices 2 e 3).
* Auto-update do Agent com verificação de assinatura. (Rotação de `kfs_`
  existe — `service rotate` com overlap, inclusive `rotate --all`, que
  grava cada credencial em arquivo em vez de imprimir o valor; rotação
  programada de PEM segue pendente; a contenção manual está documentada
  em `docs/SECURITY.md` ## Contenção.)
* Painel em **tempo real** (WebSocket) — o painel de gestão existe
  (`serve --panel`, slices 1–3: leitura, ações e fluxo guiado; prova em
  `tests/panel_actions_test.kf`); o que segue fora do MVP é a página se
  atualizar sozinha.
* Object storage, banco gerenciado, VPS, Kubernetes.

## Limites do MVP implementado

"As limitações honestas" é a função deste documento, e por isso era fácil
ler só a lista e concluir que nada de proteção existe. O que **existe de
verdade**, com prova executável, está aqui — os buracos de cada item vêm
logo abaixo, na mesma lista:

* **Rate limit por hostname:** `429 KCF107` + `Retry-After` antes de rotear,
  janela fixa por processo (`security.rateLimit`) — `tests/ratelimit_test.kf`.
* **Abuso:** suspensão reversível (`SUSPENDED`) de serviço/dispositivo,
  trilha de auditoria só com metadados e `freeze`/`unfreeze` por
  serviço/device/conta — `tests/abuse_test.kf`, `tests/freeze_test.kf`.
* **Tetos de tamanho:** corpo de requisição (`httpBodyBytes`, `413 KCF110`,
  inclusive por rota via `--max-body`), leitura absoluta do pedido
  (`maxBytes`), resposta (`maxResponseBytes` → `502 KCF115`), cabeçalho sem
  fim (`431 KCF116`), WAF inspeciona até 64 KiB — `tests/framing_test.kf`,
  `tests/route_cap_test.kf`, `tests/waf_detect_test.kf`.
* **Ciclo de vida de credencial:** `kfs_`/`kfc_` com TTL opcional em `Long`,
  revogação por token ou id curto, inventário sem valores, rotação com
  overlap (`rotate --all`), varredura de vencidas no `serve` —
  `tests/service_token_test.kf` (`token-ttl`), `tests/rotate_test.kf`,
  `tests/account_token_test.kf`, `tests/incidents_test.kf`
  (`incident-expiry`).
* **Detecção de incidente:** rajada de 401, perda de túnel, queda por
  ociosidade, expiração em massa, suspensão em lote, com webhook HMAC,
  status page sem login e template de post-mortem — `tests/incidents_test.kf`.
* **Política de segurança publicada:** `/.well-known/security.txt` no Edge
  (antes de rate limit/rota/auth) e `docs/SECURITY.md` (o que é suportado,
  SLA de reporte, kill-switch) — `tests/security_test.kf` (`security-policy`).

* **A trilha de auditoria é detectável, não imutável.** Cada evento tem hash
  encadeado (`audit --verify` acusa adulteração/remoção parcial) e há espelho
  append-only em `<estado>.worm.jsonl` (uma linha JSON por evento, mesma
  serialização da API; sobrevive à poda do anel e à rescrita do estado) —
  mas o append é read+write (o stdlib não expõe append atômico, então duas
  escritas concorrentes podem intercalar e quem tem o disco pode reescrever
  o arquivo inteiro); WORM real (storage separado com lock de retenção)
  segue pendente. O espelho agora tem teto (`kofcloud.limits.wormMirrorBytes`,
  default 8 MiB): acima dele o append poda a **cabeça** do arquivo em linhas
  inteiras, mantendo sempre o evento mais novo — é a rotação possível sem
  rename no stdlib. Honestidade: a poda remove eventos antigos do espelho
  (indistinguível de remoção por quem tem o disco) e o `audit --verify`
  tolera essas raízes desconhecidas por desenho (verify de sufixo) — trilha
  completa e imutável de verdade continua dependendo do WORM real.
* **Estado em arquivo JSON único, sem backend de banco.** A carga agora é
  explícita (`ok`/`absent`/`corrupt`, com aviso no `serve`) e há
  `migrate --dry-run`, mas ainda não existe backend SQLite/Postgres atrás de
  interface (Fase 16 slice 1, parte pendente).
* **Um nó de control plane, um nó de Edge, em memória + arquivo JSON.** Não há
  consenso distribuído nem banco. É suficiente para provar o fluxo, não para
  produção multi-nó.
* **`AUTHENTICATED` usa token de conta**, não um provedor de identidade
  federado.
* **Presença é por processo do Edge.** `/v1/presence` responde as sessões
  vivas *neste* Edge; com control plane e Edge separados, ou com mais de um
  Edge, a lista fica vazia/parcial (ausência, não mentira). Não há pub/sub
  entre nós — é o slice 2 da Fase 16.
* **O rate limit do login é por handle, não por IP.** O control plane não vê
  o IP de origem em `web.app()`, então a chave é o handle da conta: protege a
  conta de força bruta dirigida, mas um atacante que distribua tentativas
  entre vários handles (password spray) enche o inventário mais devagar e
  não é contido por este teto — é o mesmo trade-off do rate limit do Edge
  (chave hostname, por falta de IP visível).
* **Rate limiting é por processo.** Não é global entre nós.
* **`parseStatusLine` também.** O código da linha de status da sonda de
  health (`agent/Health.kf`) vai pelo mesmo `parseIntBounded` com teto 599;
  fora de 100–599 é 0 (resposta quebrada → DEGRADED), nunca um código
  inventado.
* **Auditoria de `parseIntSafe` (veredito).** Entrada externa usa
  `parsePortSafe`/`parseIntBounded`; os `parseIntSafe` restantes leem apenas
  contadores que o próprio código escreve (`Session.pending`,
  `EdgeState.serviceConns`, `Registry.expiryProbes`) — parse falho vira 0,
  que fecha a porta (fila vazia, sem vaga) em vez de abrir. O contador de
  `token prune` também foi para `parseIntBounded`.
* **Inteiros do protocolo também.** `frameInt` (campos numéricos de
  `HELLO`/`WELCOME`/`DATA`) usava `parseIntSafe`: medido, um `HELLO` com
  `v: 4294967297` fazia wrap para **1** e passava a checagem de versão, e um
  `heartbeatMs` forjado virava 1500 ms. Agora lê pelo mesmo
  `parseIntBounded`; fora de `[0, Int.MAX]` vira 0, que os chamadores já
  tratam como "ausente".
* **Inteiros de contagem/TTL passam por `parseIntBounded`.** `parseIntSafe`
  acumula em 32 bits: medido, `--tail 4294967396` virava **100** (o usuário
  pedia bilhões e recebia cem linhas, sem erro) e um `ttlHours` gigante virava
  100 horas. Agora `--tail`, TTL (CLI e API) e os contadores lidos de JSON
  (`pending`/`count`) usam o parser em `Long` com teto de `Int`; valor fora da
  faixa é erro, não wrap.
* **Portas passam por `parsePortSafe`.** `parseIntSafe` fazia aritmética de
  `Int` (32 bits): `--port 4294968080` virava 784 e era aceito em silêncio
  (publicava na porta errada) e `4294967296` virava 0. Agora publish (CLI,
  manifesto e API) e as três portas do `serve` lêem em `Long` com teto de
  65535; flag inválida falha em vez de cair no default.
* **Certificados são externos.** O operador fornece o PEM. O
  `domain verify --tls-cert` confere localmente só o que dá para julgar sem
  confiar na cadeia: SAN/CN cobrindo o hostname (curinga de um rótulo) e
  expiração em 30 dias. **Não** valida assinatura, cadeia, revogação nem se
  o PEM é o que o Edge vai realmente servir — o `openssl x509` lê o arquivo,
  não a configuração de runtime. Também depende do binário `openssl` no
  PATH: ausente, o veredito é "aviso" (nunca recusa), pela mesma regra do
  `doctor`/`scan`.
* **HTTPS no control plane cifra, mas o cliente não autentica o servidor.**
  `kof.http` é trust-all por construção (`JvmWebHttpRuntime.java:112-132`:
  `X509TrustManager` que aceita qualquer certificado +
  `endpointIdentificationAlgorithm` desligado, sem opt-in). O login e o
  `kfc_` viajam cifrados, mas um MITM com certificado forjado é aceito.
  Mitigação: control em rede confiável/loopback ou terminador TLS verificado.
  Não é limite do KofCloud — é a superfície atual do cliente do Kof, que tem
  fatia própria ("cliente verificador") no plano de TLS dele.
* **TLS no Edge público não existe** (L2: `net` sem TLS; L16: handler
  `web.app()` devolve `String` e corrompe binário). O Edge faz HTTP com
  terminador externo. Ver `12-tls.md`.
* **O health check lê só a linha de status da resposta.** O `GET` do Agent
  não segue o corpo nem revalida a cada requisição do Edge: um serviço que
  responde `200` e depois falha é detectado na próxima checagem (ciclo
  `kofcloud.limits.healthMs`, 5 s), não na hora. O código precisa ser **2xx**
  (`healthStatusFor`): 3xx/4xx contam como `DEGRADED` (a sonda não segue
  redirecionamento nem trata 404 como saudável). O prazo é por checagem e
  mede até o cabeçalho chegar — um serviço que manda cabeçalho rápido e
  trava no corpo conta como `ONLINE`.
* **O Edge fala HTTP/1.1.** Sem pipelining avançado, sem trailers.
* **O check local do deploy só sonda `http`.** Para outros protocolos
  (`checkLocalServiceFor` recebe o do manifesto) a verificação continua sendo
  um connect TCP, porque não há como fazer uma sonda genérica. E a sonda lê
  apenas a linha de status: um serviço que responde `200` no `/` e falha nas
  rotas reais passa — o `--check` confirma que a aplicação está de pé, não que
  cada rota funciona.
* **Nem todo inteiro lido da rede usa o parser seguro.** Portas (publish,
  manifesto, `serve`, `parseEdgeAddress`, `edgePort` do `/v1/connect`, porta
  da URL de webhook) e contagens/TTL usam `parsePortSafe`/`parseIntBounded`,
  que acumulam em `Long` e recusam o excesso. O que usa `parseIntSafe` faz
  wrap em 2^32 (L18) — hoje só valores onde o wrap não tem consequência
  (contadores de protocolo, duração já limitada, status HTTP de 3 dígitos).
* **A fila de entrada é limitada, a de saída não.** `queueDepth` (64) vale
  para os itens que chegam a um stream e ainda não foram consumidos; a
  `outbox` compartilhada do túnel continua sem teto — quem enche é o próprio
  produtor local, e o escritor drena na velocidade do socket. Um serviço que
  produza mais rápido que a rede ainda ocupa memória no Agent; o teto de
  resposta (`maxResponseBytes`) e o de fila por stream limitam o caso comum.
* **Sem half-close no protocolo.** Um stream termina com `END`/`RESET`; não há
  sinal de "não vou mais escrever, mas ainda leio".
* **Percentil de latência é o teto da faixa, não o valor.** O histograma do
  Edge guarda contagens por faixa (`le_50`…`gt_2000`), não amostras — então
  `metrics --percentiles` responde `<=250 ms` ou `sem amostras`, nunca um
  número exato (inventá-lo seria mentir com duas casas decimais). A
  distribuição é agregada: percentil por hostname fica fora por desenho
  (cardinalidade por cliente).
* **Limites aplicados.** Dispositivos ativos por conta, serviços por
  dispositivo, streams por túnel, rate limit por hostname, frame DATA fatiado
  (blocos de 16 KiB), teto de linha do túnel (fail-closed), idle timeout,
  conexões simultâneas por serviço, teto de corpo de requisição
  (`httpBodyBytes`, 413) e validação de entrada (nome/porta/exposure/protocol/
  address loopback) — todos com enforcement. Ver `03-device.md` e
  `08-multiplexing.md`. **Rate limit é fixed-window por processo**: não há
  consenso entre nós nem janela deslizante; a janela é config em ms e chega ao
  `security.rateLimit` em segundos por `Limits.httpWindowSeconds` (arredonda
  para cima, mínimo 1 s).
* **WAF (Fase 10) — slices 1–3 feitos; o que não existe ainda.** Prontos: teto
  de corpo (slice 2a), timeout de upstream (slice 2b), detecção/enforce de
  assinaturas em método+path, corpo até 64 KiB e **valores de header** (slice
  1) e allowlist de métodos/paths por serviço (`403 KCF111`, fail-closed só
  para quem declarou a lista — slice 3). **Percent/form-encoding** é
  decodificado em **uma** passada antes do casamento (`wafDecodedContains`,
  sem reconstruir a string), então `%2e%2e%2f`, `union+select` e `%3Cscript`
  casam; dupla codificação (`%252e`) não é desfeita em cadeia, e base64
  dentro de JSON/URL-encode aninhado continua fora. **Barra invertida casa
  nos três lanes:** `..\` cru e a forma JSON-escapada `..\\` (que contém a
  crua como prefixo) casam em path, corpo e header — sem normalizar a
  string (concatenar em Kof é O(n²)); o pedido repassado segue byte a byte.
  O corpo é convertido para String **só até o teto** de 64 KiB antes da
  varredura (`wafBodyPrefix`): bytes além dele nunca entram na inspeção.
  **Falta** um WAF mais
  profundo: regras por conta e corpo estruturalmente decodificado (JSON
  base64).
  Headers da plataforma ficam fora da varredura por desenho
  (`wafHeaderIgnored`): `Authorization` é credencial (nunca é lida nem ecoada
  pelo detector), `Host` é rota e `Content-Length`/`Connection` são do
  protocolo — só valores de headers do cliente (User-Agent, Referer, Cookie,
  X-*) são inspecionados, até o mesmo teto de 64 KiB.
  O timeout usa polling (`limitIdleSweepMs`) e o prazo é medido
  desde o último DATA — a granularidade é a do passo de varredura, não exata.
  Há **dois** tetos distintos. O de **política** (`httpBodyBytes`, 1 MiB,
  413 KCF110) usa o `Content-Length` **declarado** e vale para carga no
  serviço do usuário. O de **leitura** (`maxBytes` do `serve`, 16 MiB por
  padrão) é o teto absoluto do pedido inteiro (cabeçalho + corpo): a leitura
  recusa sem drenar o corpo quando o declarado já estoura e também quando os
  bytes **reais** passam do teto — um `Content-Length` subdeclarado não
  escapa. A memória do Edge fica limitada a cerca de `maxBytes` por conexão
  (mais o bloco de `receive`, mais a folga do acumulador: no pior caso menos
  de 2× o lido), não ao declarado. O custo de CPU da acumulação deixou de ser
  quadrático: a leitura anexa num buffer com folga (`bytesGrown`) em vez de
  `acc = bytesConcat(acc, chunk)`, que copiava tudo a cada bloco (medido: 1 MiB
  em blocos de 8 KiB custava ~4,8 s, hoje ~16 ms). O corpo só é varrido pelo
  WAF até o teto de 64 KiB — e agora só esse prefixo é convertido para String
  (antes o corpo inteiro era convertido para depois descartar: ~22 s num
  upload de 512 KiB, hoje ~0,7 s).
* **Logs em três arquivos, por desenho (débito quitado).** `kof cloud logs`
  lê `~/.kofcloud/agent.log` (marcos de sessão gravados pela CLI: `connect`,
  `disconnect`). `kof cloud logs --runtime` lê
  `~/.kofcloud/agent-runtime.log`, onde o Agent grava os eventos reais
  (connect, reconnect, recusa fatal, mudança de health, GOAWAY, parada,
  derrubada local por ociosidade, reset de streams presos) pelo mesmo ponto
  que emite para o stdout (`agent/Events.kf`). `kof cloud logs --edge` lê
  `~/.kofcloud/edge-runtime.log`, com os eventos do lado da plataforma
  (recusa de túnel com o motivo do protocolo — inclusive o fail-closed do
  HELLO gigante; túnel aceito; queda por ociosidade; sessão fechada por linha
  gigante no meio da sessão; túnel perdido — EOF do agent, abre incidente;
  teto de conexões públicas; parada limpa; e os desfechos upstream do caminho
  público — teto de resposta, tempo esgotado, teto de streams, rota pendente,
  falha de túnel, resposta vazia, resposta não-HTTP) pelo mesmo padrão
  (`edge/Events.kf`). Cada arquivo tem teto próprio (256 KiB, corta o
  início). Nenhum dos três recebe conteúdo de requisição, credencial, chave
  ou hostname.
* **Os contadores de conexão são aproximados sob corrida** (check-then-act
  sem lock, L3). Vale para as conexões públicas simultâneas (1024) e para as
  conexões por serviço (64): o pior caso estoura por poucas unidades, nunca
  recusa a menos nem trava.
* **Contador por serviço é aproximado sob corrida** (check-then-act sem lock;
  o Kof não expõe mutex, L3): pode estourar em poucas unidades, nunca recusa
  a menos. Contadores internos usam `Map<String, String>` porque
  `Map<String, Int>` como campo de classe quebra o verificador JVM
  (VerifyError medido; sem issue aberta no Kof ainda).
* **A expiração de `kfc_` e `kfs_` é imediata nas validações e periódica no
  `serve`.** Credencial vencida é podada na hora quando alguma validação
  passa por ela (`accountByToken` / `serviceByServiceToken`) E pelo laço
  próprio do `serve` (`credentialExpirySweeper` → `sweepExpiredCredentials`,
  `kofcloud.limits.expirySweepMs`, default 60 s — o mesmo núcleo de poda,
  fonte única: `sweepServiceTokensExpired`), então conta ociosa não acumula
  vencidas sem sinal. O laço é por processo (morre com o `serve`, sem
  coordenação entre nós) e é mais um escritor concorrente do estado — mesma
  aproximação do L3 (check-then-act sem lock, save com a mesma granularidade
  dos demais caminhos). `logout`
  agora tenta auto-revogar a credencial da sessão no servidor antes de apagar
  o local (falha de rede/control fora do ar vira aviso explícito e a
  credencial sobrevive — o operador revoga com `kof cloud token revoke`).
  Credencial já vencida/revogada no `logout` conta como `dead` (nada vivo,
  não é falha). Cada `login` também
  emite credencial nova sem revogar a anterior: `kof cloud token prune` remove
  as antigas, mas a poda é manual — não há expiração automática por padrão.
  O instante de expiração é calculado em `Long` (`tokenExpiryAt`, fonte única
  de kfc_/kfs_): `Int` no Kof é 32 bits, então `ttlHours * 3600000` estourava
  acima de 596 h (~24,8 dias) e um `--ttl 720` (30 dias) nascia vencido — 401
  imediato. `ttlHours` continua `Int` (até ~2,1 bilhões de horas); a
  multiplicação é a única aritmética larga.
* **As marcas de `freeze` são persistidas e agora têm teto higiênico
  (`kofcloud.limits.freezeMarks`, default 1000).** Reiniciar o control plane
  não perde a reversibilidade: o `unfreeze` restaura o status anterior
  gravado (estado antigo sem a chave carrega sem marcas — o `unfreeze` nele
  segue respondendo `409 KCF141`, fail-closed). Remover um alvo congelado
  (serviço removido ou device revogado) poda a marca — ids nunca se repetem,
  então a marca de um id que saiu não serve para nada; a poda persiste no
  arquivo (o `unfreeze` de removido segue 404, o de revogado recusa com
  `409 KCF023` — REVOKED é terminal nos dois sentidos do ciclo, e a marca
  legada de um revogado fica inerte no arquivo). Resíduo de estados
  anteriores às recusas (serviço vivo com device REVOKED e `kfs_` apontando
  para o par) morre na carga: `pruneServiceTokens` roda no `load` — antes a
  credencial autenticava para sempre um serviço cuja rota nunca volta. O
  teto (fonte única `limitFreezeMarks` em `util/Limits.kf`, recuo seguro
  como os demais) **nunca recusa containment**: acima dele, o freeze
  seguinte poda só as marcas inertes (`freezeMarkIsInert` — alvo removido
  ou device REVOKED, cujo unfreeze já responde 404/409 sem ler a marca); se
  todas forem vivas, o teto cede — congelar NUNCA falha por limite de
  marcas, e apagar marca viva perderia o status anterior do unfreeze.
* **O incidente de expiração em massa enxerga conta ociosa.** O detector
  (`mass_token_expiry`) é alimentado pelos DOIS caminhos da mesma fonte —
  a validação (`serviceByServiceToken`, a cada request AUTHENTICATED) e o
  laço periódico do `serve` — então numa conta sem tráfego o sinal chega
  mesmo assim (3 vencidas na janela abrem o incidente da conta dona, sem
  token/serviço/handle no resumo). Os incidentes são efêmeros no processo:
  reiniciar o control plane zera as sondas (o incidente já aberto fica no
  arquivo de estado).
* **A resposta do serviço local tem teto (`kofcloud.maxResponseBytes`, 16 MiB).**
  Acima dele o Edge responde `502 KCF115` e cancela o stream — antes o
  excedente era **cortado em silêncio** e o cliente recebia 200 com corpo
  incompleto. O Agent agora envia em fluxo e para de produzir quando o Edge
  cancela, mas o teto continua existindo: respostas maiores exigem aumentar
  a config (a codificação linear do próximo item já não é o gargalo, mas a
  reconstrução `bytesConcat` O(n²) ainda é).
* **Codificação base64 agora é linear em bytes (caminho quente resolvido).**
  `bytesToBase64`/`bytesFromBase64` deixaram de passar por `bytesToString`
  (concatenação `out = out + c` O(n²)) e pelo `encoding.base64Encode` do
  stdlib (que empacotava byte ≥ 0x80 como UTF-8 e inflava a saída). A nova
  tabela RFC 4648 mapeia byte-a-byte com concatenação em blocos de ~3 KiB via
  `String.concat` (o `java.lang.String.concat`, O(tamanho atual) por chamada).
  Medido: 100 KiB em **~46 ms** (antes ~1137 ms, ~25×) e 512 KiB em ~331 ms;
  a saída é byte-exata (9000 bytes → 12000 chars, sem expansão). Por isso o
  bloco de envio (16 KiB) hoje existe por latência/memória, não para escapar
  de O(n²). `bytesToString` também passou a concatenar em blocos de ~3 KiB
  (mesma técnica; o bloco vazio do fim não é concatenado — um `concat("")`
  devolvia o buffer com folga e dobrava o resultado em múltiplos exatos do
  bloco), então a conversão bytes→String é linear; o que **permanece**
  O(n²) é `bytesConcat`, e ele só resta em trechos pequenos (resposta de
  health, reconstrução de resposta no Edge e acumulação do Agent). O teto de
  `maxResponseBytes` continua; respostas de dezenas de MiB funcionam — o
  caminho de acumulação do Edge é o que ainda paga cópia repetida nesse
  tamanho.

* **O platô do incidente segura reabertura por até 1 h (por sinal e conta).**
  Depois de resolvido, o mesmo sinal que volta dentro de
  `kofcloud.limits.incidentPlateauWindowMs` (default 3600000 ms) não abre
  incidente novo: a reincidência so é contada no embasamento persistido no
  estado (`plateaus`), e o sinal só reaparece como `<kind>_reopened` CRIT
  depois da janela. É deliberado — um ataque contínuo virava um incidente
  por evento (o `freeze` de conta abria um por serviço, todos no mesmo
  instante) e afogava o anel de 100 e o webhook. O custo honesto: um
  resolvedor apressado que fecha o incidente e vê o sinal voltar 5 min
  depois não recebe webhook novo; a contagem fica no estado, e o histórico
  está no audit. Teto higiênico (`limitIncidentRing`) poda embasamentos
  antigos — podar nunca recusa um incidente, só devolve o sinal ao estado
  "primeira vez" (WARN) se ele voltar.

## O que o KofCloud deliberadamente NÃO é

Não é VPS, VM, plataforma de containers, Kubernetes, PaaS, object storage
obrigatório, CDN, banco gerenciado, nem um servidor dedicado por usuário.
Não copia AWS/GCP/Azure. O computador e os dados continuam sendo do usuário.

## Regra fundamental

Não construir uma cloud. Construir uma ponte.
