# Post-mortems de incidentes

Cada incidente relevante do KofCloud fecha com um post-mortem neste
diretório (`AAAA-MM-DD-titulo.md`). Sem ele, o ciclo da Fase 17 não
fechou (detectar → conter → comunicar → aprender): o game-day só conta
como prova se o aprendizado ficou registrado.

## Regras

* Um arquivo por incidente, nomeado pela data de abertura.
* Linguagem operacional: o que rodou (comando), o que respondeu (código),
  o que mudou (config). Nada de dado de cliente — a mesma regra que vale
  para log/métrica/audit vale aqui: sem hostname, corpo, credencial,
  `kfc_` ou `kfs_`. Cite o `kind` do incidente (ex. `freeze_service`,
  `edge_refusals`) e a severidade, nunca o dono.
* A ação preventiva tem dono e prazo. Sem dono, é desejo, não ação.
* O template abaixo é o contrato: um teste trava as 5 seções
  (`tests/incidents_test.kf`, `postmortem-template`).

## Template

```markdown
# Post-mortem: <título curto>

- Data: AAAA-MM-DD
- Incidente: <kind> (severidade <INFO|WARN|CRIT>)
- Detectado por: <sinal — ex. detector do anel de auditoria, Edge, operador>
- Contido com: <comando exato — ex. `kof cloud freeze service <nome>`>

## Causa

O que aconteceu, em termos de mecanismo (não de culpa).

## Impacto

O que o usuário viu (rota fora, 429, 503) e por quanto tempo.
Escopo: só plataforma (global) ou conta afetada — sem identificar.

## Timeline

- HH:MM — sinal abriu o incidente (`kind`, severidade).
- HH:MM — contenção aplicada (`kof cloud ...`).
- HH:MM — `incidents --resolve <id>` após verificação.

## Correção

O que foi feito para voltar ao normal (comando, config, revert).

## Ação preventiva

- [ ] <ação> — dono: <nome> — prazo: <AAAA-MM-DD>
```
