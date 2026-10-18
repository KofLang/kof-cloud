# APIs Kof necessárias — e as que faltam

Todas as linhas abaixo foram **medidas** compilando e executando sondas contra
`kof` 0.6.4 (JVM), não inferidas da documentação. Onde a doc diverge do
comportamento, vale o comportamento.

## Necessárias e disponíveis (verificadas em execução)

| Necessidade | API | Verificado |
|---|---|---|
| aceitar conexão TCP | `net.listen(port)` + `listener.accept()` | ✅ |
| conectar (outbound) | `net.connect(host, port)` | ✅ |
| ler/escrever bytes | `conn.receive(n)`, `conn.send(Byte[])` | ✅ |
| resolver DNS | `net.resolve(host) → List<String>` | ✅ |
| concorrência | `spawn { }`, `await h` | ✅ |
| tarefas + canais | `channel<T>()`, `.send`, `.receive` | ✅ |
| espera | `time.sleep(ms)`, `time.now()` | ✅ |
| IDs | `uuid.v4()` | ✅ |
| JSON | `json.encode`, `json.decode<T>` | ✅ |
| base64/hex | `encoding.base64Encode/Decode`, `hexEncode` | ✅ |
| hash/HMAC | `crypto.sha256`, `crypto.hmacSha256` | ✅ |
| cifra simétrica | `crypto.encryptAesGcm/decryptAesGcm` | ✅ |
| aleatório seguro | `crypto.randomHex(n)`, `security.randomInt` | ✅ |
| comparação segura | `security.constantTimeEquals` | ✅ |
| rate limit | `security.rateLimit(key, limit, window)` | ✅ |
| acordo de chaves | `keyExchange.privateKey/publicKey/shared` (X25519) | ✅ JVM |
| KDF | `keyExchange.hkdfSha256(Secret, saltHex, infoHex, len)` | ✅ JVM |
| assinatura | `crypto.sign/verify` (Ed25519) | ✅ JVM |
| `Secret` | `secrets.of`, `.reveal()` | ✅ JVM |
| HTTP server (control plane) | `web.app()`, `app.get/post`, `app.listen` | ✅ |
| HTTP client (Agent) | `http.get/post`, `http.timeout/retry` | ✅ |
| filesystem | `File.readText/writeText`, `Directory.createDirectories` | ✅ |
| processos | `process.run`, `process.exit` | ✅ |
| config | `config.get/env/int/str/bool/required` | ✅ |
| log | `log.info/warn/error` | ✅ |
| observabilidade | `observability.health/metrics/counter` | ✅ |

## Lacunas do Kof (bloqueiam ou moldam o desenho)

### L1 — `Conn`/`Listener` não são classes reais ⚠️ **molda o desenho**

Medido: usar `Conn` como **campo de classe** ou **tipo de retorno de `spawn`**
produz `NoClassDefFoundError: KofRuntime$NetConn` (o descritor emitido referencia
o tipo sem pacote). Funciona como variável local, parâmetro e captura de closure.

**Consequência:** nenhuma classe guarda um `Conn`. Sessões recebem o `Conn` por
parâmetro; streams guardam `Channel<String>` (tipo real) num `Map`. Já
incorporado ao desenho.

### L2 — Sem TLS em socket cru, sem ACME

`net.*` não expõe TLS; `listenSecure` é JVM-only e pertence a `web.app()`.
Não há ACME em lugar algum. **Consequência:** terminação TLS via `web.app()`
(JVM) ou fora do processo; certificados emitidos externamente. Ver `12-tls.md`.

### L3 — Sem watch de arquivo, sem locks, sem chmod

Sem inotify/watch, sem mutex/atomic (por decisão da linguagem), sem API de
permissão. **Consequência:** permissão `0600` de `identity.json` é *best-effort*
(o Kof não expõe `chmod`); a concorrência usa canais, nunca locks.

### L4 — Sem `String.getBytes()` fora do JVM (`STR003`)

**Consequência:** payloads binários são montados com `new Byte[n]` + atribuição
por elemento; `encoding.hexDecode` devolve `String`, não bytes.

### L5 — `main(args)` vazio em Native/JS

**Consequência:** a CLI é JVM-first. Em native, os argumentos não chegam.

### L6 — Sem HTTP/2, WebSocket server-side em native/JS

`app.ws` é JVM-only (`WEB004`); não há upgrade de protocolo no listener cru.
**Consequência:** WebSocket público fica fora do MVP (documentado).

### L7 — Sem chunked request body / streaming HTTP

O corpo é lido inteiro. **Consequência:** upload grande é limitado por
`limits.frameBytes` × fragmentação; streaming contínuo (SSE longo) fica
limitado pelo idle timeout.

### L8 — `web.app()` intercepta o header `Authorization` ⚠️ **moldou o protocolo**

Medido em `kof-compiler/.../jvm/JvmRuntimeWebDispatch.java` (passo 6 do
pipeline): se a requisição traz `Authorization` e o app não configurou
`authHeader`, o runtime valida o valor **como Bearer JWT** e, se não for um JWT
válido, devolve **401 antes do handler** — mesmo para um token opaco legítimo da
aplicação. `http.post` ainda devolve esse corpo 401 (ver L9), então a falha
passa silenciosa se o código só decodifica o corpo.

**Consequência:** o control plane (que roda sobre `web.app()`) usa o header
próprio `X-KofCloud-Token`. O Edge público, com parser HTTP próprio, aceita
`Authorization: Bearer` normalmente para serviços AUTHENTICATED.

### L9 — `http.*` não lança em 4xx

Medido em `JvmWebHttpRuntime.kof_http_request`: o cliente só lança para status
≥ 500 ou erro de rede; em 4xx devolve o corpo. **Consequência:** o código que
usa o control plane passa por `requireOk(raw)` (`kofcloud/util/Text.kf`), que
transforma um corpo `{"error":"..."}` em exceção — sem isso, um login recusado
virava token `""` silenciosamente.

### L10 — Interromper um `accept()` bloqueado exige fechar o `Listener`

Não há timeout de accept nem sinal. Medido: `listener.close()` faz o `accept()`
bloqueado lançar `"kof.net: accept failed: Socket closed"` (capturável). E um
`accept()` em voo **mantém o processo JVM vivo** (diferente de
`web.app().listen`, que não bloqueia a saída).

**Consequência:** os listeners do Edge são paráveis por um arquivo-sentinela
observado por um watcher que chama `listener.close()`. É o mecanismo de parada
limpa do `serve` e o que permite o E2E terminar sozinho.

## APIs ausentes que seriam necessárias para o produto completo

| Necessidade | Estado | Contorno no MVP |
|---|---|---|
| ACME/Let's Encrypt | ausente | certificado externo (wildcard) |
| TLS em socket cru | ausente | `web.app().listenSecure` (JVM) |
| HTTP/2 e HTTP/3 | ausente | HTTP/1.1 |
| WebSocket no Edge | parcial (JVM) | fora do MVP |
| watch de arquivo | ausente | polling |
| permissões de arquivo | ausente | best-effort |
| assinatura ES256 | ausente (só Ed25519) | token opaco + X25519 |
| socket Unix | ausente | loopback TCP |
| relay UDP | parcial | HTTP/HTTPS apenas |
