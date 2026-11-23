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
| 7 — deploy (declarativo, --check/--plan, daemon, doctor + build local) | ✅ implementado e testado; sem artefato transferível, com artefato de daemon por SO (systemd/launchd/Windows, ativação manual) |

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
compartilhando um registro único. Com `--tls-cert cert.pem --tls-key
key.pem`, o control plane serve HTTPS (o login e o `kfc_` viajam cifrados;
o Edge público segue HTTP — ver `docs/12-tls.md`).

> **Ressalva honesta:** o cliente HTTP do Kof (`kof.http`) é trust-all —
> aceita qualquer certificado. O HTTPS do control **cifra** o login/token,
> mas o CLI **não autentica** o servidor. Use em rede confiável/loopback ou
> com um terminador TLS verificado na frente. Ver `docs/12-tls.md`.

### 2. Usar o CLI (na máquina do usuário)

```bash
kof run src/Main.kf login mel --password segredo --control http://127.0.0.1:8080
kof run src/Main.kf publish site --port 8080
kof run src/Main.kf connect
```

Ou declarativo, com `kofcloud.json` na pasta (`{"services":[{"name":"site",
"port":8080}]}` — `"build": "npm run build"` opcional por serviço, roda
antes do publish; `"artifact": "dist/index.html"` opcional, conferido no
deploy e no `--check`; ver `docs/09-flows.md` §3.2):

```bash
kof run src/Main.kf deploy [--file kofcloud.json] [--check] [--plan] [--watch]   # publica (+ verifica; --plan so mostra; --watch observa e re-publica o que mudar — nunca remove)
kof run src/Main.kf down --yes [--prune]        # remove os do manifesto (+ orfaos do device)
```

Em CI/CD, veja o exemplo de pipeline GitHub Actions em
`docs/examples/ci-pipeline.yml` (login com device fixo `ci`,
`deploy --plan` revisável no PR e `deploy --check` falhando o
pipeline) e a action reutilizável em `docs/examples/action.yml`
(composite, pronta para copiar e usar com `uses:`); ambos exigem
runner self-hosted na máquina do `connect`.

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
device list [--json]   (--json = saida estavel para CI, com o eixo de presenca)
device revoke <nome|deviceId> [--yes]  # irreversivel: sem --yes so avisa
device suspend <nome|deviceId>      (túnel recusado, nada removido)
device unsuspend <nome|deviceId>
publish <nome> --port <porta> [--exposure PUBLIC|AUTHENTICATED|PRIVATE] [--address <addr>] [--device <nome>]
        [--allow-method VERBO]... [--allow-path /caminho]...
