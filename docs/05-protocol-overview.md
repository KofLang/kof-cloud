# Protocolo — visão geral

Dois protocolos independentes, com transportes distintos:

| Protocolo | Entre | Transporte | Formato |
|---|---|---|---|
| **Control Protocol** | Agent ↔ Control Plane | HTTP/1.1 | JSON |
| **Tunnel Protocol** | Agent ↔ Edge | TCP persistente | frames JSON por linha |

Separar os dois é deliberado (§26 do pedido): o túnel não é acoplado ao HTTP, e
o control plane pode evoluir (gRPC, WebSocket, fila) sem tocar no data plane.

## Versionamento

Todo handshake declara `v` (inteiro). Regra:

* mesmo major → compatível;
* major maior recebido → `WELCOME{ok:false, reason:"VERSION_UNSUPPORTED"}`.

Versão atual: `PROTOCOL_VERSION = 1`.

## Enquadramento (framing)

O túnel é um fluxo de bytes sem fronteiras (`Conn.receive` devolve "o que
chegou"). O enquadramento é **linha por linha**:

```
<json-frame>\n
```

* `\n` (0x0A) é o único delimitador; nunca aparece dentro do JSON (todo binário
  vai codificado em base64).
* Um frame é um objeto JSON com as chaves do §`06-tunnel-protocol.md`.
* O leitor acumula bytes até `\n`, decodifica **uma** linha por vez, e mantém o
  resto no buffer.

Essa escolha evita depender de um codec binário que o Kof ainda não oferece, e
é robusta às fronteiras arbitrárias de `receive`.

```
recebido: {"t":"DATA","s":3,"d":"aGk="}\n{"t":"DAT
          └────── frame 1 ──────────┘ └─ buffer parcial ─┐
                                                         │
                                  acumula até o próximo \n
```

## Identificação e capacidades

O `HELLO` identifica o peer (deviceId + token) e declara `caps` — uma lista de
capacidades. Hoje: `http`. Capacidade desconhecida é ignorada (forward-compat);
capacidade exigida e ausente → erro explícito.

## Erros e encerramento

* Erro de protocolo em um stream → frame `RESET{streamId, code}`; o stream
  fecha, o túnel continua.
* Erro de protocolo fatal (handshake inválido) → `WELCOME{ok:false}` ou
  `GOAWAY{code}` seguido de `close()`.
* Fechamento limpo: `close()` do socket. Não há half-close explícito no
  protocolo; um stream termina com `END{streamId}`.

## Códigos

| Código | Significado |
|---|---|
| `AUTH_FAILED` | token ausente/inválido |
| `DEVICE_REVOKED` | dispositivo revogado |
| `DEVICE_UNKNOWN` | deviceId não registrado |
| `VERSION_UNSUPPORTED` | major incompatível |
| `SERVICE_UNKNOWN` | serviço não existe / `PRIVATE` |
| `SERVICE_OFFLINE` | serviço sem health |
| `LIMIT_EXCEEDED` | limite de streams/conexões |
| `BAD_FRAME` | JSON inválido ou tipo desconhecido |
| `TIMEOUT` | idle/read timeout |
