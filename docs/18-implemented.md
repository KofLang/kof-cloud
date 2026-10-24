# O que está implementado (e verificado)

Este documento descreve o estado **real** do código, não o plano. Tudo listado
como verificado foi medido com o CLI real e/ou com a suíte de testes.

## Verificado ponta a ponta

O critério de sucesso do pedido (§44) foi atingido em processos separados:

```
kof run src/Main.kf serve --control-port 28140 --tunnel-port 27060 --http-port 28141
kof run src/Main.kf login mel --password segredo --control http://127.0.0.1:28140
kof run src/Main.kf publish site --port 29010
kof run src/Main.kf connect

curl -H "Host: site.mel.kofcloud.io" http://127.0.0.1:28141/
→ hello from localhost
```

Sem port forwarding, sem IP público, sem VPN, sem reverse proxy manual. Host
desconhecido devolve `404`; serviço publicado mas sem túnel devolve `503`.

## Componentes implementados

| Módulo | Conteúdo | Teste |
|---|---|---|
| `protocol/` | enquadramento por linha, frame JSON, tunnel protocol, sessão multiplexada | `protocol_test.kf` |
| `identity/` | par X25519, HKDF-SHA256, AES-GCM, tokens opacos com hash | `identity_test.kf` |
| `model/` + `util/` | Account/Device/Service/Exposure, hostnames, parse de texto | `hosts_test.kf` |
| `store/` | persistência JSON | `registry_test.kf` |
| `control/` | registry + HTTP `/v1` (contas, devices, serviços, domínios, connect, heartbeat) | `control_test.kf` |
| `edge/` | listener de túnel, router por `Host`, HTTP front, relay, parada limpa | `e2e_test.kf` |
| `agent/` | connect, reconexão com backoff, heartbeat, health, relay, parada | `e2e_test.kf`, `backoff_test.kf` |
| `cli/` | dispatcher + comandos (`login`, `connect`, `publish`, …) | aceitação manual |
| limites | `util/Limits.kf` + enforcement no registry/sessão | `limits_test.kf` |
| múltiplos serviços | roteamento por `Host` no mesmo túnel | `e2e_multiservice_test.kf` |
| credencial de serviço | `kfs_` emitido no control plane, validado no Edge | `service_token_test.kf`, `e2e_authenticated_test.kf` |
| erros estáveis (R6) | `model/Errors.kf`: `KCFnnn` no control plane e no Edge + `Hint:` na CLI | `errors_test.kf` |

## Comportamentos provados

* **Túnel reverso persistente** iniciado pela máquina; sobrevive atrás de NAT
  (conexão de saída).
* **Multiplexação**: vários streams (requisições) sobre uma conexão, com filas
  por stream (canais), sem um serviço monopolizar os outros.
* **Vários serviços no mesmo túnel (Fase 4)**: um Agent serve N hostnames em N
  portas locais; cada host chega na sua porta e derrubar um serviço local não
  afeta os outros (`e2e_multiservice_test.kf`).
* **Limites aplicados**: dispositivos ativos por conta, serviços por
  dispositivo e streams simultâneos por túnel — 429 no control plane e
  429/`RESET LIMIT_EXCEEDED` no Edge (`limits_test.kf`).
* **Reconexão com backoff exponencial + jitter** (observável no log do agente
  quando o Edge não está no ar).
* **Health check**: o agente reavalia os serviços periodicamente e reporta só na
  mudança; o Edge para de encaminhar quando fica `OFFLINE` (`503`).
* **Cifra do túnel** (opcional, `KOFCLOUD_TUNNEL_SECRET`): X25519 efêmero +
  HKDF por sessão + AES-GCM nos payloads — dois lados derivam a mesma chave
  (testado).
* **Exposição**: `PRIVATE` não gera rota; `AUTHENTICATED` exige credencial de
  serviço `kfs_` **antes** do túnel (`e2e_authenticated_test.kf`), com o token
  da conta recusado e a credencial removida antes de chegar ao serviço local.
* **Revogação de dispositivo** derruba a sessão e remove as rotas dele.
* **Erros estáveis (R6, padrão kof-share)**: todo erro viaja como `KCFnnn`
  (`KCF001` handle inválido … `KCF106` streams esgotados); recurso de outra
  conta responde o mesmo `404` opaco de inexistente; `POST /v1/services/status`
  tem owner-gate (só a conta dona altera); a CLI imprime `Hint:` humana sem
  vazar segredos (`errors_test.kf`).
* **Parada limpa** do agente (`disconnect` via arquivo-sentinela) e da
  plataforma (`serve`, via fechamento do `Listener`).

## Não implementado (honesto)

* Terminação **TLS** no processo Kof e **ACME** — o Kof não expõe TLS em socket
  cru; ver `12-tls.md`. O hostname/roteamento por `Host` está pronto, mas o
  HTTPS depende de um terminador (`web.app().listenSecure`, JVM) ou externo.
* **DNS** como serviço — o wildcard por conta é configuração externa de DNS
  (ver `11-dns.md`); o KofCloud já emite o hostname correto.
* **WebSocket** público, **HTTP/2/3**, **domínio próprio automatizado**,
  **deploy** (`kof cloud deploy`), **daemon** (systemd/launchd), **painel web**.
* **Múltiplos nós** de control plane/edge (sem consenso; registro único em
  memória + arquivo).

Ver `16-plan.md` para as fases e `17-limitations.md` para as lacunas.
