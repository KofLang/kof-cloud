# KofCloud — documentação

| # | Documento | Conteúdo |
|---|---|---|
| 01 | [Arquitetura](01-architecture.md) | componentes, decisões, fronteiras de confiança |
| 02 | [Identidade](02-identity.md) | Account, Device, Service, Token |
| 03 | [Dispositivo](03-device.md) | ciclo de vida, estado local, presença, revogação |
| 04 | [Serviço](04-service.md) | modelo, exposição, health check, roteamento |
| 05 | [Protocolo — visão geral](05-protocol-overview.md) | versionamento, enquadramento, erros |
| 06 | [Control Protocol](06-control-protocol.md) | endpoints HTTP/JSON |
| 07 | [Tunnel Protocol](07-tunnel-protocol.md) | frames, handshake, cifra |
| 08 | [Multiplexação](08-multiplexing.md) | streams, backpressure, limites |
| 09 | [Fluxos](09-flows.md) | auth, conexão, publicação, reconexão |
| 10 | [Edge](10-edge.md) | listener público, router, sessões, abuse prevention |
| 11 | [DNS](11-dns.md) | wildcard por conta, domínio próprio |
| 12 | [TLS](12-tls.md) | terminação, certificados, lacuna de ACME |
| 13 | [APIs Kof](13-kof-apis.md) | o que existe, o que falta (medido) |
| 14 | [Módulos](14-modules.md) | estrutura e regras |
| 15 | [Testes](15-tests.md) | matriz unit/integração/E2E/falha |
| 16 | [Plano](16-plan.md) | fases 1–8 |
| 17 | [Limitações](17-limitations.md) | lacunas honestas |
| 18 | [Implementado](18-implemented.md) | o que está pronto e verificado |
| — | [STATUS](STATUS.md) | estado medido: verde/prova/gaps + como reproduzir |

## Como rodar

```bash
# Toolchain: CLI Kof 0.6.3 + JDK 25 (ver 13-kof-apis.md e 15-tests.md)
export PATH=/home/mel/tools/jdk-25/bin:$PATH
KOF=/home/mel/Kof4j/kof-cli/target/kof-cli-0.6.3.jar
# da raiz do repositório (kof.toml declara app=src, test=testes)
java -jar $KOF build --output /tmp/kofcloud-check   # compila o app (alvo JVM)
java -jar $KOF test --target jvm --timeout 150      # 21 suites, 0 falhas
java -jar $KOF run src/Main.kf <comando>            # CLI
```

Ver o [README](../README.md) para os comandos da CLI.
