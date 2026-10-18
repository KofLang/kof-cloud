# Estrutura dos módulos Kof

```
kof-cloud/
├── kof.toml                       manifesto: [sources] app="src", test="tests"
├── src/
│   ├── Main.kf                    entrada da CLI: `kof cloud …`
│   └── kofcloud/
│   ├── protocol/                  enquadramento e protocolos (sem I/O)
│   │   ├── Version.kf             PROTOCOL_VERSION
│   │   ├── Bytes.kf               base64, concatenação, limites
│   │   ├── Frame.kf               frame JSON por linha
│   │   ├── LineCodec.kf           buffer + split por \n
│   │   ├── Control.kf             mensagens do Control Protocol
│   │   └── Tunnel.kf              mensagens do Tunnel Protocol
│   ├── model/                     tipos de domínio (records)
│   │   ├── Account.kf
│   │   ├── Device.kf
│   │   ├── Service.kf
│   │   └── Exposure.kf
│   ├── identity/
│   │   ├── Keys.kf                par X25519, handshake, session key
│   │   └── Tokens.kf              geração/verificação de token
│   ├── store/
│   │   └── Store.kf               persistência JSON (control plane + agent)
│   ├── control/
│   │   ├── Registry.kf            estado em memória (contas, devices, serviços)
│   │   └── ControlPlane.kf        HTTP `/v1/…` sobre `web.app()`
│   ├── edge/
│   │   ├── Router.kf              Host → serviço
│   │   ├── Streams.kf             streamId, filas, ciclo de vida
│   │   ├── TunnelSession.kf       handshake + leitura/escrita de frames
│   │   ├── TunnelListener.kf      aceita Agents
│   │   └── HttpFront.kf           listener HTTP/1.1 público
│   ├── agent/
│   │   ├── Agent.kf               connect / reconnect / loop de frames
│   │   ├── Relay.kf               relay de um stream ↔ localhost
│   │   └── Health.kf              health check dos serviços
│   ├── cli/
│   │   ├── Cli.kf                 dispatcher de subcomandos
│   │   ├── Command.kf             interface
│   │   ├── Usage.kf               ajuda
│   │   └── commands/              um arquivo por subcomando
│   └── util/
│       ├── Hosts.kf               parsing/validação de hostname
│       └── Text.kf                helpers de string
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
