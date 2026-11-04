# Modelo de visibilidade (quem vê o quê)

Declarado em 2026-11-04 (Fase 15, slice 1 — **sem código, alinhamento**).
Responde uma pergunta antes de qualquer cifra nova: hoje, cada parte do
sistema vê o quê? O alvo da Fase 15 só faz sentido medido contra este
retrato. Nada aqui muda comportamento — o código citado já existe e já é
testado; a novidade é o inventário.

## O retrato atual

| Dado | Cliente | Edge | Túnel (fio) | Agent | Serviço local |
|---|---|---|---|---|---|
| Host solicitado | vê (ele escolhe) | vê (roteia por ele) | vê (decifrado pelo Edge para rotear) | vê (via `services[]`) | não (só recebe path + headers) |
| Corpo da requisição/resposta | vê | vê (decifra para repassar) | opaco **se** `KOFCLOUD_TUNNEL_SECRET` (AES-GCM); legível sem ele | vê (decifra para servir) | vê |
| Credencial `kfs_` | vê (ele envia) | vê (valida) e **remove** antes de repassar | opaco com cifra, legível sem | não vê (`dropCredential`) | não vê |
| Token da conta `kfc_` | vê | vê (auth do control plane) | n/a (só no control plane, HTTPS externo) | vê (sessão local) | não vê |
| Chave privada do device | não | não | nunca trafega | só a máquina dona | não |
| Hostname/métricas | vê o próprio | só agregados (sem cardinalidade por cliente) | n/a | só agregados | não |

Fontes no código: `edge/Relay.kf` (decifra com `session.sessionKey`),
`edge/Http.kf` (`dropCredential` remove `authorization:`/`x-kofcloud-token:`),
`protocol/Payload.kf` (AES-GCM com chave, base64 sem), `control/Registry.kf`
(tokens guardados só como hash), `edge/Metrics.kf` (sem dado de cliente).

## O que "E2EE real" significa aqui

**Conteúdo opaco para a plataforma**: o Edge continua vendo o Host (rotear
exige) e os metadados de operação (contadores, latência), mas o corpo viaja
cifrado de ponta a ponta (chave por serviço, distribuída pelo control plane
autenticado) e o Edge repassa bytes opacos sem decifrar. O operador da nuvem
— e quem comprometer o Edge — vê para onde vai, nunca **o quê**.

## O que NÃO é prometido

* **Host visível**: roteamento por `Host`/`SNI` exige o destino em claro.
  Com TLS na borda (Fase 9), o SNI vaza o hostname — mitigar com ECH quando
  o stdlib permitir, ou documentar como resíduo aceito.
* **WAF × E2EE**: inspecionar corpo exige ver corpo. Por serviço, o operador
  escolhe **um** dos dois — nunca os dois fingindo (ver `16-plan.md` Fase 15).
* **Metadados**: tamanho, tempo e destino continuam visíveis — E2EE esconde
  conteúdo, não tráfego.
