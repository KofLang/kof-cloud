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
   │ REVOKED │◄─────│  REGISTERED  │──device suspend──┐
   └─────────┘      └──────────────┘                   ▼
   device revoke         │  connect            ┌────────────┐
                         ▼                     │ SUSPENDED  │ (tunel recusado,
                   ┌───────────┐   queda de    │ nada e     │  reversivel com
                   │ CONNECTED │◄──────────────┤ removido)  │  unsuspend)
                   └───────────┘               └────────────┘
                         │  falha
                         ▼
                   ┌──────────────┐
                   │ DISCONNECTED │──backoff + reconnect──┐
                   └──────────────┘                       │ (volta sozinho,
                        ▲                                 │  exceto REVOKED/
                        └────────────────────────────────┘  SUSPENDED=fatal)
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

## Suspensão

`kof cloud device suspend <id>` (reversível com `unsuspend`): marca
`status = SUSPENDED` sem remover nada; o túnel é recusado com
`DEVICE_SUSPENDED` (fatal — sem reconexão) e sessões vivas caem no próximo
frame útil ou request (enforcement em request + PING). Diferente de revogar,
suspender preserva serviços, tokens e domínios — `unsuspend` libera tudo.

## Limites

| Limite | Default | Config | Aplicado? |
|---|---|---|---|
| dispositivos ativos por conta | 5 | `kofcloud.limits.devices` | ✅ (429 `KCF080`) |
| serviços por dispositivo | 10 | `kofcloud.limits.services` | ✅ (429 `KCF081`) |
| streams simultâneos por túnel | 128 | `kofcloud.limits.streams` | ✅ (429 / `RESET LIMIT_EXCEEDED`) |
| conexões simultâneas por serviço | 64 | `kofcloud.limits.serviceConns` | ✅ (429 `KCF108` + `Retry-After`, antes do OPEN) |
| bytes por frame DATA | 1 MiB | `kofcloud.limits.frameBytes` | ✅ (envio fatiado em 16 KiB; teto fail-closed) |
| requisições por hostname | 100 / 60 s | `kofcloud.limits.httpRequests` / `httpWindowMs` | ✅ (429 `KCF107` + `Retry-After`) |
| idle timeout do túnel | 120 s | `kofcloud.limits.tunnelIdleMs` | ✅ (cai sem frames úteis; PING não renova) |
| idle timeout do stream | 30 s | `kofcloud.limits.streamIdleMs` | ✅ (`RESET IDLE_TIMEOUT`, sem matar a sessão) |

Limites são lidos de `kof.config`/env (`KOF_KOFCLOUD_LIMITS_*`). Revogar um
dispositivo libera a vaga; re-registrar o mesmo nome é idempotente e não
consome vaga. O limite de serviços é por **dispositivo**, não por conta.
