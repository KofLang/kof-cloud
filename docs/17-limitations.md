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

## Fora do MVP (por decisão)

* HTTP/2 e HTTP/3.
* `Transfer-Encoding` / chunked: **não decodificado** — o Edge recusa o pedido
  com `400 KCF118` em vez de repassar um header que o serviço local esperaria
  indefinidamente. Pedido com corpo precisa de um `Content-Length` único,
  numérico e canônico; é ele que delimita o corpo repassado (bytes além do
  declarado são descartados, nunca viram uma segunda requisição).
* QUIC, UDP forwarding, TCP forwarding genérico.
* CDN, edge distribuído, relay regional.
* Domínio próprio: registro + verificação + rota (Fase 6 slice 1) ✅; sem
  TLS por SNI e sem automação por provedor (Cloudflare/Route53).
* Deploy (`kof cloud deploy`).
* Daemon: **artefato por SO gerado** (systemd/launchd/Windows, Fase 13 s1) e
  ativação sempre manual; o core não depende deles. Tray/estado visível e
  auto-update assinado seguem pendentes (slices 2 e 3).
* Auto-update do Agent com verificação de assinatura. (A rotação de `kfs_`
  por serviço existe — `service rotate` com overlap — mas `rotate --all`,
  rotação programada de PEM e kill-switch global seguem pendentes.)
* Painel web **completo** — só o slice 1 existe (somente leitura via
  `serve --panel`); criar/alterar pelo navegador (slice 2) e onboarding
  guiado (slice 3) seguem pendentes.
* Object storage, banco gerenciado, VPS, Kubernetes.

## Limites do MVP implementado

* **A trilha de auditoria é detectável, não imutável.** Cada evento tem hash
  encadeado (`audit --verify` acusa adulteração/remoção parcial), mas o
  arquivo de estado pode ser reescrito inteiro por quem tem o disco — WORM
  real (append-only em storage separado) segue pendente.
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
* **Rate limiting é por processo.** Não é global entre nós.
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
  `kofcloud.limits.healthMs`, 5 s), não na hora. O prazo é por checagem e
  mede até o cabeçalho chegar — um serviço que manda cabeçalho rápido e
  trava no corpo conta como `ONLINE`.
* **O Edge fala HTTP/1.1.** Sem pipelining avançado, sem trailers.
* **A fila de entrada é limitada, a de saída não.** `queueDepth` (64) vale
  para os itens que chegam a um stream e ainda não foram consumidos; a
  `outbox` compartilhada do túnel continua sem teto — quem enche é o próprio
  produtor local, e o escritor drena na velocidade do socket. Um serviço que
  produza mais rápido que a rede ainda ocupa memória no Agent; o teto de
  resposta (`maxResponseBytes`) e o de fila por stream limitam o caso comum.
* **Sem half-close no protocolo.** Um stream termina com `END`/`RESET`; não há
  sinal de "não vou mais escrever, mas ainda leio".
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
  dentro de JSON/URL-encode aninhado continua fora. **Falta** um WAF mais
  profundo: regras por conta e corpo estruturalmente decodificado (JSON
  base64). A barra invertida do JSON não é normalizada: traversal
  JSON-escapado não casa no corpo nem no header (casa na query crua).
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
  (mais o bloco de `receive`), não ao declarado; o que continua fora da
  garantia é o custo de CPU do `bytesConcat` O(n²) nesse caminho.
* **Logs em dois arquivos, por desenho (débito quitado).** `kof cloud logs`
  lê `~/.kofcloud/agent.log` (marcos de sessão gravados pela CLI: `connect`,
  `disconnect`). `kof cloud logs --runtime` lê
  `~/.kofcloud/agent-runtime.log`, onde o Agent grava os eventos reais
  (connect, reconnect, recusa fatal, mudança de health, GOAWAY, parada) pelo
  mesmo ponto que emite para o stdout (`agent/Events.kf`). Cada arquivo tem
  teto próprio (256 KiB, corta o início). Nenhum dos dois recebe conteúdo de
  requisição, credencial ou chave.
