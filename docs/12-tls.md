# Estratégia de TLS

HTTPS é o padrão. Não existe modo "HTTP público" como comportamento default.

```
Browser ── HTTPS ──► Edge ── túnel cifrado ──► Agent ──► localhost:8080
```

## O que o Kof oferece hoje (medido)

| Capacidade | Kof | Alvo |
|---|---|---|
| servidor TLS | `app.listenSecure(port)` / `listenSecure(port, certPem, keyPem)` | **JVM only** (`WEB002` fora do JVM) |
| cliente HTTPS | `http.get("https://…")` | JVM + JS; **native lança** |
| certificado de arquivo | PKCS#8 PEM via `listenSecure(port, certPem, keyPem)` | JVM |
| ACME / Let's Encrypt | **não existe** | — |
| TLS sobre socket cru (`net`) | **não existe** | — |
| SNI explícito | **não existe** | — |

## Consequência de arquitetura

O Edge **não** implementa TLS sobre sockets crus, porque o Kof não expõe TLS em
`net.*`. Duas opções reais:

1. **Edge = `web.app()` com `listenSecure`** (JVM). O `web.app()` aceita
   `listenSecure(port, certPem, keyPem)` e serve HTTPS. O handler resolve o
   `Host` e faz o relay pelo túnel. É o caminho suportado pelo Kof hoje.
2. **Terminação TLS fora do processo Kof** (ex.: um terminador na frente) e o
   Edge Kof escuta HTTP em loopback. É o que a produção real faria enquanto o
   Kof não tiver TLS em socket cru.

O núcleo do Edge (router + túnel + streams) é o mesmo nos dois casos: ele
trabalha com bytes de requisição já decifrados.

## Emissão e renovação de certificado

O Kof **não tem ACME**. Então:

* **MVP/demonstração:** certificado do Edge é fornecido ao operador como PEM
  (PKCS#8) e passado a `listenSecure(port, certPem, keyPem)`. Para `*.mel.kofcloud.io`
  é um certificado wildcard emitido uma vez.
* **Futuro:** um conector ACME (DNS-01, porque o Edge é wildcard) escrito em
  Kof usando `net` + `crypto` (o handshake ACME é HTTP + JWS/ES256 — o Kof tem
  o HTTP; a assinatura ES256 ainda **não** está exposta, só Ed25519). Enquanto
  isso, a emissão é externa.

## Regras

* Renovação é responsabilidade da plataforma, não do usuário.
* O usuário nunca gera, instala ou renova certificado.
* Redirecionamento `80 → 443` é responsabilidade do Edge (fora do núcleo Kof se
  a terminação for externa).
* O túnel Edge↔Agent tem cifra própria (`07-tunnel-protocol.md`), independente
  do TLS público — defesa em profundidade.

## Lacuna documentada

`TLS001` — **TLS em socket cru e ACME não existem no Kof.** O Edge valida o
caminho HTTPS apenas via `web.app().listenSecure` no target JVM. Ver
`12-kof-apis.md` §Lacunas e `16-limitations.md`.

## Estado entregue no KofCloud (medido 2026-11-04)

* **Control plane HTTPS ✅:** `serve --tls-cert/--tls-key` →
  `listenSecure(port, certPem, keyPem)`; HSTS vem do runtime do Kof
  (`Policy.headers` + gate em `req.secure`). O fluxo real da CLI
  (`loginOrCreate` + `registerDevice`) está provado sobre HTTPS em
  `tests/control_test.kf` (`control-tls-cli`).
* **PEM × hostname ✅ (antes da borda):** `domain verify --tls-cert <pem>`
  julga o certificado do operador localmente e **antes de qualquer rede** —
  SAN DNS (fallback CN) cobre o hostname? Curinga vale um rótulo, nunca o
  domínio base (RFC 6125). Não cobre = `FALHA tls` e a verificação não
  segue; expira em 30 dias = aviso na linha `ok`; `openssl` ausente = aviso
  (mesma regra do `doctor`/`scan`). Só caminho e nomes aparecem, nunca o
  conteúdo. Prova: `tests/domains_test.kf` (`domains-cert`).
* **Edge público: segue HTTP.** Duas barreiras medidas, não suposições:
  `net` não tem TLS (`KofNet.java` só expõe `listen`/`connect` crus — L2) e o
  handler do `web.app()` devolve `String`, corrompendo bytes altos (L16). O
  Edge tem parser HTTP próprio sobre socket cru e precisa repassar binário.

> **Ressalva de segurança do cliente (importante).** O cliente `kof.http` é
> **trust-all por construção**: `JvmWebHttpRuntime.java:112-132` instala um
> `X509TrustManager` que aceita qualquer certificado e desliga
> `endpointIdentificationAlgorithm` para **toda** requisição `https://` — não
> há opt-in nem env. Consequência honesta: com o control plane em HTTPS o
> conteúdo (login, `kfc_`) viaja **cifrado**, mas o cliente **não autentica o
> servidor** — um MITM com certificado forjado é aceito. Mitigação hoje:
> control plane em rede confiável/loopback, ou terminador TLS verificado na
> frente. Verificação real depende do slice "cliente `kof.http` verificador"
> do plano do Kof.

> **Plano a jusante:** o endurecimento de TLS do próprio Kof (SNI por
> hostname, hot-reload de certificado, cliente `kof.http` verificador,
> ACME DNS-01) foi **promovido de `future/` para `docs/development/`** em
> 02/11 e tem fatias pousadas no branch `lab` (SNI 02/11 21:45, hot-reload
> 02/11 22:08, ACME JWS ES256 03/11 00:18). Nenhuma está nos jars pinados
> (0.6.3 e 0.6.7, ambos de 27/10), então o KofCloud ainda não as usa. Quando
> saírem num release, a Fase 9 do KofCloud (slices 2–3 em `16-plan.md`) ganha
> SNI e rotação de certificado sem restart.
