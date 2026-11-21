# Controles mapeados (meta SOC2-tipo-II — retrato honesto)

A Fase 19 (slice 4) põe o relatório SOC2 como meta: **cada controle mapeado
1:1 com o que já existe no código**, com a prova executável ao lado. Este
documento é esse mapa — e, na coluna de baixo de cada item, o que o controle
**não** cobre. Sem "certificado", sem checklist aspiracional: se o mecanismo
não existe, o item diz isso.

Como ler: **mecanismo** = o arquivo/função que realiza o controle;
**prova** = o teste que falha se o mecanismo regredir; **não cobre** = o
limite medido (ver `docs/17-limitations.md`).

## CC6.1 — acesso lógico a recursos

| Aspecto | Mecanismo | Prova |
|---|---|---|
| toda leitura/escrita do control plane exige credencial | `X-KofCloud-Token` validado por `Registry.accountByToken` (tokens só como hash SHA-256) | `tests/control_test.kf` (`control plane`, 401 sem token) |
| escopo por conta | 404 **opaco** entre contas (existente-alheio ≡ inexistente), aplicado em cada handler do `control/ControlPlane.kf` | `tests/errors_test.kf` (404 opaco) |
| acesso público a um serviço exige credencial de serviço | `EdgeState.authorize` + `kfs_` (`AUTHENTICATED`); o Edge **remove** a credencial antes do relay | `tests/service_token_test.kf`, `tests/e2e_authenticated_test.kf` |
| credencial de serviço não autoriza outro serviço | `serviceByServiceToken` → compara o id do dono | `tests/service_token_test.kf` |

**Não cobre:** sem IP de origem no control plane (`web.app()`), o rate limit
de login é por handle, não por IP (`docs/17-limitations.md`); a credencial da
conta (`kfc_`) continua válida em qualquer dispositivo da conta até expirar
ou ser revogada.

## CC6.2 — registro/remoção de usuários e dispositivos

| Aspecto | Mecanismo | Prova |
|---|---|---|
| ciclo de vida do dispositivo (ACTIVE/SUSPENDED/REVOKED) | `model/Device.kf` + `Registry.setDeviceStatus`; REVOKED é terminal (unsuspend/suspend → 409) | `tests/freeze_test.kf`, `tests/abuse_test.kf` |
| re-registro de um dispositivo revogado cria um device **novo** (sem ressuscitar o antigo) | `Registry.activeDeviceByName` | `tests/registry_test.kf` |
| teto de dispositivos ativos por conta | `deviceLimitFrom`, default 5, recuo seguro | `tests/limits_test.kf` |
| revogação derruba a sessão e remove as rotas | `EdgeState.unbind` + recusa `DEVICE_REVOKED` (fatal no Agent) | `tests/abuse_test.kf`, `tests/e2e_test.kf` |

**Não cobre:** não há MFA nem aprovação de dois operadores para revogação;
um único `kfc_` de conta executa o ciclo.

## CC6.3 — autorização e segregação

| Aspecto | Mecanismo | Prova |
|---|---|---|
| owner-gate (só a conta dona muda a status do serviço) | `handleReportStatus` no `control/ControlPlane.kf` | `tests/errors_test.kf` (owner-gate) |
| allowlist de métodos/caminhos por serviço, fail-closed só para quem declarou | `wafAllowCheck` (Edge, antes do relay; `403 KCF111`) | `tests/waf_allow_test.kf` |
| teto de corpo por serviço (limite por rota) | `serviceMaxBodyCap` + `maxBodyBytes` no `Service` | `tests/route_cap_test.kf` |

**Não cobre:** não há papéis/perfis — a conta é a unidade de autorização;
quem tem o `kfc_` tem a conta inteira.

## A1.2 / CC6.6 — auditoria, monitoramento e detecção

| Aspecto | Mecanismo | Prova |
|---|---|---|
| toda mutação sensível vira evento de audit com vocabulário fechado | construtores de `model/Audit.kf`; a trava quebra se um literal cru escapar | `tests/audit_chain_test.kf` (`chain-unit`) |
| integridade detectável por terceiro | hash encadeado **por conta** + `/v1/audit/verify` + espelho append-only `<estado>.worm.jsonl` com teto | `tests/audit_chain_test.kf` (`worm-unit`, `worm`, `worm-privacy`) |
| nenhum segredo no anel, no espelho nem no fio | `worm-privacy` (emissão/revogação reais de `kfs_`) | `tests/audit_chain_test.kf` (`worm-privacy`) |
| detecção de abuso por sinal agregado (túnel caindo, 401 em rajada, ociosidade, revoke/suspensão em lote, tempo local) | `RateProbe` no `EdgeState` + `Registry.detectBulk*` + varredura periódica (`credentialExpirySweeper`) | `tests/incidents_test.kf`, `tests/idle_test.kf` |
| detecção nunca carrega dado de cliente | incidentes sem hostname/token/device no resumo | `tests/incidents_test.kf` |

