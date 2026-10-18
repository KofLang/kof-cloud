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
