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
| `POST` | `/v1/devices/suspend` | conta | `{deviceId}` → `{suspended:true}` (túnel recusado, nada removido) |
| `POST` | `/v1/devices/unsuspend` | conta | `{deviceId}` → `{suspended:false}` |
| `POST` | `/v1/services` | conta | publica `{deviceId,name,localAddress,localPort,protocol,exposure}` → `{service}` |
| `GET` | `/v1/services` | conta | lista serviços |
| `POST` | `/v1/services/token` | conta | emite credencial `kfs_` do serviço `{serviceId}` (+ `ttlHours` opcional, horas) → `{token,serviceId,ttlHours?}` (TTL inválido = 400 `KCF062`) |
| `POST` | `/v1/services/tokens` | conta | inventário do serviço `{serviceId}` → `[{serviceId,id,createdAt}]` (id = 8 hex do hash; nunca o valor `kfs_`; 404 opaco se alheio/inexistente) |
| `POST` | `/v1/services/token/revoke` | conta | revoga UMA credencial `{serviceId,token}` → `{revoked:true}` (404 opaco se alheio/inexistente) |
| `POST` | `/v1/services/remove` | conta | `{serviceId}` |
| `POST` | `/v1/services/suspend` | conta | `{serviceId}` → `{suspended:true}` (sai da rota, nada removido) |
| `POST` | `/v1/services/unsuspend` | conta | `{serviceId}` → `{suspended:false}` |
| `GET` | `/v1/audit` | conta | últimos eventos da conta (anel de 100, só metadados) |
| `POST` | `/v1/services/status` | dispositivo | `{serviceId,status}` (health reportado pelo Agent) |
| `POST` | `/v1/heartbeat` | dispositivo | `{deviceId}` → `{ok:true, serverTime}` |
| `POST` | `/v1/connect` | dispositivo | `{deviceId}` → `{edgeHost,edgePort,tunnelToken}` |
| `GET` | `/v1/presence` | conta | devices DA CONTA com túnel vivo neste Edge → `["<deviceId>", ...]` (vazio = offline, não erro; 401 sem token; nada de terceiros vaza) |
| `GET` | `/v1/domains` | conta | domínios/hostnames da conta (wildcard + serviços + próprios) |
| `POST` | `/v1/domains` | conta | registra domínio próprio `{hostname,service}` → `{hostname,status,instruction}` (PENDING + CNAME) |
| `POST` | `/v1/domains/verify` | conta | `{hostname}` → `{hostname,status,verifiedAt}` (verifica o DNS e ativa a rota) |
| `POST` | `/v1/domains/remove` | conta | `{hostname}` → `{removed:true}` |
| `GET` | `/v1/health` | — | liveness do control plane |
| `GET` | `/v1/metrics` | — | export Prometheus (`observability.metrics()`): contadores do control plane + Edge (mesmo processo no `serve`); sem auth, só agregados |

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

Emitir credencial de serviço (AUTHENTICATED):

```
POST /v1/services/token
X-KofCloud-Token: kfc_...
{"serviceId":"..."}

200 {"token":"kfs_...","serviceId":"..."}

Com expiracao (`--ttl 24` = 1 dia na CLI; sem flag, sem expiracao):

```
POST /v1/services/token
X-KofCloud-Token: kfc_...
{"serviceId":"...","ttlHours":"24"}

200 {"token":"kfs_...","serviceId":"...","ttlHours":"24"}
```

Expirado nao autoriza (401, como revogado) e some do inventario na
proxima validacao (poda preguicosa). TTL zero/lixo = `400 KCF062`.

Inventario (saber o que revogar sem expor valores — o `kfs_` em claro so
existe no instante da emissao):

```
POST /v1/services/tokens
X-KofCloud-Token: kfc_...
{"serviceId":"..."}

200 [{"serviceId":"...","id":"a1b2c3d4","createdAt":"..."}]

