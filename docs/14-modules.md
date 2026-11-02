# Estrutura dos módulos Kof

```
kof-cloud/
├── kof.toml                       manifesto: [sources] app="src", test="tests"
├── src/
│   ├── Main.kf                    entrada da CLI: `kof cloud …`
│   └── kofcloud/
│   ├── protocol/                  enquadramento e protocolo do túnel (sem I/O)
│   │   ├── Version.kf             versão do protocolo + terminador de linha
│   │   ├── Bytes.kf               bytes ↔ string, base64, slice/concat
│   │   ├── Frame.kf               frame JSON por linha (kind/field/int/bool)
│   │   ├── LineCodec.kf           buffer + split por \n (fail-closed)
│   │   ├── Payload.kf             encode/decode + fatiamento em chunks
│   │   ├── Session.kf             TunnelSession: streams, filas, idle
│   │   └── Tunnel.kf              HELLO/WELCOME/PING/OPEN/DATA/END/RESET/GOAWAY
│   ├── model/                     tipos de domínio (records + literais)
│   │   ├── Account.kf
│   │   ├── Device.kf              ACTIVE/REVOKED/SUSPENDED + predicados
│   │   ├── Service.kf             status + exposure + predicados
│   │   ├── Exposure.kf            PUBLIC/AUTHENTICATED/PRIVATE + defaults
│   │   ├── CustomDomain.kf        PENDING/VERIFIED + verificação DNS
│   │   ├── Audit.kf               nomes dos eventos da trilha
│   │   └── Errors.kf              erros estáveis KCFnnn + Hint
│   ├── identity/
│   │   ├── Keys.kf                X25519/HKDF/AES-GCM, session key
│   │   └── Tokens.kf              geração/verificação de token (hash opaco)
│   ├── store/
│   │   └── Store.kf               persistência JSON (control plane + agent)
│   ├── control/
│   │   ├── Registry.kf            estado em memória (contas, devices,
│   │   │                         serviços, tokens, domínios, presença)
│   │   ├── ControlPlane.kf        HTTP `/v1/…` sobre `web.app()`
│   │   └── Metrics.kf             contadores de lifecycle
│   ├── edge/
│   │   ├── EdgeState.kf           rotas + sessões vivas + limites
│   │   ├── Tunnel.kf              listener de túneis + handshake + sweeper
│   │   ├── Http.kf                parsing/leitura de HTTP/1.1 cru
│   │   ├── HttpFront.kf           roteamento por Host + relay público
│   │   ├── Relay.kf               request/response por stream (com OPENED)
│   │   └── Metrics.kf             túneis/recusas, ok/negadas, bytes, latência
│   ├── agent/
│   │   ├── Agent.kf               connect/reconnect, pinger, sweeper,
│   │   │                         health, refresher, watchdog
│   │   ├── Backoff.kf             backoff exponencial + jitter
│   │   ├── Health.kf              health check TCP dos serviços locais
│   │   ├── Relay.kf               relay de um stream ↔ localhost
│   │   └── Metrics.kf             connects/reconnects/fatais/streams
│   ├── cli/
│   │   ├── Cli.kf                 dispatcher de subcomandos
│   │   ├── Commands.kf            login…serve (maioria dos comandos)
│   │   ├── Deploy.kf              deploy/down (manifesto, build, --check/--plan)
│   │   ├── Doctor.kf              doctor (diagnóstico em camadas)
│   │   ├── Daemon.kf              daemon install/status/uninstall (systemd)
│   │   ├── Local.kf               sessão/identidade/paths locais
│   │   └── Usage.kf               ajuda
│   └── util/
│       ├── Hosts.kf               hostnames + portas default do serve
│       ├── Limits.kf              limites/tetos configuráveis (fonte única)
│       └── Text.kf                parse seguro, tailLines, bearer, uso
└── tests/                         suites (mesmo módulo raiz)
```

## Regras estruturais

* **Duas raízes declaradas em `kof.toml` (`[sources] app="src" test="tests"`,
  `D-CLI-SOURCE-ROOTS` #708).** `kof test` compila a raiz de teste e resolve
  `import kofcloud.*` contra a raiz de app — sem cópias. (Medido: `kof test
  <dir>` posicional **não** resolve imports cross-root; a raiz de módulo passa
  a ser o diretório dado, e compilar `kofcloud/` como módulo standalone falha
  com funções indefinidas. Por isso o manifesto.)
* **Um `main` só** — em `Main.kf` (`PKG002` se houver outro).
* **`protocol/` não faz I/O.** É pura transformação de dados: testável sem
  sockets. O I/O vive em `edge/` e `agent/`.
* **Sem classes gigantes** — alvo ≤ 500 linhas por arquivo, como o repositório
  Kof exige.
* Módulos só existem se houver código real. A árvore acima é o resultado, não um
  alvo a preencher: `protocol/`, `model/`, `identity/`, `store/`, `control/`,
  `edge/`, `agent/`, `cli/`, `util/`.

## Dependências externas

Nenhuma. O KofCloud usa **somente** stdlib do Kof. `kofdeps` não é usado.
Bibliotecas candidatas no futuro: `libs/file` (TOML/YAML de config),
`libs/kofmd` (docs). Nenhuma é necessária hoje.
