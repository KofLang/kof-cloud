# Control Protocol (Agent ↔ Control Plane)

Transporte: HTTP/1.1. Corpo: JSON. Autenticação: header **`X-KofCloud-Token:
<token>`** (comparado com `security.constantTimeEquals`).

> Por que não `Authorization: Bearer`: o runtime do Kof (`web.app()`) reserva
> `Authorization` para Bearer JWT e devolve **401 antes do handler** para
> qualquer Authorization que não seja um JWT válido (medido em
> `JvmRuntimeWebDispatch`). Como os tokens do KofCloud são opacos, o control
> plane usa um header próprio. O Edge público, que tem parser HTTP próprio,
> aceita `Authorization: Bearer` normalmente para serviços AUTHENTICATED.
> Ver `13-kof-apis.md` L8.

Base: `http://<control-host>/v1`

## Endpoints

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| `POST` | `/v1/accounts` | — | cria conta `{handle,email}` → `{accountId,handle,token}` |
| `POST` | `/v1/login` | — | `{handle}` → `{accountId,token}` |
| `POST` | `/v1/devices` | conta | registra dispositivo `{name,publicKey}` → `{deviceId}` |
| `GET` | `/v1/devices` | conta | lista dispositivos |
| `POST` | `/v1/devices/revoke` | conta | `{deviceId}` → `{revoked:true}` |
| `POST` | `/v1/services` | conta | publica `{deviceId,name,localAddress,localPort,protocol,exposure}` → `{service}` |
| `GET` | `/v1/services` | conta | lista serviços |
| `POST` | `/v1/services/remove` | conta | `{serviceId}` |
| `POST` | `/v1/services/status` | dispositivo | `{serviceId,status}` (health reportado pelo Agent) |
| `POST` | `/v1/heartbeat` | dispositivo | `{deviceId}` → `{ok:true, serverTime}` |
| `POST` | `/v1/connect` | dispositivo | `{deviceId}` → `{edgeHost,edgePort,tunnelToken}` |
| `GET` | `/v1/domains` | conta | domínios/hostnames da conta |
| `GET` | `/v1/health` | — | liveness do control plane |

## Exemplos

Criar conta:

```
POST /v1/accounts
{"handle":"mel","email":"mel@example.com"}

201 {"accountId":"...","handle":"mel","token":"kfc_..."}
```

Publicar serviço:

```
POST /v1/services
X-KofCloud-Token: kfc_...
{"deviceId":"...","name":"site","localAddress":"127.0.0.1",
 "localPort":8080,"protocol":"http","exposure":"PUBLIC"}

201 {"id":"...","hostname":"site.mel.kofcloud.io","status":"UNKNOWN", ...}
```

`connect` (o Agent pede o ponto de entrada do Edge):

```
POST /v1/connect
X-KofCloud-Token: kfc_...
{"deviceId":"..."}

200 {"edgeHost":"127.0.0.1","edgePort":7000,"tunnelToken":"kft_...",
     "heartbeatMs":15000,"services":[ ... ]}
```

## Erros

| HTTP | Corpo | Quando |
|---|---|---|
| `400` | `{"error":"..."}` | payload inválido |
| `401` | `{"error":"unauthorized"}` | token ausente/inválido |
| `403` | `{"error":"forbidden"}` | recurso de outra conta |
| `404` | `{"error":"not_found"}` | id inexistente |
| `409` | `{"error":"conflict"}` | handle já existe |
| `429` | `{"error":"rate_limited"}` | rate limit |
