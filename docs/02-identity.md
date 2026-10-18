# Modelo de identidade

Três identidades distintas, com ciclos de vida independentes:

```
Account          (pessoa / organização)      ── login, e-mail
   │
   ├── Device    (uma instalação do Agent)   ── chave pública própria
   │      │
   │      └── Service (um serviço publicado) ── nome, porta, exposição
   │
   └── Token     (credencial de acesso)      ── emitido pelo control plane
```

## Account

| Campo | Tipo | Notas |
|---|---|---|
| `id` | String (uuid v4) | imutável |
| `handle` | String | vira o subdomínio: `mel` → `mel.kofcloud.io` |
| `email` | String | login |
| `createdAt` | Long (epoch ms) | |

A conta é o dono do *namespace* de subdomínios. O `handle` é validado
(`[a-z0-9-]{3,32}`, sem hífen nas pontas) porque vira rótulo DNS.

## Device

| Campo | Tipo | Notas |
|---|---|---|
| `id` | String (uuid v4) | imutável |
| `accountId` | String | dono |
| `name` | String | rótulo humano (`notebook-mel`) |
| `publicKey` | String (hex) | **identidade criptográfica do dispositivo** |
| `status` | `ACTIVE` / `REVOKED` | revogação por dispositivo |
| `lastSeenAt` | Long | atualizado no heartbeat |
| `createdAt` | Long | |

### Identidade criptográfica do dispositivo

Cada instalação gera um par **X25519** local (`keyExchange.privateKey()`); a
chave privada **nunca sai da máquina** (persistida em `~/.kofcloud/identity.json`
com permissão restrita). A chave pública é registrada no control plane no
momento do `device register`.

* Revogar um dispositivo (`device revoke <id>`) marca `status = REVOKED` e
  **não** invalida a conta nem os demais dispositivos.
* Um dispositivo revogado é recusado no handshake do túnel (`WELCOME` de erro
  `DEVICE_REVOKED`).

> **Nota honesta.** O MVP usa a chave X25519 para o *handshake de sessão*
> (ECDH + HKDF) e o token de conta para *autorização*. A assinatura de
> desafio com Ed25519 (`crypto.sign`) existe no Kof mas é JVM-only
> (`SECN013` fora do JVM); o fluxo de prova-de-posse por assinatura está
> especificado em `08-flows.md` e implementado onde o target suporta.

## Service

| Campo | Tipo | Notas |
|---|---|---|
| `id` | String (uuid v4) | imutável |
| `accountId` | String | dono |
| `deviceId` | String | onde roda |
| `name` | String | `site`, `api`, `files` |
| `localAddress` | String | default `127.0.0.1` |
| `localPort` | Int | porta local |
| `protocol` | String | MVP: `http` |
| `hostname` | String | `<name>.<handle>.kofcloud.io` |
| `exposure` | `PUBLIC`/`AUTHENTICATED`/`PRIVATE` | |
| `status` | `ONLINE`/`OFFLINE`/`DEGRADED`/`UNKNOWN` | health |
| `createdAt` | Long | |

## Token

| Campo | Tipo | Notas |
|---|---|---|
| `value` | String | opaco; `kfc_` + 32 bytes aleatórios em hex |
| `accountId` | String | |
| `deviceId` | String \| `""` | escopo de dispositivo, quando aplicável |
| `scope` | `account` / `device` | |
| `createdAt` | Long | |
| `expiresAt` | Long | 0 = sem expiração |

O token é apresentado no header `Authorization: Bearer <value>` para o control
plane e no frame `HELLO` para o Edge. Comparação **sempre** com
`security.constantTimeEquals` (nunca `==`), para não vazar por tempo.

## Separação de poderes

| Ação | Account | Device | Service |
|---|---|---|---|
| criar/remover serviço | ✅ | — | — |
| conectar túnel | ✅ | ✅ (chave) | — |
| rotar tráfego | — | — | ✅ (só se `PUBLIC`/`AUTHENTICATED`) |
| revogar | ✅ | ✅ (individual) | ✅ (unpublish) |

Um serviço `PRIVATE` **não tem rota no Edge** — não existe caminho público para
ele, independentemente do estado do dispositivo.
