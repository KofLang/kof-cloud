# Política de segurança do KofCloud

O que é suportado, o que não é, como reportar e o que esperar. Sem
promessa aspiracional: cada afirmação aponta para a prova executável.

## O que é suportado

| Área | Estado | Prova |
|---|---|---|
| Túnel cifrado (opcional, `KOFCLOUD_TUNNEL_SECRET`) | X25519 + HKDF por sessão, AES-GCM nos payloads | `tests/identity_test.kf` |
| `AUTHENTICATED` com credencial de serviço `kfs_` | emitida no control plane, validada no Edge antes do túnel, removida antes do serviço local; revogação por token, inventário sem valores, expiração opcional, rotação com overlap | `tests/service_token_test.kf`, `tests/rotate_test.kf` |
| Erros estáveis `KCFnnn` | nunca removidos nem renumerados; 404 opaco entre contas; `Hint:` sem segredo | `tests/errors_test.kf` |
| Rate limit por hostname (100 req/60 s) | `429 KCF107` + `Retry-After` antes de rotear | `tests/ratelimit_test.kf` |
| Suspensão reversível + auditoria | `SUSPENDED` (serviço/dispositivo), anel de 100 eventos só com metadados | `tests/abuse_test.kf` |
| Control plane HTTPS | `serve --tls-cert/--tls-key`; HSTS do runtime | `tests/control_test.kf` (`control-tls`) |
| `security.txt` | `/.well-known/security.txt` no Edge, antes de rate limit/rota/auth | `tests/security_test.kf` |

## O que NÃO é suportado (honesto)

* **Verificação de servidor no cliente da CLI.** `kof.http` é trust-all
  por construção — o HTTPS cifra, mas o CLI não autentica o servidor.
  Mitigação: rede confiável/loopback ou terminador TLS verificado.
  Ver `docs/12-tls.md`.
* **TLS no Edge público.** Terminação externa; ver `docs/12-tls.md`.
* **WAF profundo / E2EE / PQC / ACME automático.** Ver `docs/17-limitations.md`
  e o plano (`docs/16-plan.md` Fases 9–11, 15).

## Como reportar

* Canal: `security@kofcloud.local` (o mesmo do `security.txt` servido
  pelo Edge).
* Inclua: versão (`kof cloud --version`, Runtime Kof), comandos
  reproduzíveis, logs **sem credenciais** (`kfs_`/`kfc_`/chaves nunca
  viajam em reporte — a CLI nunca os imprime e os logs nunca os gravam).
* Não publique o reporte antes de correção ou acordo (avisa o impacto,
  depois corrige — precedente da Fase 17).

## SLA de resposta

* Confirmação de recebimento: **2 dias úteis**.
* Triagem com severidade (o que vaza × o que só derruba): **5 dias úteis**.
* Correção ou mitigação documentada: **30 dias** para severidade alta
  (vazamento de credencial, bypass de auth, RCE no serviço local via
  Edge); **90 dias** para o resto.
* Toda correção vira teste de regressão — se não tem teste novo, não
  fechou (precedente das Fases 9–19).

## Escopo

Vale para o código deste repositório (control plane + Edge + Agent +
CLI, alvo JVM). Fora do escopo: o runtime Kof (reporte no repositório
dele), o terminador TLS externo do operador, DNS do operador e o
sistema operacional da máquina do usuário.

## Contenção (kill-switch documentado)

Se uma credencial vazar ou um serviço for comprometido, o caminho rápido
é um comando só (Fase 17 slice 2 — congela, revoga e abre incidente):

```
kof cloud freeze service <nome>     # suspende + revoga kfs_ + incidente
kof cloud freeze device <nome>      # device: suspende + revoga de todos os servicos dele
kof cloud unfreeze service <nome>   # restaura EXATAMENTE o status anterior
```

Sem o `freeze` (ou para entender o que ele faz por baixo), o procedimento
manual usa os mesmos primitivos — tudo reversível por desenho (nada aqui
destrói configuração); a ordem importa: primeiro tira a rota, depois mata
a credencial, por último derruba o túnel.

```
# 1. Tira a rota (o Edge responde 404 opaco, como PRIVATE):
kof cloud service suspend <nome>

# 2. Mata a credencial vazada (achar pelo inventario, sem valores):
kof cloud service token-list <nome>
kof cloud service token-revoke <nome> <id-curto>

# 3. Se o device foi comprometido, recusa o tunel (fatal, sem reconexao):
kof cloud device suspend <nome>

# 4. Derruba o tunel local (o Edge limpa a rota via unbind):
kof cloud disconnect

# 5. Confere: rota fora, credencial morta, tunel caido:
kof cloud service status <nome>   # service ... SUSPENDED, fora da rota
kof cloud doctor                  # tunel OFFLINE confirmado pelo Edge
```

Reverter o manual é o inverso (`unsuspend` libera a rota; o túnel volta
no próximo `connect`; credencial revogada **não** volta — emita outra com
`service token`). Se um passo mudar de nome, a trava `security-policy`
quebra de propósito — kill-switch desatualizado é pior que nenhum.
