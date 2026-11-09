# Arquitetura do Edge

O Edge é a camada pública. Ele recebe tráfego da internet, resolve a rota e
encaminha pelo túnel. **A aplicação do usuário nunca fica exposta diretamente.**

```
                 ┌──────────────── KofCloud Edge ────────────────┐
  internet :443  │                                               │
  ──────────────►│  PublicListener (HTTP/1.1)                    │
                 │     │  parse request line + headers           │
                 │     ▼                                          │
                 │  Router ── host → {deviceId, service, port}   │
                 │     │                                          │
                 │     ▼                                          │
                 │  SessionRegistry ── deviceId → Session        │
                 │     │                                          │
  Agent :7000    │     ▼                                          │
  ◄──────────────│  TunnelListener (frames JSON por linha)       │
                 │     │  HELLO/WELCOME, OPEN/DATA/END            │
                 └─────┼─────────────────────────────────────────┘
                       ▼
                   Agent → localhost:8080
```

## Sub-componentes

| Módulo | Responsabilidade |
|---|---|
| `PublicListener` | aceita TCP, faz o parse HTTP/1.1, aplica limites e rate limit |
| `Router` | `Host` → serviço; aplica exposição (`PUBLIC`/`AUTHENTICATED`/`PRIVATE`) |
| `SessionRegistry` | `deviceId` → sessão de túnel viva; presença e `lastSeen` |
| `Session` | uma conexão de túnel: handshake, leitor de frames, escritor, streams |
| `Streams` | alocação de `streamId`, filas por stream, ciclo de vida |

## O que o Edge resolve

| Capacidade | Estado no MVP |
|---|---|
| HTTP/1.1 público | ✅ implementado (parser próprio sobre `net.listen`) |
| roteamento por `Host` | ✅ |
| WebSocket | ⏳ especificado; requer upgrade — ver `16-limitations.md` |
| HTTP/2 / HTTP/3 | ⏳ futuro (não bloqueia o MVP) |
| rate limiting | ✅ contador por host + janela (`security.rateLimit`) |
| TLS termination | ⏳ especificado (`11-tls.md`); Kof só tem TLS server no JVM |
| DNS | ⏳ externo por design (`10-dns.md`) |
| `AUTHENTICATED` | ✅ checagem de token antes do túnel |

## Limites aplicados no Edge