unpublish <nome>
service list [--json]   (--json = saida estavel para CI)
service status <nome> [--tls-cert <pem>]  (com o PEM, julga expiracao e cobertura para o hostname do servico + dominios proprios)
service suspend <nome>       (fora da rota, nada removido)
service unsuspend <nome>
service token <nome> [--ttl <horas>]  (credencial kfs_ para AUTHENTICATED; ex.: --ttl 24)
service token-list <nome>   (inventario, sem valores)
service token-revoke <nome> <kfs_|id>  (revoga UMA credencial; id curto se unico)
service tokens   (credenciais validas SEM expiracao na conta)
service rotate (<nome>|--all) [--ttl <horas>] [--prune]  (credencial nova; revoga as antigas SO depois do TTL)
service rotation <nome>  (quantas credenciais ainda valem)
domain list [--json]   (--json = saida estavel para CI)
domain add <hostname> --service <nome> [--dns-webhook <url> | --dns-cmd <cmd>] [--dry-run]   (dominio proprio; UMA fonte de DNS — --dns-webhook pede a criacao do CNAME num receptor HTTP do operador, token via KOF_KOFCLOUD_DNS_WEBHOOK_TOKEN, ou --dns-cmd roda o CLI do provedor com {hostname}/{target}/{service}; --dry-run mostra e nao registra nem envia)
domain verify <hostname> [--tls-cert <pem>]  (confere o DNS e ativa a rota; --tls-cert julga o PEM antes da rede)
domain remove <hostname>
deploy [--file kofcloud.json] [--check] [--plan] [--watch]  (roda build + publica + verifica; --plan so mostra o diff; --watch observa o manifesto e re-publica o que mudar — nunca remove)
down [--file kofcloud.json] [--prune] [--yes]  (remove os do manifesto; --prune remove orfaos; sem --yes so avisa)
daemon install|status|uninstall   (artefato por SO — systemd/launchd/Windows — sem root, ativação manual)
doctor [--file kofcloud.json] [--tls-cert <pem>]  (diagnostico: login, control, device, tunel, auditoria, tls, servicos)
usage [--export]   (uso da conta: requests e bytes roteados pelo Edge; --export fecha a janela — le e zera, para billing externo)
metrics [--grep <texto>] [--percentiles] [--control <url>]   (export Prometheus do control; --percentiles = p50/p95/p99 agregados, teto de faixa; sem auth, so agregados)
audit [--sbom] [--verify] [--action <acao>] [--tail <n>]   (trilha da conta; --sbom = inventario local; --action filtra, --tail corta)
scan [--file <caminho>] [--tls-cert <pem>]   (segredos no manifesto/ambiente, PEM expirando e .kfs commitado no repo; exit 1 com achado)
webhook set <url> --secret <s> | show | remove | test
freeze service|device <nome|id> | account
unfreeze service|device <nome|id> | account
incidents [--all] [--resolve <id>]
migrate --from <estado.json> [--to <destino.json>] [--dry-run]
state-backup [--file <estado.json>]
token list|revoke <id|kfc_...>|rotate [--ttl <horas>]|rotation|prune   (prune: revoga as antigas, mantém a da sessão)
logs [--tail <linhas>] [--runtime|--edge]   (marcos de sessao; --runtime = eventos do Agent; --edge = eventos do Edge; teto 256 KiB por arquivo)
serve [--control-port N] [--tunnel-port N] [--http-port N] [--state <arquivo>] [--backup-dir <dir>] [--panel] [--status-page] [--waf detectar|enforce] [--tls-cert <pem> --tls-key <pem>]
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
doctor, presença do túnel, eventos do agent, teto de corpo WAF, detectar
WAF, timeout de upstream, allowlist WAF, painel somente-leitura, rotação de kfs_, security.txt, incidentes, contenção (service/device/account), status page, webhook de incidentes, cadeia de auditoria, scan de ambiente/PEM, higiene de credenciais, painel de gestão completa, ranking de Edges, carga explícita/migrate, teto de resposta no relay, credencial da conta com ciclo de vida, aviso de expiração próxima da credencial da conta no `token list`, camada própria no `doctor` e aviso no `status`, e contagem de `kfs_` perto do fim no `service status` (mesmo predicado do `token-list`) e aviso de `kfs_` perto do fim no `deploy --check`, no `deploy --plan` e no `doctor`, health HTTP com prazo (só 2xx é ONLINE), fila limitada por stream, teto de conexões públicas, enquadramento de `Content-Length`, `Host` não ambíguo, hop-by-hop nos dois sentidos, resposta chunked, status sem corpo, `Content-Length` da resposta, resposta inválida do upstream, linha de requisição, cabeçalhos, respostas 1xx, presença no `device list`, erro do login com cadastro implícito, flags com valor em lista única e cobertura do host no `doctor --tls-cert` e check local do deploy com GET/prazo e origem do serviço na CLI e escolha do dispositivo no painel e criar conta e registrar device e webhook pelo painel e diagnóstico do doctor (nomeia o estado ilegível do control em vez de "fora do ar") e camada da credencial no Diagnóstico do painel e banner de onboarding no painel e fluxo guiado de 3 telas no painel e contenção (freeze/unfreeze) no painel e sufixos de métrica completos e portas lidas sem wrap na borda do Agent/webhook cadeia de auditoria por conta e contrato do `/v1/connect` documentado rate limit no login, guarda de `--resolve` no uso, booleanas sem duplicação e recusa de nome homônimo em status/token e contrato `/v1` documentado e varredura periódica de credenciais vencidas, eventos do Edge (`logs --edge`), SERVICES empurrado pelo túnel no hot-reload, `deploy --watch` uso por conta com ciclo de billing (le-e-zera), e segredos de fonte externa no serve (`--tunnel-secret-cmd`/`--tls-cert-cmd`) webhook de uso para billing, teto de corpo por rota (`--max-body`) e percentis de latência (`metrics --percentiles`) e provedor DNS por comando (`--dns-cmd`) e readiness do control plane (`/v1/health` 503 com estado corrupto + `serve` fail-closed) bind visível no serve (porta ocupada = recusa, nunca borda silenciosa) base64 linear em bytes no relay (`protocol/Bytes.kf`, 100 KiB em ~46 ms contra ~1,1 s do caminho antigo) e leitura do pedido linear (`bytesGrown`, 1 MiB em ~16 ms contra ~4,8 s) backup automático do estado no `serve --backup-dir` (estável, com teto) `metrics` na CLI listagens em `--json` (73 suites). Ver `docs/15-tests.md` para a matriz.

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
  configurável em `httpRequests`/`httpWindowMs`; a janela configurada em ms
  chega ao stdlib em segundos por um ponto único, `Limits.httpWindowSeconds`)
  — excedeu, `429 KCF107` + `Retry-After` antes de rotear,
  sem revelar se o host existe e sem consumir o túnel; suspensão reversível
  de serviço/dispositivo (`SUSPENDED`, opaco) + trilha de auditoria
  (`kof cloud audit`, anel de 100 eventos só com metadados + espelho
  append-only `<estado>.worm.jsonl` com teto de 8 MiB que poda a cabeça
  (`kofcloud.limits.wormMirrorBytes`); `audit --verify` confirma
  anel E espelho).
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
