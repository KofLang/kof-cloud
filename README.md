# KofCloud

**A internet, sua. De novo.**

O KofCloud conecta um serviço que já roda na sua máquina à internet pública —
sem alugar servidor, sem IP público, sem port forwarding, sem configurar DNS,
TLS ou reverse proxy. A sua máquina continua sendo o servidor. O KofCloud só
resolve a parte que historicamente obrigava a alugar outro computador.

```
Internet ──HTTP──► KofCloud Edge ──túnel reverso──► KofCloud Agent ──► localhost:8080
                        │
                 KofCloud Control Plane
```

Escrito em **Kof** (alvo JVM). Ver [docs/](docs/README.md) para a arquitetura
completa e os protocolos.

## Status (medido, não aspiracional)

| Fase | Estado |
|---|---|
| 1 — túnel reverso persistente, heartbeat, reconexão com backoff | ✅ implementado e testado |
| 2 — publicação HTTP (`publish` → `curl` pela borda) | ✅ implementado e testado ponta a ponta |
| 3 — hostname automático + DNS (`*.handle.kofcloud.io`) | ✅ hostname/roteamento implementados; **DNS e TLS são externos** (o Kof não tem ACME nem TLS em socket cru — ver `docs/12-tls.md`) |
| 4 — múltiplos serviços no mesmo túnel + limites por dispositivo | ✅ implementado e testado |
| 5 — `AUTHENTICATED` com credencial de serviço (`kfs_`) | ✅ implementado e testado |
| 6 — domínio próprio (slice 1: registro + verificação DNS + rota) | ✅ implementado e testado; sem TLS/SNI (externo — ver `docs/12-tls.md`) |
| 6+ — observabilidade, deploy | parcial / planejado (`docs/16-plan.md`) |

O critério de sucesso do pedido foi atingido com o CLI real, em processos
separados: `login` → `publish site --port 29010` → `connect`, e
`curl -H "Host: site.mel.kofcloud.io" …` devolve o corpo do serviço local.
Prova em `tests/e2e_test.kf` e na seção "Rodar" abaixo.

## Requisitos

* A distribuição do Kof (`kof`) no PATH — usada aqui: `/home/mel/Kof4j/bin/kof`.
* Alvo **JVM** (o único alvo onde sockets TCP, `spawn`/`await`, canais,
  X25519/HKDF, AES-GCM e processos coincidem todos — ver `docs/13-kof-apis.md`).

## Rodar

Tudo a partir da raiz do repositório. O manifesto `kof.toml` declara as raízes
`app = "src"` e `test = "tests"` (ver `docs/14-modules.md`).

### 1. Subir a plataforma (control plane + edge)

```bash
kof run src/Main.kf serve --control-port 8080 --tunnel-port 7000 --http-port 8081 --state ~/.kofcloud/control.json
```

Isso sobe o control plane (HTTP/JSON em `:8080`), o listener de túneis
(TCP `:7000`) e o listener HTTP público (`:8081`) no mesmo processo,
compartilhando um registro único.

### 2. Usar o CLI (na máquina do usuário)

```bash
kof run src/Main.kf login mel --password segredo --control http://127.0.0.1:8080
kof run src/Main.kf publish site --port 8080
kof run src/Main.kf connect
```

`connect` mantém o túnel e reconecta sozinho. Enquanto ele roda:

```bash
curl -H "Host: site.mel.kofcloud.io" http://127.0.0.1:8081/
# → resposta do seu localhost:8080
```

> **Nota sobre `kof run`:** a forma posicional (`kof run src/Main.kf …`) usa o
> diretório do arquivo como raiz do módulo. Para empacotar e rodar de qualquer
> lugar, use `kof build --output <dir>` e execute a classe `Default.Main` pelo
> runtime do Kof, ou `kof deploy`.

### Comandos do CLI

```
login <handle> --password <p> [--control <url>] [--device <nome>]
logout
connect
disconnect
status
device list
device revoke <deviceId>
publish <nome> --port <porta> [--exposure PUBLIC|AUTHENTICATED|PRIVATE] [--address <addr>] [--device <nome>]
unpublish <nome>
service list
service status <nome>
service token <nome>        (credencial kfs_ para servico AUTHENTICATED)
domain list
domain add <hostname> --service <nome>   (dominio proprio, devolve a instrucao CNAME)
domain verify <hostname>                 (confere o DNS e ativa a rota)
domain remove <hostname>
logs
serve [--control-port N] [--tunnel-port N] [--http-port N] [--state <arquivo>]
```

## Testes

```bash
kof test --target jvm --timeout 120
```

Suítes verdes: framing, bytes, protocolo de túnel, hostnames/exposição,
identidade/cifra, registry, backoff, integração do control plane e E2E.
Ver `docs/15-tests.md` para a matriz.

## Segurança

* Cada instalação tem identidade criptográfica própria (par X25519 local); a
  chave privada nunca sai da máquina.
* O túnel pode ser cifrado ponta a ponta (`KOFCLOUD_TUNNEL_SECRET`), derivando
  a chave de sessão por X25519 + HKDF e cifrando os payloads com AES-GCM.
* Serviços têm exposição `PUBLIC` / `AUTHENTICATED` / `PRIVATE`; `PRIVATE` não
  tem rota pública. Bancos de dados não são publicados por padrão.
* O control plane guarda apenas hash dos tokens; comparações são
  constant-time (`security.constantTimeEquals`).
* Erros estáveis `KCFnnn` (padrão kof-share R6): a CLI imprime `Hint:` humana
  sem vazar segredos; recurso de outra conta responde o mesmo `404` opaco de
  inexistente; só a conta dona altera o status do serviço.
* Observabilidade mínima: o Edge conta túneis/recusas, requisições
  ok/negadas, bytes e latência; o control plane conta lifecycle (contas,
  logins, devices, serviços, tokens, domínios) — tudo via `observability.*`,
  servido sem auth em `GET /v1/metrics`; nenhuma métrica carrega hostname,
  corpo, credencial ou chave.

## Estrutura

```
kof-cloud/
├── kof.toml            manifesto (raízes app=src, test=tests)
├── src/
│   ├── Main.kf         entrada da CLI
│   └── kofcloud/
│       ├── protocol/   enquadramento, frames, tunnel/control, sessão
│       ├── model/      Account, Device, Service, Exposure
│       ├── identity/   X25519/HKDF/AES, tokens
│       ├── store/      persistência JSON
│       ├── control/    registry + HTTP /v1
│       ├── edge/       listener de túnel, router, HTTP front, relay
│       ├── agent/      connect/reconexão, relay, health, backoff
│       └── cli/        dispatcher e comandos
├── tests/              suítes (unit + integração + e2e)
└── docs/               arquitetura, protocolos, modelo, lacunas
```
