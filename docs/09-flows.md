# Fluxos

## 1. Autenticação

```
Agent                                  Control Plane
  │  POST /v1/accounts {handle,email}         │
  │ ─────────────────────────────────────────►│  cria conta + handle único
  │  201 {accountId, token}                   │
  │ ◄─────────────────────────────────────────│
  │  grava token em ~/.kofcloud/session.json  │
  │                                           │
  │  POST /v1/devices {name, publicKey}       │  publicKey = X25519 pública
  │ ─────────────────────────────────────────►│  (privada nunca sai daqui)
  │  201 {deviceId}                           │
  │ ◄─────────────────────────────────────────│
  │  grava deviceId + chave em identity.json  │
```

Autorização: todo request subsequente leva `Authorization: Bearer <token>`. O
control plane compara o token com `security.constantTimeEquals`.

## 2. Conexão (`kof cloud connect`)

```
 1. carrega identity.json  ──► deviceId + chave privada X25519
 2. POST /v1/connect       ──► {edgeHost, edgePort, tunnelToken, services}
 3. net.connect(edgeHost, edgePort)                 ◄── conexão OUTBOUND
 4. HELLO {v, deviceId, token, caps, clientPub}
 5. WELCOME {ok:true, sessionId, serverPub, heartbeatMs}
 6. deriva sessionKey = HKDF(X25519(priv, serverPub))
 7. SERVICES {services[]}                            ◄── config efetiva
 8. spawn writer   (drena outbox → conn)
 9. spawn pinger   (PING a cada heartbeatMs)
10. loop de leitura de frames  ──► roteia DATA/OPEN para os streams
```

O passo 10 é o laço principal. Cada `OPEN` cria um stream; cada stream é uma
task que conecta em `localhost:<port>` e faz o relay.

**A aplicação publicada nunca é reiniciada** quando o túnel cai: o Agent
reconecta e os serviços continuam rodando.

## 3. Publicação

```
 1. usuário: kof cloud publish site --port 8080
 2. carrega identity + token
 3. POST /v1/services {deviceId,name,localPort,protocol,exposure}
 4. control plane gera hostname = site.<handle>.kofcloud.io
 5. control plane devolve o serviço
 6. Agent grava services.json e envia STATUS/refresh ao Edge pela sessão viva
 7. Edge passa a rotear Host: site.<handle>.kofcloud.io → este dispositivo
```

Sem Nginx, sem Caddy, sem certificado manual.

## 4. Requisição pública (ponta a ponta)

```
curl https://site.mel.kofcloud.io
  │
  ▼  DNS: *.mel.kofcloud.io → Edge
Edge :443  (TLS terminate)
  │  Host = site.mel.kofcloud.io
  │  Router.resolve(host) → deviceId, localPort, exposure
  ▼
stream s=7  OPEN{s:7, service:"site"}
  │  DATA{s:7, d:base64(req)}
  ▼  (túnel cifrado)
Agent
  │  inbox[7].receive() → connect 127.0.0.1:8080
  │  send(req) → receive(resp)
  ▼  DATA{s:7, d:base64(resp)}
Edge
  │  escreve a resposta no socket do cliente
  ▼
curl recebe 200
```

## 5. Reconexão

```
CONNECTED
    │  falha de rede / troca de IP / CGNAT muda / roteador reinicia
    ▼
DISCONNECTED
    │  backoff exponencial com jitter
    ▼
RECONNECTING   delay = min(base * 2^attempt, max) * (1 ± jitter)
    │
    ▼
AUTHENTICATE   HELLO/WELCOME
    │
    ▼
RESTORE ROUTES  SERVICES + STATUS de cada serviço
    │
    ▼
CONNECTED
```

Parâmetros (default):

| Parâmetro | Valor |
|---|---|
| `base` | 500 ms |
| `max` | 30 s |
| `jitter` | ±20 % |
| tentativas | infinitas (o Agent vive enquanto a máquina viver) |

Nenhuma reconfiguração manual. O IP pode mudar; a conexão é outbound, então
nada precisa ser reaberto no roteador.

## 6. Falha de serviço local

```
Agent health check: connect 127.0.0.1:8080  → recusa
  → STATUS{service:"site", status:"OFFLINE"}
  → Edge marca a rota OFFLINE
  → requisição pública devolve 503
  → (o túnel continua; os outros serviços não são afetados)
  → quando o serviço volta, STATUS{...:"ONLINE"} reativa a rota
```

## 6.1 Acesso a serviço AUTHENTICATED

```
kof cloud service token secret        (conta emite kfs_ no control plane)
        │
        ▼
curl -H 'Authorization: Bearer kfs_…' https://secret.mel.kofcloud.io/
        │
        ▼  Edge
   authorize(serviço, kfs_)
     ├── inválido/ausente/outro serviço → 401 + WWW-Authenticate: Bearer
     └── válido → remove o Authorization e segue para o túnel
        │
        ▼  Agent → localhost:<porta>  (app já não vê a credencial)
```

O token da conta (`kfc_`) não passa nessa checagem: administração e acesso
público são credenciais distintas.

## 7. Revogação de dispositivo

```
kof cloud device revoke <id>
  → control plane: status=REVOKED, rotas removidas
  → próxima HELLO: WELCOME{ok:false, reason:"DEVICE_REVOKED"} + close
  → Agent registra o motivo e PARA de reconectar (não é falha de rede)
```