**Não cobre:** a trilha é **detectável, não imutável** (quem tem o disco pode
reescrever a cadeia inteira; o espelho não é WORM real — `docs/17-limitations.md`).
Sem Syslog/OTLP externo (`13-kof-apis.md`).

**Prova:** `tests/audit_chain_test.kf` (`chain-unit`, `chain`, `chain-filter`,
`worm-unit`, `worm`, `worm-privacy`).

## A1.3 — disponibilidade e recuperação

| Aspecto | Mecanismo | Prova |
|---|---|---|
| carga do estado explícita (ok/absent/corrupt) e nunca vazio em silêncio | `Registry.loadStatus` + `looksLikeControlState` | `tests/migrate_test.kf` |
| readiness honesta | `/v1/health` (503 com estado corrupto) + `serve` fail-closed | `tests/readiness_test.kf` |
| falha de bind não deixa metade no ar | pré-bind das três portas no `serve` | `tests/serve_bind_test.kf` |
| Edge derruba sessão ociosa e reconecta com backoff | sweeper + jitter (`agent/Backoff.kf`) | `tests/idle_test.kf`, `tests/backoff_test.kf` |
| backup do estado antes de troca de backend | `kof cloud state-backup` | `tests/migrate_test.kf` |

**Não cobre:** um nó de control plane e um de Edge (ponto único de falha); sem
réplica, sem consenso, sem failover (Fase 16). RTO/RPO não são prometidos.

## CC7.3 / CC7.4 — resposta a incidentes

| Aspecto | Mecanismo | Prova |
|---|---|---|
| contenção em um comando, reversível | `freeze service|device|account` + `unfreeze` (marcas por alvo, persistidas com teto) | `tests/freeze_test.kf` |
| contenção não tranca o operador fora | `kfc_` sobrevive ao freeze da conta | `tests/freeze_test.kf` |
| comunicação do incidente | status page sem login (só globais) + webhook HMAC-SHA256 | `tests/status_page_test.kf`, `tests/webhook_test.kf` |
| rastro e post-mortem | `webhook.set/removed` na trilha + template em `docs/incidents/README.md` | `tests/webhook_test.kf` (`webhook-audit`), `tests/incidents_test.kf` (`postmortem-template`) |

**Não cobre:** alvo `edge` (exige coordenação multi-Edge); webhook assinado
com HMAC (par Ed25519 não é exposto pelo stdlib — SECN014); game-day é ritmo
operacional, não código.

## CC7.2 — mudança e segredos

| Aspecto | Mecanismo | Prova |
|---|---|---|
| credencial de serviço/conta com ciclo de vida e overlap (sem downtime) | `tokenExpiryAt` (Long), `service rotate`, `token rotate`; sem TTL nada é revogado em silêncio | `tests/rotate_test.kf`, `tests/account_token_test.kf` |
| higiene de credencial sem prazo visível | `/v1/services/token-hygiene` + `service tokens` + `doctor` | `tests/token_hygiene_test.kf`, `tests/doctor_test.kf` |
| segredos de fonte externa lidos na partida, nunca impressos nem gravados | `readSecretSource` (`--tunnel-secret-cmd`, `--tls-*-cmd`) | `tests/secrets_test.kf` |
| varredura de segredo em manifesto/ambiente/PEM | `kof cloud scan` | `tests/deploy_test.kf` (`deploy-scan`), `tests/scan_extra_test.kf` |

**Não cobre:** rotação programada de PEM (é manual, por reinício); scan de
dependências exige rede/`kofdeps` (fora).

## CC8.1 — mudança controlada (CI)

| Aspecto | Mecanismo | Prova |
|---|---|---|
| deploy declarativo idempotente, `--plan`/`--check` antes de aplicar | `cli/Deploy.kf`; `--plan` marca `(build)`/`(artifact)`/drift | `tests/deploy_test.kf` |
| falha de build/artefato não publica aquele serviço | fail-fast por serviço | `tests/deploy_test.kf` (`deploy-build`) |

**Não cobre:** sem artefato transferível nem rollback automático (o deploy
diagnostica, não reverte — `docs/17-limitations.md`).

## Como verificar este mapa

O teste `docs-controls` em `tests/controls_doc_test.kf` lê este arquivo e
quebra se (a) uma citação `tests/<arquivo>.kf` apontar para arquivo
inexistente, (b) alguma seção `## ` não trouxer a linha **Não cobre** (o
limite), ou (c) a lista de controles encolher sem regressão de propósito.