| Limite | Default | Efeito ao exceder |
|---|---|---|
| conexões públicas simultâneas | 1024 (`kofcloud.limits.publicConns`) | `503 KCF117` + fecha, no `accept` antes de ler o pedido; métrica `kofcloud_edge_http_connections_refused_total` (`conn_limit_test.kf`) |
| conexões simultaneas por servico | 64 (`kofcloud.limits.serviceConns`) | `429 KCF108` + `Retry-After` (antes do OPEN; aproximado sob corrida, ver abaixo) |
| corpo de requisicao | 1 MiB (`kofcloud.limits.httpBodyBytes`) | `413 KCF110` (antes do relay; por `Content-Length` declarado, lido em Long) |
| enquadramento do corpo | `Content-Length` único e numérico | `400 KCF118`: duplicado divergente, valor não numérico/vazio/com sinal, `Content-Length : 5`, header sem valor, ou `Transfer-Encoding` presente (chunked não implementado); antes de rotear; métrica `_invalid_framing` (`framing_test.kf`) |
| cabeçalho `Host` | exatamente um, não vazio, sem caractere de controle/espaço | `400 KCF119`: ausente, duplicado (mesmo igual) ou inválido; antes de rotear; métrica `_invalid_host`. O Edge roteia pelo `Host` e reconstrói o pedido para o serviço local — dois valores deixariam rota e aplicação vendo destinos diferentes (host-header injection) |
| cabeçalhos do pedido | cada linha no formato `Nome: valor` — nome `token` (sem espaço/controle), valor sem CR/LF/NUL; linha sem dois pontos também invalida | `400 KCF133` antes de rotear; métrica `_invalid_header_line`. Sem isso um `X-A\rb: v` era repassado cru e a linha se partia no serviço local (header injection) (`framing_test.kf`) |
| linha de requisição | `METODO SP PATH SP HTTP/1.0\|1.1` — exatamente três partes, método sem espaço/controle, path absoluto sem caractere de controle (`\r`/`\t`/DEL) | `400 KCF132`: quatro ou duas partes, linha de status no lugar do pedido, espaço duplo, path relativo/com controle ou versão não suportada; antes de rotear; métrica `_invalid_request_line` (`framing_test.kf`) |
| query string | não decide rota (`security.txt`) nem casa allowlist — `pathWithoutQuery`; o WAF continua vendo o path **com** query (é onde moram traversal/SQLi/XSS de query) | o pedido repassado mantém a query intacta. Antes `/.well-known/security.txt?x=1` caía no roteamento (404); e cortar a query do WAF cegava o detector (`security_test.kf`, `waf_detect_test.kf`) |
| query string | não decide rota (`security.txt`) nem casa allowlist — `pathWithoutQuery`; o WAF continua vendo o path **com** query (é onde moram traversal/SQLi/XSS de query) | o pedido repassado mantém a query intacta. Antes `/.well-known/security.txt?x=1` caía no roteamento (404); e cortar a query do WAF cegava o detector (`security_test.kf`, `waf_detect_test.kf`) |
| cabeçalhos repassados ao serviço | hop-by-hop removidos (`Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade`, `Proxy-Connection`) + tokens nomeados no `Connection` + `X-Forwarded-For` do cliente | não é recusa: o pedido segue sem eles, com `Connection: close` e `X-Forwarded-For: edge` escritos pelo Edge. `Proxy-Authorization` era repassado em claro ao serviço local (`framing_test.kf`) |
| headers da resposta ao cliente | hop-by-hop do upstream removidos, `Transfer-Encoding: chunked` **decodificado** e `Connection: close` escrito pelo Edge | não é recusa: o cliente recebe o status/corpo e um `close` — um `Connection: keep-alive` do serviço local pedia para reusar uma conexão que o Edge já fecha. Também vale para a resposta parcial (timeout). Corpo chunked que não fecha entrega o que decodificou (o close delimita), nunca o enquadramento cru (`framing_test.kf`) |
| corpo da resposta | 1xx/204/304 e `HEAD` nunca entregam corpo; 1xx/204 perdem o `Content-Length` | não é recusa: o Edge descarta os bytes e mantém status/headers (304 e HEAD preservam o `Content-Length` da representação). Sem isso um 204 com bytes fazia o cliente ler corpo onde não existe (`framing_test.kf`) |
| respostas 1xx intermediárias | blocos `1xx` do upstream (ex. `100 Continue`, `103 Early Hints`) são descartados até a resposta final | não é recusa: o cliente recebe o status/corpo finais. Sem isso o Edge entregava o bloco 1xx como resposta e o cliente nunca via o final; headers do interino também não vazam (`framing_test.kf`) |
| `Content-Length` da resposta | reescrito para o tamanho REAL do corpo (e corpo maior que o declarado é cortado no declarado) | não é recusa: o Edge corrige o header antes de entregar; sem isso um CL 2 com 7 bytes entregava os 7 (lixo no fim do stream) e um CL 100 com 2 bytes mantinha o header mentiroso. 304/HEAD preservam o CL (descreve a representação do GET); sem CL o Edge não inventa |
| espera de resposta do servico | 120 s (`kofcloud.limits.tunnelIdleMs`) | `504 KCF112` se nada chegou; parcial entregue se ja havia bytes |
| assinatura WAF | detectar: conta e segue, resposta marcada `X-KofCloud-WAF: would-block` (metodo+path, corpo e valores de header do cliente ate 64 KiB; `Authorization`/`Host`/`Content-Length`/`Connection`/`X-Forwarded-For` fora por desenho) | `serve --waf=enforce`: casa vira `403 KCF109` + `X-KofCloud-WAF: blocked` antes do relay, sem stream, sem Agent |
| requisições por hostname e janela | 100 / 60 s | `429 KCF107` + `Retry-After` |
| header total | teto de leitura do Edge | `431 KCF116` (sem `\r\n\r\n` dentro do teto) |
| resposta do serviço local | 16 MiB (`kofcloud.maxResponseBytes`) | `502 KCF115` + stream cancelado (nunca trunca) |
| streams por túnel | 128 | `RESET{LIMIT_EXCEEDED}` |
| itens na fila de um stream | 64 (`kofcloud.limits.queueDepth`) | `RESET{LIMIT_EXCEEDED}` no stream, túnel segue |

