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
| 5 — `AUTHENTICATED` com credencial de serviço (`kfs_`, com revogacao por token) | ✅ implementado e testado |
| 6 — domínio próprio (slice 1: registro + verificação DNS + rota) | ✅ implementado e testado; sem TLS/SNI (externo — ver `docs/12-tls.md`) |
| 6+ — observabilidade, abuse prevention, limites | ✅ implementados e testados |
| 7 — deploy (slices 1–4: declarativo, --check, daemon, doctor) | ✅ implementado e testado; sem build/artefato, sem daemon Windows/macOS |

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

Ou declarativo, com `kofcloud.json` na pasta (`{"services":[{"name":"site",
"port":8080}]}` — ver `docs/09-flows.md` §3.1):

```bash
kof run src/Main.kf deploy [--file kofcloud.json] [--check]   # publica (+ verifica)
kof run src/Main.kf down                            # remove os do manifesto
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
device suspend <deviceId>      (túnel recusado, nada removido)
device unsuspend <deviceId>
publish <nome> --port <porta> [--exposure PUBLIC|AUTHENTICATED|PRIVATE] [--address <addr>] [--device <nome>]
unpublish <nome>
service list
service status <nome>
service suspend <nome>       (fora da rota, nada removido)
service unsuspend <nome>
service token <nome>        (credencial kfs_ para servico AUTHENTICATED)
service token-revoke <nome> <kfs_>  (revoga UMA credencial vazada)
domain list
domain add <hostname> --service <nome>   (dominio proprio, devolve a instrucao CNAME)
domain verify <hostname>                 (confere o DNS e ativa a rota)
domain remove <hostname>
deploy [--file kofcloud.json] [--check]   (publica + verifica saude local)
down [--file kofcloud.json]     (remove os servicos do manifesto)
doctor [--file kofcloud.json]   (diagnostico: login, control, device, tunel, servicos)
audit                        (trilha de auditoria da conta)
logs
serve [--control-port N] [--tunnel-port N] [--http-port N] [--state <arquivo>]
```

## Testes

```bash
export PATH=/home/mel/tools/jdk-25/bin:$PATH
java -jar /home/mel/Kof4j/kof-cli/target/kof-cli-0.6.3.jar test --target jvm --timeout 150
```
(CLI 0.6.3 + JDK 25; o `kof` 0.5.0-beta do PATH não entende `[sources]`. Ver
`docs/15-tests.md` e `docs/STATUS.md`.)

Suítes verdes: framing, bytes, protocolo de túnel, hostnames/exposição,
identidade/cifra, registry, backoff, integração do control plane, E2E,
domínio próprio, métricas, rate limit, suspensão/auditoria, frames
fatiados, idle timeout, conexões por serviço, deploy, daemon, hot-reload,
doctor e presença do túnel (25 suites). Ver `docs/15-tests.md` para a matriz.

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
  servido sem auth em `GET /v1/metrics`; o Agent conta connects,
  reconnects, health e streams no próprio processo; nenhuma métrica carrega
  hostname, corpo, credencial ou chave.
* Abuse prevention: rate limit por hostname no Edge (100 req/60 s,
  configurável) — excedeu, `429 KCF107` + `Retry-After` antes de rotear,
  sem revelar se o host existe e sem consumir o túnel; suspensão reversível
  de serviço/dispositivo (`SUSPENDED`, opaco) + trilha de auditoria
  (`kof cloud audit`, anel de 100 eventos só com metadados).
* Hot-reload: publicar depois do `connect` não exige reconnect (o Agent
  re-busca os serviços e o próximo OPEN já roteia); OPEN recusado responde
  `502` em vez de travar o relay.
* Presença real: o Edge espelha sessões vivas (`GET /v1/presence`, filtrado
  por conta); `status` mostra `Tunel: ONLINE/OFFLINE` confirmado pelo Edge e
  o `doctor` checa o túnel mesmo sem manifesto.

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
