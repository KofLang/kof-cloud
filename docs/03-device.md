# Modelo de dispositivo

Um **dispositivo** é uma instalação do Agent numa máquina. É a unidade de
identidade criptográfica e de revogação.

## Ciclo de vida

```
            kof cloud login
                 │
                 ▼
        ┌── UNREGISTERED ──┐        (sem identidade local)
        │                  │  kof cloud connect  → gera par X25519
        ▼                  ▼                     registra device
   ┌─────────┐      ┌──────────────┐
   │ REVOKED │◄─────│  REGISTERED  │
   └─────────┘      └──────────────┘
   device revoke         │  connect
                         ▼
                   ┌───────────┐   queda de rede
                   │ CONNECTED │◄──────────────┐
                   └───────────┘               │
                         │  falha              │
                         ▼                     │
                   ┌──────────────┐            │
                   │ DISCONNECTED │────────────┘
                   └──────────────┘   backoff + reconnect
```

## Estado local (`~/.kofcloud/`)

| Arquivo | Conteúdo | Sensível |
|---|---|---|
| `identity.json` | `deviceId`, `accountId`, `handle`, chave privada X25519 (hex) | **sim** |
| `session.json` | token de conta, expiração | **sim** |
| `services.json` | serviços publicados localmente | não |
| `agent.log` | log operacional | não |

Regras:

* A chave privada **nunca** é enviada ao control plane nem ao Edge.
* `identity.json` e `session.json` são gravados com permissão restrita
  (`0600` onde a plataforma permitir).
* O Agent **não** roda como root/admin por padrão.
* O Agent **não** abre porta de entrada alguma.

## Presença

O dispositivo se declara presente por:

1. o handshake do túnel (`HELLO` → `WELCOME`) — presença forte;
2. `heartbeat` periódico (default 15 s) — mantém `lastSeenAt` fresco.

Sem heartbeat por `3 × interval`, o control plane marca o dispositivo
`OFFLINE` e o Edge derruba a sessão correspondente.

## Revogação

`kof cloud device revoke <id>`:

1. marca `status = REVOKED` no control plane;
2. remove as rotas dos serviços daquele dispositivo;
3. a próxima tentativa de `HELLO` daquele device recebe `WELCOME{ok:false,
   reason:"DEVICE_REVOKED"}` e a conexão é encerrada.

A conta e os demais dispositivos permanecem válidos.

## Limites

| Limite | Default | Config |
|---|---|---|
| dispositivos por conta | 5 | `limits.devices` |
| serviços por dispositivo | 10 | `limits.services` |
| conexões simultâneas por serviço | 64 | `limits.connections` |
| streams simultâneos por túnel | 128 | `limits.streams` |
| tamanho de frame | 1 MiB | `limits.frameBytes` |
| idle timeout do túnel | 60 s | `limits.idleTimeoutMs` |