Os limites existem para proteger **as duas pontas**: a infraestrutura KofCloud e
a máquina do usuário. A contagem por serviço é aproximada sob corrida extrema
(check-then-act sem lock — o Kof nao expoe mutex, L3): o pior caso estoura em
poucas unidades, nunca recusa a menos nem trava; o rate limit (atomico)
segura o volume. O rate limit é por **hostname solicitado**
(`kofcloud.limits.httpRequests` / `kofcloud.limits.httpWindowMs`, fixed-window
do `security.rateLimit`): o Edge TCP não expõe IP de origem, e o hostname é o
que varredura e força bruta rotacionam menos. Recusado **antes** de rotear —
o 429 não revela se o host existe nem consome túnel. O nome chega normalizado
(`hostOnly`: apara, remove a porta, abaixa a caixa, tira o ponto final de
FQDN — DNS é case-insensitive, então `Host: Site...` roteava para 404 e cada
variação de caixa ganhava cota própria, evadindo o 429). A janela é configurada
em **milissegundos** e convertida para **segundos** (unidade de
`security.rateLimit`) num único ponto, `httpWindowSeconds` — arredonda para
cima, mínimo 1 s, e janela zerada/negativa recai no default. O `Retry-After`
deriva da mesma conversão (+1 s), então limite e recuo não divergem.

## Por que um listener HTTP próprio

O Kof oferece `web.app()`, mas um proxy precisa de controle fino que ele não
expõe hoje: ler o request cru, repassar bytes sem reescrever, controlar o
keep-alive do lado do cliente independentemente do lado do túnel, e mapear
conexão pública ↔ stream. O listener é pequeno (~200 linhas) e usa `net.listen`
+ `spawn`, primitivas estáveis. `web.app()` continua sendo a escolha certa para
o **control plane**, que é um serviço HTTP normal.

## Abuse prevention

O Edge assume que **nem todo serviço publicado é confiável**:

* rate limit por hostname (fixed-window) e por dispositivo;
* `AUTHENTICATED` para serviços sensíveis;
* `PRIVATE` sem rota;
* revogação de dispositivo e de serviço propagada imediatamente;
* auditoria operacional (conexão, desconexão, rota negada) **sem** conteúdo;
* `503`/`429`/`404` distintos para não vazar a existência de rotas privadas.

## Códigos de erro (R6)

O corpo das respostas de erro do Edge também viaja como `KCFnnn`:

