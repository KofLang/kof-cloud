# Modelo de serviço e exposição

## Serviço

```
Service
├── id             uuid v4
├── accountId      dono
├── deviceId       onde roda
├── name           rótulo + primeiro rótulo do hostname
├── localAddress   127.0.0.1 (default)
├── localPort      8080
├── protocol       http (MVP)
├── hostname       <name>.<handle>.kofcloud.io
├── exposure       PUBLIC | AUTHENTICATED | PRIVATE
├── status         ONLINE | OFFLINE | DEGRADED | UNKNOWN | SUSPENDED
└── createdAt
```

`SUSPENDED` é administrativo (abuse prevention — `kof cloud service
suspend`): sai da rota como `PRIVATE` (opaco), sem remover; health do Agent
e status via API **não** dessuspendem, só `kof cloud service unsuspend`
libera. Ver `16-plan.md` (abuse slice 2) e `abuse_test.kf`.

Publicar:

```
kof cloud publish site --port 8080
  → hostname: https://site.mel.kofcloud.io
```

## Níveis de exposição

| Nível | Rota no Edge | Quem acessa |
|---|---|---|
| `PUBLIC` | sim | qualquer pessoa |
| `AUTHENTICATED` | sim | somente após autenticação |
| `PRIVATE` | **não** | ninguém pela internet |

`PRIVATE` é o default para bancos de dados e SSH. **Publicação é sempre
explícita**: o Agent nunca publica uma porta só porque ela está aberta
localmente. `kof cloud publish` é a única forma de criar rota.

### `AUTHENTICATED`

O Edge exige uma **credencial de serviço** (`kfs_`) antes de encaminhar:

```
kof cloud service token secret                 # emite (autenticado pela conta)
kof cloud service token secret --ttl 24        # emite com expiracao de 1 dia
curl -H 'Authorization: Bearer kfs_…' https://secret.mel.kofcloud.io/
```

Sem credencial, ou com credencial de outro serviço, devolve `401` com
`WWW-Authenticate: Bearer` — e **não** vaza o conteúdo. O token da conta
(`kfc_`) **não** autoriza acesso público: as credenciais são separadas.

**Ciclo de vida da credencial.** O valor `kfs_` em claro existe uma única
vez: no instante da emissão. Depois, só o hash é guardado — não há como
recuperar, só emitir outra. Operação do dia a dia:

```
kof cloud service token-list secret         # inventario: id curto + emissao (+ expiracao)
kof cloud service token-revoke secret <kfs_…>   # revoga UMA (vazou, rotacao)
```

Detalhes honestos: sem `--ttl`, a credencial vale para sempre; com
`--ttl <horas>`, expirado vira `401` (como revogado) e some do inventário
na próxima validação (poda preguiçosa, contada em
`kofcloud_control_service_tokens_expired_total`). O inventário nunca
devolve o valor — só id curto (8 hex do hash) + emissão (+ expiração).
Revogar credencial inexistente/alheia devolve o mesmo `404` opaco de
recurso inexistente (sem oráculo entre contas).

A autenticação acontece **no Edge**, antes de o tráfego tocar o túnel, e o Edge
**consome** a credencial: ela é removida da requisição repassada ao serviço
local (assim a plataforma não vaza o próprio segredo para a aplicação). Em
serviços `PUBLIC` o `Authorization` é repassado como proxy transparente —
atenção: um serviço local feito com `web.app()` recusa qualquer `Authorization`
antes do handler (lacuna L8 do Kof), então apps Kof não podem usar esse header
para autenticação própria hoje.

## Health check

O **Agent** verifica localmente cada serviço publicado (uma conexão TCP a
`localAddress:localPort`; para `http`, um `GET /`). Resultado:

| Estado | Condição |
|---|---|
| `ONLINE` | conecta e responde |
| `OFFLINE` | recusa conexão / timeout |
| `DEGRADED` | conecta mas responde ≥ 500 ou timeout parcial |
| `UNKNOWN` | ainda não verificado |

O estado vai ao Edge no frame `STATUS`. Quando um serviço fica `OFFLINE`, o Edge
**para de encaminhar** para ele e devolve `503` — sem derrubar os outros
serviços do mesmo túnel.

## Roteamento

O Edge resolve o serviço pelo header `Host`:

```
Host: api.mel.kofcloud.io
   → account "mel" + service "api"  → deviceId → sessão de túnel → stream
```

Sem correspondência → `404`. Serviço `PRIVATE` → `404` (não revela existência).
Serviço `SUSPENDED` → `404` (indistinguível — suspensão não revela existência).
Serviço `OFFLINE` → `503`.

## Limites por dispositivo (Fase 4)

Um mesmo Agent pode servir vários serviços: cada hostname roteia para a sua
porta local, no mesmo túnel. O control plane aplica, na publicação,
`kofcloud.limits.services` por dispositivo (default 10) e
`kofcloud.limits.devices` dispositivos ativos por conta (default 5); ao
exceder, responde `429` com `{"error":"KCF081: ..."}` /
`{"error":"KCF080: ..."}` (códigos estáveis R6 — ver `06-control-protocol.md`). Derrubar um serviço local não afeta os outros
(provado em `tests/e2e_multiservice_test.kf`).
