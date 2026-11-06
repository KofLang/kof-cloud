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
* **Um nó de control plane, um nó de Edge, em memória + arquivo JSON.** Não há
  consenso distribuído nem banco. É suficiente para provar o fluxo, não para
  produção multi-nó.
* **`AUTHENTICATED` usa token de conta**, não um provedor de identidade
  federado.
* **Rate limiting é por processo.** Não é global entre nós.
* **Certificados são externos.** O operador fornece o PEM.
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
* **O Edge fala HTTP/1.1.** Sem pipelining avançado, sem trailers.
* **Sem half-close no protocolo.** Um stream termina com `END`/`RESET`; não há
  sinal de "não vou mais escrever, mas ainda leio".
* **Limites aplicados.** Dispositivos ativos por conta, serviços por
  dispositivo, streams por túnel, rate limit por hostname, frame DATA fatiado
  (blocos de 16 KiB), teto de linha do túnel (fail-closed), idle timeout,
  conexões simultâneas por serviço, teto de corpo de requisição
  (`httpBodyBytes`, 413) e validação de entrada (nome/porta/exposure/protocol/
  address loopback) — todos com enforcement. Ver `03-device.md` e
  `08-multiplexing.md`.
* **WAF (Fase 10) — slices 1–3 feitos; o que não existe ainda.** Prontos: teto
  de corpo (slice 2a), timeout de upstream (slice 2b), detecção/enforce de
  assinaturas em método+path, corpo até 64 KiB e **valores de header** (slice
  1) e allowlist de métodos/paths por serviço (`403 KCF111`, fail-closed só
  para quem declarou a lista — slice 3). **Falta** um WAF mais profundo: regras
  por conta e corpo decodificado (JSON base64/URL-encode). As assinaturas são
  literais: no corpo e nos headers, a barra invertida do JSON não é
  normalizada, então traversal JSON-escapado não casa ali (casa na query crua).
  Headers da plataforma ficam fora da varredura por desenho
  (`wafHeaderIgnored`): `Authorization` é credencial (nunca é lida nem ecoada
  pelo detector), `Host` é rota e `Content-Length`/`Connection` são do
  protocolo — só valores de headers do cliente (User-Agent, Referer, Cookie,
  X-*) são inspecionados, até o mesmo teto de 64 KiB.
  O timeout usa polling (`limitIdleSweepMs`) e o prazo é medido
  desde o último DATA — a granularidade é a do passo de varredura, não exata.
  O teto usa o
  `Content-Length` **declarado**, e o corpo inteiro já foi lido antes da
  decisão: é proteção de política e de carga no serviço do usuário, não uma
  garantia de memória no Edge. Quem declara `Content-Length` falso continua
  limitado pelo `maxBytes` global do `serve`.
* **Logs em dois arquivos, por desenho (débito quitado).** `kof cloud logs`
  lê `~/.kofcloud/agent.log` (marcos de sessão gravados pela CLI: `connect`,
  `disconnect`). `kof cloud logs --runtime` lê
  `~/.kofcloud/agent-runtime.log`, onde o Agent grava os eventos reais
  (connect, reconnect, recusa fatal, mudança de health, GOAWAY, parada) pelo
  mesmo ponto que emite para o stdout (`agent/Events.kf`). Cada arquivo tem
  teto próprio (256 KiB, corta o início). Nenhum dos dois recebe conteúdo de
  requisição, credencial ou chave.
* **Contador por serviço é aproximado sob corrida** (check-then-act sem lock;
  o Kof não expõe mutex, L3): pode estourar em poucas unidades, nunca recusa
  a menos. Contadores internos usam `Map<String, String>` porque
  `Map<String, Int>` como campo de classe quebra o verificador JVM
  (VerifyError medido; sem issue aberta no Kof ainda).
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