| HTTP | Corpo | Quando |
|---|---|---|
| `404` | `KCF100` no route | host desconhecido **ou** serviço não roteável (PRIVATE) — opaco por desenho |
| `401` | `KCF101` authentication required | AUTHENTICATED sem credencial válida (+ `WWW-Authenticate: Bearer`) |
| `503` | `KCF102` device not connected | sem túnel vivo para o dispositivo dono |
| `503` | `KCF103` service … is … | serviço fora de rota conforme health reportado |
| `502` | `KCF104` tunnel error | falha no relay pelo túnel |
| `502` | `KCF105` empty response | serviço local devolveu resposta vazia |
| `429` | `KCF106` too many concurrent streams | limite `kofcloud.limits.streams` por túnel |
| `429` | `KCF107` too many requests | rate limit por hostname (+ `Retry-After`); antes de rotear, sem revelar nada |
| `429` | `KCF108` too many connections for this service | limite por serviço (+ `Retry-After`); antes do OPEN, aproximado sob corrida |
| `403` | `KCF109` blocked by edge protection | assinatura WAF casou com enforce ligado (`serve --waf=enforce`); antes do relay, sem stream (+ header `blocked`, métrica `_waf_denied` só com o nome do padrão); no detectar a resposta segue com `would-block` |
| `413` | `KCF110` request body too large | `Content-Length` acima de `kofcloud.limits.httpBodyBytes`; recusado antes do relay |
| `504` | `KCF112` upstream did not respond in time | servico local mudo alem de `kofcloud.limits.tunnelIdleMs`; watchdog entrega RESET no stream |
| `400` | `KCF118` invalid or ambiguous request framing | `Content-Length` duplicado divergente, nao numerico ou com sinal, nome fora da forma canonica, ou `Transfer-Encoding`; recusado antes de rotear. O Edge usa o declarado como fronteira do corpo que repassa — bytes alem dele ficam fora, para o servico local nunca receber um corpo de tamanho diferente do anunciado |
| `400` | `KCF119` missing or ambiguous Host header | `Host` ausente, duplicado (mesmo igual) ou com caractere de controle/espaco; recusado antes de rotear. Sem isso o Edge roteava pelo primeiro `Host` e repassava os dois ao servico local (o ultimo era o que a aplicacao via) — host-header injection |
| `400` | `KCF132` invalid or ambiguous request line | linha fora de `METODO SP PATH SP HTTP/1.x` (partes faltando/sobrando, método com espaço, path relativo) ou uma linha de status usada como pedido; recusado antes de rotear |
| `400` | `KCF133` invalid header line | cabeçalho fora de `Nome: valor` (nome com espaço/controle, valor com CR/LF/NUL, linha sem dois pontos); recusado antes de rotear |
| `502` | `KCF131` upstream sent an invalid HTTP response | o serviço local respondeu bytes sem linha de status enquadrável (`HTTP/<versão> <código 100–599>`); o Edge recusa em vez de repassar lixo ao cliente, sem ecoar os bytes recebidos; métrica `_invalid_response` (`framing_test.kf`) |

## Observabilidade mínima

O Edge conta eventos via `observability.*` do stdlib (`edge/Metrics.kf`) —
export Prometheus em `observability.metrics()`, servido sem auth em
`GET /v1/metrics` do control plane (mesmo processo no `serve`).

| Métrica | Tipo | O que conta |
|---|---|---|
| `kofcloud_edge_tunnel_accepted_total` | counter | túneis aceitos (handshake ok) |
| `kofcloud_edge_tunnel_refused_total{,_motivo}` | counter | recusas (`_version`, `_unknown_device`, `_revoked_device`, `_auth`) |
| `kofcloud_edge_tunnel_sessions_open` | gauge | sessões de túnel vivas |
| `kofcloud_edge_http_requests_total` | counter | requisições públicas terminadas |
| `kofcloud_edge_http_denied_total{,_motivo}` | counter | negadas (`_no_route`, `_auth_required`, `_device_offline`, `_service_down`, `_rate_limited`, `_service_overloaded`, `_body_too_large`, `_headers_too_large`, `_waf_denied`, `_waf_allow_denied`, `_invalid_framing`, `_invalid_host`, `_invalid_request_line`, `_invalid_header_line`, `_other`) |
| `kofcloud_edge_http_upstream_errors_total{,_timeout,_response_too_large,_invalid_response,_empty_response}` | counter | falhas de relay (502), por causa quando conhecida |
| `kofcloud_edge_http_request_bytes_total` | counter | bytes de pedido somados |
| `kofcloud_edge_http_response_bytes_total` | counter | bytes de resposta somados |
| `kofcloud_edge_tunnel_streams_total` | counter | streams abertos no relay |
| `kofcloud_edge_http_request_latency_ms_{count,sum}` | histogram | latência fim-a-fim por requisição (ms) |
| `kofcloud_edge_http_request_latency_ms_le_{50,100,250,500,1000,2000}` | counter | quantas requisições ficaram **até** cada faixa (ms, borda inclusiva) |
| `kofcloud_edge_http_request_latency_ms_gt_2000` | counter | requisições acima da última faixa (cauda) |

Regra de privacidade: só metadados operacionais agregados, sem cardinalidade
por cliente — nenhum hostname, caminho, header, corpo, credencial ou chave
chega às métricas (provado em `tests/metrics_test.kf`).
