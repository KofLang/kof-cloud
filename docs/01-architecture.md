# KofCloud — Arquitetura

O KofCloud conecta um serviço que roda na máquina do usuário à internet pública.
Ele **não** é um provedor de cloud: não fornece VM, não hospeda código, não armazena
dados. Ele fornece a camada de conexão — identidade, descoberta, túnel reverso,
DNS, HTTPS, roteamento e observabilidade mínima.

```
Internet
   │  HTTPS :443 (Edge)
   ▼
KofCloud Edge ──────────────► KofCloud Control Plane
   │  túnel reverso (TCP)          (contas, dispositivos,
   │  outbound da máquina           serviços, rotas, tokens)
   ▼
KofCloud Agent
   │  localhost:<port>
   ▼
Serviço do usuário
```

## Componentes

| Componente | Papel | Onde roda | Implementação |
|---|---|---|---|
| **Agent** | identidade do dispositivo, conexão, túnel, heartbeat, relay local | máquina do usuário | `kofcloud/agent/` |
| **Control Plane** | contas, dispositivos, serviços, domínios, rotas, autorização, estado | servidor KofCloud | `kofcloud/control/` |
| **Edge** | recebe tráfego da internet, roteia pelo túnel, HTTP/1.1 público | servidor KofCloud | `kofcloud/edge/` |
| **Protocol** | enquadramento, control protocol, tunnel protocol, multiplexação | ambos | `kofcloud/protocol/` |

## Decisões de arquitetura (medidas, não presumidas)

1. **A máquina inicia a conexão.** O Agent abre uma conexão TCP de saída para o
   Edge (`net.connect`). Funciona atrás de NAT, CGNAT e firewall. Nenhuma porta
   de entrada é necessária na máquina do usuário.

2. **Uma conexão, muitos streams.** Um único túnel carrega todos os serviços
   publicados, multiplexados por `streamId` (§`07-multiplexing.md`).

3. **Control Plane e Edge são planos separados.** O Control Protocol (JSON sobre
   HTTP) e o Tunnel Protocol (frames JSON delimitados por linha sobre TCP) são
   independentes. Trocar o transporte do control plane não afeta o túnel.

4. **Target: JVM.** O Kof só oferece, hoje, *todas* as primitivas necessárias
   (sockets TCP, `spawn`/`await`, canais, X25519/HKDF, Ed25519, AES-GCM,
   filesystem, processos) no target JVM. `native.risc`/`native.arm` não têm
   `kof.security` (`SECN000`); `js` recusa `kof.net` (`NETN001`). O núcleo é
   portável por construção, mas a validação é JVM-first — ver
   `12-kof-apis.md`.

5. **Canais, não locks.** A concorrência usa `spawn`/`await` e `channel<T>`.
   Kof não expõe mutex/atomic, e não precisa: cada stream é uma task que
   possui seu próprio socket e se comunica por canal.

## Limite real da linguagem que molda o desenho

`Conn` e `Listener` (handles de socket do `kof.net`) **não são classes reais
emitidas**: são nomes declaráveis. Medido em `kof-compiler`:

* ✅ funcionam como **variável local**, **parâmetro de função** e **captura de
  closure**;
* ❌ **quebram** como **campo de classe** e como **tipo de retorno de `spawn`**
  — o descritor emitido referencia `KofRuntime$NetConn` sem pacote e o runtime
  lança `NoClassDefFoundError`.

Consequência de projeto: **nenhuma classe guarda um `Conn`**. Uma sessão é uma
função que recebe o `Conn` por parâmetro e o mantém local; tasks filhas o
capturam por closure; o roteamento entre streams usa `Map<String, Channel<String>>`
(que é um tipo real e funciona como campo). Ver `12-kof-apis.md` §Lacunas.

## Fronteiras de confiança

* A máquina do usuário é tratada como **rede não confiável**.
* O Control Plane **não** recebe código nem dados da aplicação.
* O Edge roteia por hostname; o conteúdo trafega cifrado ponta-a-ponta entre
  Edge e Agent quando `KOFCLOUD_TUNNEL_SECRET` está configurado.
* Observabilidade registra metadados operacionais, **nunca** conteúdo de
  requisição, credenciais ou chaves.
