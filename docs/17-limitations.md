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
* Daemon (systemd/launchd/Windows Service) — o core não depende deles.
* Auto-update do Agent com verificação de assinatura.
* Painel web (a CLI vem primeiro).
* Object storage, banco gerenciado, VPS, Kubernetes.

## Limites do MVP implementado

* **Um nó de control plane, um nó de Edge, em memória + arquivo JSON.** Não há
  consenso distribuído nem banco. É suficiente para provar o fluxo, não para
  produção multi-nó.
* **`AUTHENTICATED` usa token de conta**, não um provedor de identidade
  federado.
* **Rate limiting é por processo.** Não é global entre nós.
* **Certificados são externos.** O operador fornece o PEM.
* **O Edge fala HTTP/1.1.** Sem pipelining avançado, sem trailers.
* **Sem half-close no protocolo.** Um stream termina com `END`/`RESET`; não há
  sinal de "não vou mais escrever, mas ainda leio".
* **Limites aplicados.** Dispositivos ativos por conta, serviços por
  dispositivo, streams por túnel, rate limit por hostname, frame DATA fatiado
  (blocos de 16 KiB), teto de linha do túnel (fail-closed), idle timeout e
  conexões simultâneas por serviço — todos configuráveis e com enforcement.
  Ver `03-device.md` e `08-multiplexing.md`.
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
