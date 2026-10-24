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
| conexões públicas simultâneas | 1024 | `503` + fecha |
| conexões simultaneas por servico | 64 (`kofcloud.limits.serviceConns`) | `429 KCF108` + `Retry-After` (antes do OPEN; aproximado sob corrida, ver abaixo) |
| requisições por hostname e janela | 100 / 60 s | `429 KCF107` + `Retry-After` |
| header total | 16 KiB | `431` |
| corpo | 16 MiB | `413` |
| streams por túnel | 128 | `RESET{LIMIT_EXCEEDED}` |

Os limites existem para proteger **as duas pontas**: a infraestrutura KofCloud e
a máquina do usuário. A contagem por serviço é aproximada sob corrida extrema
(check-then-act sem lock — o Kof nao expoe mutex, L3): o pior caso estoura em
poucas unidades, nunca recusa a menos nem trava; o rate limit (atomico)
segura o volume. O rate limit é por **hostname solicitado**
(`kofcloud.limits.httpRequests` / `kofcloud.limits.httpWindowMs`, fixed-window
do `security.rateLimit`): o Edge TCP não expõe IP de origem, e o hostname é o
que varredura e força bruta rotacionam menos. Recusado **antes** de rotear —
o 429 não revela se o host existe nem consome túnel.

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
| `kofcloud_edge_http_denied_total{,_motivo}` | counter | negadas (`_no_route`, `_auth_required`, `_device_offline`, `_service_down`, `_rate_limited`, `_service_overloaded`, `_other`) |
| `kofcloud_edge_http_upstream_errors_total` | counter | falhas de relay (502) |
| `kofcloud_edge_http_request_bytes_total` | counter | bytes de pedido somados |
| `kofcloud_edge_http_response_bytes_total` | counter | bytes de resposta somados |
| `kofcloud_edge_tunnel_streams_total` | counter | streams abertos no relay |
| `kofcloud_edge_http_request_latency_ms_{count,sum}` | histogram | latência fim-a-fim por requisição (ms) |

Regra de privacidade: só metadados operacionais agregados, sem cardinalidade
por cliente — nenhum hostname, caminho, header, corpo, credencial ou chave
chega às métricas (provado em `tests/metrics_test.kf`).
