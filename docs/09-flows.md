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

## 3.1 Presenca real do tunel (`GET /v1/presence`)

```
kof cloud status    → Tunel: ONLINE (sessao viva no Edge)
kof cloud doctor    → ok tunel ativo no Edge (sessao viva)

GET /v1/presence (X-KofCloud-Token: kfc_...)
  → ["<deviceId>", ...]   # so devices DA CONTA com sessao viva
                            # neste Edge; vazio = offline (nao erro)
```

O Edge espelha bind/unbind das sessoes num mapa do Registry (nao
persistido — restart limpa, reconexao recria). Filtrado por conta:
inexistente/alheio nunca aparece. `status` e `doctor` deixam de inferir
pelo processo local e passam a mostrar o que o Edge confirma.

## 3.2 Deploy declarativo (`kofcloud.json`)

```
kofcloud.json:
  {"services": [{"name": "site", "port": 8080},
                {"name": "api", "port": 3000, "exposure": "AUTHENTICATED"}]}

Campos por serviço: `name` (obrigatório), `port` (número ou string),
`address` (default 127.0.0.1; só loopback), `exposure` (default PUBLIC),
`protocol` (default http; único aceito no MVP), `build` (opcional: comando
de shell rodado antes do publish, ex. `"build": "npm run build"`; ausente,
vazio ou só-espaços = sem build), `artifact` (opcional: caminho local que o `--check`
confere, ex. `"artifact": "dist/index.html"`; ausente, vazio ou só-espaços = sem
checagem — aviso de staleness, nunca bloqueio). Campos de texto com
trim (espaços sobrando em `name`/`port`/`address`/`exposure`/`protocol` são
aparados; `name` só-espaços falha como sem nome). Todos validados no parse
com erro posicional — nada com default silencioso perigoso.

 1. usuário: kof cloud deploy [--file kofcloud.json] [--check] [--plan]
    com `--plan`: só mostra o diff (`+ novo`, `~ atualiza`, `= igual`,
    `- fora do manifesto`, com `(build)`/`(artifact)` em `+`/`~`/`=` quando o
    serviço declara cada um) via `diffDeployment` pura — nada aplicado, nada executado
 2. carrega identity + token + manifesto (parse com erro posicional)
 3. para cada serviço: roda `build` (se declarado; falhou = não publica
    esse serviço), confere `artifact` (se declarado e ausente = não publica
    esse serviço) e depois POST /v1/services (igual ao publish, idempotente)
 4. reporta publicados/mantidos/falhas (exit 1 se alguma falhar)
 5. usuário: kof cloud down → unpublish só dos listados (extra sobrevive)
    com `--prune`: remove também os órfãos DESTE device fora do manifesto
    (nunca toca serviço de outro device)
 6. com `--check`: publicado E respondendo localmente? (`checkLocalService`,
    mesma regra do Agent) + artefato declarado existe no disco? (aviso
    `artefato ausente`, nunca bloqueio) — sem rollback, só diagnóstico
```

Sem transferência de artefato neste slice: o build roda na máquina do
usuário (onde o código já está) e só o publish atravessa a rede.

## 3.3 Diagnostico (`kof cloud doctor`)

```
kof cloud doctor [--file kofcloud.json]
  1. login local presente? (senao: FALHA e para)
  2. GET /v1/health com o token? (control alcancavel + autenticado?)
  3. device registrado no control plane?
  4. para cada servico do manifesto (se houver kofcloud.json),
     via verifyDeployment (mesma ordem do deploy --check):
     nao publicado? config diferente do publicado (porta/address/
     exposure/protocol)? responde localmente? artefato declarado
     ausente no disco (aviso, nunca bloqueio)?
  ok em tudo -> exit 0; qualquer FALHA -> exit 1 + Hint do proximo passo
```

Somente leitura: nao publica, nao remove, nao revoga, nao emite token.

## 3.2 Daemon user-level (`kof cloud daemon`)

```
kof cloud daemon install   → grava ~/.config/systemd/user/kofcloud-agent.service
                             (ExecStart = mesmo `connect`, Restart=always,
                              NoNewPrivileges, WantedBy=default.target)
                             + systemctl --user daemon-reload (se houver)
kof cloud daemon status    → is-active (ou INATIVO + como subir)
kof cloud daemon uninstall → stop + remove a unit + reload
```

Escopo: só Linux com systemd --user, sem root, enable manual de propósito
(o Kof não mexe em serviços sem consentimento). Sem systemd: erro honesto
(`connect` em tmux/screen). Windows/macOS: Fase 8.
Nada fora do manifesto é tocado — deploy nunca remove.

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

Revogação de UMA credencial (vazou no log, no chat, no repo):

```
kof cloud service token-revoke secret kfs_abc123…
        │
        ▼  POST /v1/services/token/revoke {serviceId, token}
   conta dona? token pertence ao servico?
     ├── nao → 404 opaco (igual a inexistente; nao confirma nada alheio)
     └── sim → remove o hash, audita service.token_revoked, 200
```

Só a conta dona revoga; outros tokens do mesmo serviço continuam válidos
(rotação sem downtime); remover o serviço ou revogar o device poda o resto.

## 7. Revogação de dispositivo

```
kof cloud device revoke <nome|id>
  → control plane: status=REVOKED, rotas removidas
  → próxima HELLO: WELCOME{ok:false, reason:"DEVICE_REVOKED"} + close
  → Agent registra o motivo e PARA de reconectar (não é falha de rede)
```
