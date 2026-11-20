# Estratégia de DNS

## MVP — subdomínio KofCloud

```
mel.kofcloud.io          → Edge
*.mel.kofcloud.io        → Edge        (site, api, files, grafana, git…)
```

Registros:

```
mel.kofcloud.io.            A      <edge-ipv4>
*.mel.kofcloud.io.          A      <edge-ipv4>
```

Um **wildcard por conta** resolve todos os serviços sem tocar em DNS a cada
`publish`. O Edge decide pelo header `Host`; o DNS não precisa saber quais
serviços existem. Isso mantém o DNS estável e o roteamento dinâmico.

O subdomínio da conta é criado quando o `handle` é reservado no control plane —
não há passo manual de DNS para o usuário.

## Fase 6 (slice 1) — domínio próprio ✅ implementado

```
api.example.com    CNAME   site.mel.kofcloud.io
```

Fluxo real (CLI):

```
kof cloud domain add api.example.com --service site
# → PENDING + instrucao: CNAME api.example.com -> site.mel.kofcloud.io
# (operador cria o CNAME no provedor dele)
kof cloud domain verify api.example.com
# → resolve via net.resolve, confere contra kofcloud.edgeHosts, vira VERIFIED
# → Edge roteia api.example.com para o servico
```

Detalhes honestos do slice:

* comparação **exata** contra `kofcloud.edgeHosts` (config, sem default —
  vazio = verificação sempre recusa até o operador configurar);
* NXDOMAIN/falha de rede vira `KCF124` indistinguível de "resolveu para outro
  lugar"; conta alheia recebe `404 KCF122` opaco **antes** de qualquer DNS;
* **resolver também para fora não verifica** (`409 KCF164`): se a resposta tem
  o Edge **e** um IPv4 de terceiro (registro A antigo, CDN/provedor no meio),
  o domínio fica PENDING — senão parte dos clientes cai fora da plataforma.
  CNAME-only (sem endereço próprio) continua aprovando. IPv6 não é julgado
  neste slice (o destino esperado é IPv4); remove o registro extra e rode
  `domain verify` de novo;
* PENDING nunca roteia (indistinguível de host inexistente: `404 KCF100`);
* remoção e poda (serviço/dispositivo) desativam a rota imediatamente;
* **sem TLS/SNI neste slice**: o hostname próprio serve HTTP pela borda; HTTPS
  depende de terminador externo (ver `12-tls.md`).

## Fase futura — certificado + automação

## Automação de DNS

Automatizar a criação do registro exige credencial da API do provedor de DNS do
usuário (Cloudflare, Route53, …). O wildcard por conta elimina a necessidade
para o caminho feliz.

Para o domínio próprio, a plataforma **não** acopla provedor no núcleo: fala o
protocolo aberto do mercado (Fase 18 slice 1):

* `domain add api.example.com --service site --dns-webhook <url>` — o
  operador hospeda um receptor que cria o CNAME (POST assinado opcional com
  `KOF_KOFCLOUD_DNS_WEBHOOK_TOKEN`);
* `domain add api.example.com --service site --dns-cmd '<cli do provedor>'` —
  o CLI do provedor roda via `sh -c` com `{hostname}`/`{target}`/`{service}`
  expandidos (ex.: `cloudflare-cli`, `aws route53 change-resource-record-sets`,
  `wrangler`). Sem SDK e sem lock-in; a credencial do provedor fica no
  ambiente do operador.

Nos dois casos `--dry-run` mostra o que seria feito sem executar, e falha do
provedor vira AVISO — o domínio fica PENDING e o caminho manual
(`domain verify`) continua válido.

## O que o usuário nunca precisa configurar (MVP)

* registro DNS do subdomínio — criado junto com a conta;
* wildcard — criado uma vez por conta;
* port forwarding, IP público, VPN — não existem, a conexão é outbound.

## Verificação

| Verificação | Como |
|---|---|
| wildcard resolve | `net.resolve("qualquer.mel.kofcloud.io")` não vazio |
| host desconhecido | Edge devolve `404` |
| domínio próprio aponta para o Edge | `net.resolve` contém o IP do Edge |