* **Os contadores de conexão são aproximados sob corrida** (check-then-act
  sem lock, L3). Vale para as conexões públicas simultâneas (1024) e para as
  conexões por serviço (64): o pior caso estoura por poucas unidades, nunca
  recusa a menos nem trava.
* **Contador por serviço é aproximado sob corrida** (check-then-act sem lock;
  o Kof não expõe mutex, L3): pode estourar em poucas unidades, nunca recusa
  a menos. Contadores internos usam `Map<String, String>` porque
  `Map<String, Int>` como campo de classe quebra o verificador JVM
  (VerifyError medido; sem issue aberta no Kof ainda).
* **A expiração de `kfc_` e `kfs_` é preguiçosa.** Credencial vencida só é
  podada quando alguma validação passa por ela (`accountByToken` /
  `serviceByServiceToken`). Numa conta ociosa, credenciais vencidas ficam no
  estado até a próxima validação — não há varredura periódica. `logout`
  continua sendo **só local**: apaga o arquivo da máquina, não invalida no
  servidor (para isso existe `kof cloud token revoke`). Cada `login` também
  emite credencial nova sem revogar a anterior: `kof cloud token prune` remove
  as antigas, mas a poda é manual — não há expiração automática por padrão.
  O instante de expiração é calculado em `Long` (`tokenExpiryAt`, fonte única
  de kfc_/kfs_): `Int` no Kof é 32 bits, então `ttlHours * 3600000` estourava
  acima de 596 h (~24,8 dias) e um `--ttl 720` (30 dias) nascia vencido — 401
  imediato. `ttlHours` continua `Int` (até ~2,1 bilhões de horas); a
  multiplicação é a única aritmética larga.
* **A expiração de `kfs_` é preguiçosa — e o incidente também.** Credencial
  vencida só é podada quando alguma validação de token passa por
  `serviceByServiceToken` (o que acontece a cada request AUTHENTICATED no
  Edge, mas não em conta ociosa). Numa conta sem tráfego, credenciais vencidas
  acumulam no estado sem sinal até a próxima validação; o detector de
  expiração em massa herda a mesma janela (não há varredura periódica no
  control plane). Os incidentes são efêmeros no processo: reiniciar o control
  plane zera as sondas (o incidente já aberto fica no arquivo de estado).
* **A resposta do serviço local tem teto (`kofcloud.maxResponseBytes`, 16 MiB).**
  Acima dele o Edge responde `502 KCF115` e cancela o stream — antes o
  excedente era **cortado em silêncio** e o cliente recebia 200 com corpo
  incompleto. O Agent agora envia em fluxo e para de produzir quando o Edge
  cancela, mas o teto continua existindo: respostas maiores exigem aumentar
  a config (e pagam o custo O(n²) abaixo).
* **Throughput de payloads grandes é limitado por `bytesToString` O(n²).**
  Medido: ~1 s para 100 KiB em `bytesToBase64`; 1 MiB levaria ~100 s. Por isso
  o envio é fatiado em blocos de 16 KiB (custo total linear no tamanho do
  bloco, não no payload) e o teto de linha existe para recusar, não para
  transportar. Respostas de dezenas de MiB funcionam, mas devagar — quando o
  stdlib tiver base64/slice O(n) sobre bytes, o bloco pode subir. Também por
  isso os testes evitam materializar 1.5 MiB em String (`frames_test.kf` lê em
  chunks e confere tamanho + pontas).

## O que o KofCloud deliberadamente NÃO é

Não é VPS, VM, plataforma de containers, Kubernetes, PaaS, object storage
obrigatório, CDN, banco gerenciado, nem um servidor dedicado por usuário.
Não copia AWS/GCP/Azure. O computador e os dados continuam sendo do usuário.

## Regra fundamental

Não construir uma cloud. Construir uma ponte.