(`expiresAt` aparece quando a credencial tem TTL.)
```

Revogar UMA credencial (`kfs_` vazou): o corpo leva o `serviceId` (dono) +
o proprio token a revogar. Inexistente, de outro servico ou de outra conta:
mesmo `404` opaco (nunca confirma nem nega token alheio).

```
POST /v1/services/token/revoke
X-KofCloud-Token: kfc_...
{"serviceId":"...","token":"kfs_..."}

200 {"revoked":true}
```

Registrar domínio próprio (Fase 6, slice 1):

```
POST /v1/domains
X-KofCloud-Token: kfc_...
{"hostname":"api.example.com","service":"site"}

200 {"hostname":"api.example.com","status":"PENDING",
     "instruction":"CNAME api.example.com -> site.mel.kofcloud.io"}
```

Verificar (o operador cria o CNAME e chama):

```
POST /v1/domains/verify
X-KofCloud-Token: kfc_...
{"hostname":"api.example.com"}

200 {"hostname":"api.example.com","status":"VERIFIED","verifiedAt":"..."}
```

`connect` (o Agent pede o ponto de entrada do Edge):
```
POST /v1/connect
X-KofCloud-Token: kfc_...
{"deviceId":"..."}

200 {"edgeHost":"127.0.0.1","edgePort":7000,"tunnelToken":"kft_...",
     "heartbeatMs":15000,"services":[ ... ]}
```

Presença real do túnel (o Edge espelha bind/unbind num mapa do Registry, não
persistido; filtrado por conta — inexistente/alheio nunca aparece):

```
GET /v1/presence
X-KofCloud-Token: kfc_...

200 ["<deviceId>", ...]
```

Sem túnel a lista vem vazia (ausência, não erro). `kof cloud status` mostra
`Tunel: ONLINE/OFFLINE` confirmado pelo Edge e o `doctor` checa a camada do
túnel antes dos serviços (roda também sem `kofcloud.json`).

## Erros

Todo erro viaja como `{"error":"KCFnnn: texto"}` — códigos estáveis (R6, como
o kof-share): nunca são removidos nem renumerados, só adicionados. A CLI
traduz para `Hint:` humana (`errHint` em `model/Errors.kf`).

| HTTP | Corpo | Quando |
|---|---|---|
| `400` | `KCF001` invalid handle | handle fora do formato |
| `400` | `KCF005` password too short | senha curta |
| `400` | `KCF020` name/publicKey | registro de device sem campos |
| `400` | `KCF040` invalid service name | nome de serviço fora do formato |
| `400` | `KCF043` localPort required | publish sem porta |
| `400` | `KCF044` serviceId required | endpoint sem serviceId |
| `401` | `KCF003` invalid credentials | conta/senha não conferem (opaco: conta inexistente responde igual) |
| `401` | `KCF004` unauthorized | token da conta ausente/inválido |
| `404` | `KCF021` device not found | device inexistente **ou de outra conta** (opaco por desenho) |
| `404` | `KCF041` service not found | serviço inexistente **ou de outra conta** (opaco por desenho) |
| `404` | `KCF122` domain not found | domínio inexistente **ou de outra conta** (opaco por desenho) |
| `400` | `KCF120` invalid custom hostname | hostname fora do formato, IP literal ou `*.kofcloud.io` |
| `409` | `KCF121` hostname already registered | hostname já registrado (outra conta ou outro serviço) |
| `409` | `KCF124` does not resolve to this Edge | DNS ainda não aponta para este Edge |
| `409` | `KCF002` handle taken | handle já em uso |
| `429` | `KCF080` too many active devices | limite `kofcloud.limits.devices` |
| `429` | `KCF081` too many services | limite `kofcloud.limits.services` por dispositivo |

Owner-gate: `POST /v1/services/status` só aceita a conta dona (`KCF042`
interno vira o mesmo `404 KCF041` na borda, para não revelar recurso alheio).
`handleRegisterDevice` responde `429` apenas para limite; erro de validação é
`400`.
