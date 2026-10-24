# Multiplexação

Uma conexão de túnel carrega **N streams** independentes — um por requisição
pública em andamento.

```
                 KofCloud Tunnel (1 conexão TCP)
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    stream 1          stream 2          stream 3
   site:8080          api:3000          files:9000
```

## Regras

* `streamId` é um Int ≥ 1, único dentro da sessão, alocado por quem abre
  (`OPEN` no Edge).
* Um stream carrega um fluxo de bytes bidirecional: `OPEN` → `DATA`* →
  (`END` | `RESET`).
* **Nenhum stream monopoliza a conexão.** O Edge aplica *round-robin* sobre os
  streams prontos e limita a profundidade da fila de saída por stream
  (`limits.queueDepth`, default 64 frames). Fila cheia → o stream recebe
  `RESET{LIMIT_EXCEEDED}` em vez de bloquear a conexão inteira.
* `DATA` é limitado a `limits.frameBytes` (1 MiB). Payload maior é fatiado
  em blocos de `limits.sendChunkBytes` (16 KiB) já codificados — um frame por
  bloco; o receptor acumula até `END`. O bloco de envio é bem menor que o
  teto de propósito: chega mais cedo, usa menos memória e evita o custo
  O(n²) de `bytesToString` em blocos grandes (ver `17-limitations.md`).
  O teto continua valendo na recepção: linha acima de
  `maxTunnelLineBytes()` (= 2×frameBytes + margem, cobrindo a expansão
  UTF-8 do base64) derruba a conexão (fail-closed).

## Backpressure e cancelamento

O modelo do Kof não tem locks; usa canais. Por stream:

```
Edge (por stream)                       Agent (por stream)
  inbox:  Channel<String>   ◄── DATA    inbox:  Channel<String>
  outbox: Channel<String>   ──► DATA    outbox: Channel<String>
```

* Cada stream tem **duas** filas com profundidade limitada.
* `RESET` e `END` são enfileirados como frames normais e chegam na ordem.
* Cancelar = enviar `RESET{s, "CANCELED"}` e descartar a fila daquele stream.
* O túnel como um todo só fecha quando a conexão TCP cai ou chega `GOAWAY`.

## Roteamento no Agent

O Agent mantém, por sessão:

| Estrutura | Tipo | Uso |
|---|---|---|
| `inboxes` | `Map<String, Channel<String>>` | streamId → fila de entrada |
| `outbox` | `Channel<String>` | fila única de saída (drenada pela task writer) |

O leitor do túnel lê frames, e para cada `DATA{s,...}` faz
`inboxes.get(s).send(...)`. A task do stream consome sua inbox, escreve no
socket local, e empurra a resposta na `outbox` compartilhada. Como o `Conn` do
túnel não pode ser campo de classe, ele vive como **parâmetro/captura** da task
writer — ver `01-architecture.md` e `12-kof-apis.md`.

## Limites

| Limite | Default | Aplicado? |
|---|---|---|
| streams simultâneos por túnel | 128 (`kofcloud.limits.streams`) | ✅ — `openStream` recusa; o Edge devolve `429`, o Agent responde `OPENED{ok:false}` + `RESET LIMIT_EXCEEDED` |
| profundidade de fila por stream | 64 | ⏳ |
| bytes por frame | 1 MiB | ✅ — envio em blocos de 16 KiB (`sendChunkBytes`), recepção recusa acima de `maxTunnelLineBytes` (`frames_test.kf`) |
| bloco de envio | 16 KiB | ✅ (`kofcloud.limits.sendChunkBytes`) |
| linha maxima do tunel | 2 MiB + margem | ✅ — `LineCodec` trava em overflow e o leitor derruba a conexao |
| idle timeout do stream | 30 s | ✅ — stream preso leva `RESET IDLE_TIMEOUT` (`tunnelIdleSweeper`/`agentIdleSweeper`, `idle_test.kf`) |
| idle timeout do tunel | 120 s | ✅ — sem frame util, a sessao cai e o Agent reconecta (`kofcloud.limits.tunnelIdleMs`, PING nao renova) |

O limite de streams é por sessão: fechar um stream libera a vaga imediatamente
(ver `tests/limits_test.kf`).
