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
├── status         ONLINE | OFFLINE | DEGRADED | UNKNOWN
└── createdAt
```

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

O Edge exige um token de sessão antes de encaminhar. Sem token válido devolve
`401` com `WWW-Authenticate`. A autenticação acontece **no Edge**, antes de o
tráfego tocar o túnel — a máquina do usuário nunca vê requisição não
autenticada.

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
Serviço `OFFLINE` → `503`.
