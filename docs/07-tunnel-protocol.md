# Tunnel Protocol (Agent ↔ Edge)

Transporte: **TCP persistente iniciado pelo Agent**. Formato: um objeto JSON por
linha (`\n`). Toda carga binária é base64.

```
Agent ──── net.connect(edgeHost, edgePort) ────► Edge
   │                                                │
   │  {"t":"HELLO", ...}\n                          │
   │ ──────────────────────────────────────────────►│
   │  {"t":"WELCOME","ok":true, ...}\n              │
   │ ◄──────────────────────────────────────────────│
   │  {"t":"STATUS", ...}\n                         │   handshake completo
   │ ──────────────────────────────────────────────►│
   │  {"t":"OPEN","s":1,"service":"site"}\n         │   ◄── Edge abre stream
   │ ◄──────────────────────────────────────────────│
   │  {"t":"DATA","s":1,"d":"<base64>"}\n           │   requisição HTTP
   │ ◄──────────────────────────────────────────────│
   │  {"t":"DATA","s":1,"d":"<base64>"}\n           │   resposta HTTP
   │ ──────────────────────────────────────────────►│
   │  {"t":"END","s":1}\n                           │
   │ ──────────────────────────────────────────────►│
```

## Frames

| `t` | Direção | Campos | Significado |
|---|---|---|---|
| `HELLO` | Agent→Edge | `v`, `deviceId`, `token`, `caps[]` | abre a sessão |
| `WELCOME` | Edge→Agent | `v`, `ok`, `reason?`, `sessionId`, `heartbeatMs` | aceita/recusa |
| `PING` | ambos | — | keepalive |
| `PONG` | ambos | — | resposta ao PING |
| `STATUS` | Agent→Edge | `service`, `status` | health reportado |
| `SERVICES` | Edge→Agent | `services[]` | config efetiva (hostnames, exposição) |
| `OPEN` | Edge→Agent | `s`, `service`, `meta?` | abre stream para um serviço |
| `OPENED` | Agent→Edge | `s`, `ok`, `reason?` | aceita/recusa o stream |
| `DATA` | ambos | `s`, `d` (base64) | bytes do stream |
| `END` | ambos | `s` | fim limpo do stream |
| `RESET` | ambos | `s`, `code` | erro no stream |
| `GOAWAY` | Edge→Agent | `code` | encerra a sessão |

`s` = `streamId` (Int, ≥ 1, único por sessão, alocado por quem abre).

## Handshake

1. Agent conecta e envia `HELLO{v, deviceId, token, caps}`.
2. Edge valida: versão → token (`constantTimeEquals`) → dispositivo (`ACTIVE`).
3. Edge responde `WELCOME{ok:true, sessionId, heartbeatMs}` ou
   `WELCOME{ok:false, reason}` + `close()`.
4. Edge envia `SERVICES` com a configuração efetiva (o Agent não adivinha
   hostnames).
5. Sessão ativa: `PING`/`PONG` a cada `heartbeatMs`.

## Handshake por stream (OPEN → OPENED)

O Edge **sempre** espera o `OPENED` antes de enviar `DATA`: sem ele, um OPEN
recusado (`ok:false`, ex. serviço publicado depois do `connect`, antes do
hot-reload) deixaria o relay bloqueado para sempre esperando resposta.
`OPENED{ok:false}` → o Edge fecha o stream e responde `502` imediato, sem
consumir a vaga de conexão (a vaga é liberada junto). O Agent reaprende a
lista via refresher periódico (`/v1/connect`) e o próximo OPEN já passa.

## Cifra do túnel

Quando `KOFCLOUD_TUNNEL_SECRET` está definido nos dois lados:

1. Agent gera par X25519 efêmero; envia a pública no `HELLO.clientPub`.
2. Edge gera par efêmero; responde `WELCOME.serverPub`.
3. Ambos derivam `sessionKey = HKDF-SHA256(X25519(priv, peerPub), salt=sessionId,
   info="kofcloud-tunnel-v1", 32 bytes)`.
4. Todo `d` (payload de `DATA`) passa a ser `AES-GCM(plaintext, sessionKey)` em
   base64; os metadados (tipo, stream) permanecem em claro.

Isso é defesa em profundidade sobre o TLS do Edge: o Edge termina TLS e depois
cifra de novo até o Agent, de modo que um operador com acesso ao processo do
Edge não lê payload em trânsito sem a chave de sessão. **Não** substitui TLS no
caminho público.

> Estado: a derivação de chave e a cifra estão implementadas e testadas
> (`tests/tunnel_crypto_test.kf`). O transporte TLS público do Edge está
> especificado em `11-tls.md` e é JVM-only no Kof atual.
