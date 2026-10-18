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

## Fase futura — domínio próprio

```
api.example.com    CNAME   api.mel.kofcloud.io
www.example.com    CNAME   www.mel.kofcloud.io
```

Fluxo:

1. usuário informa `example.com` no control plane;
2. o control plane devolve o registro a criar (CNAME ou `A`);
3. o usuário cria o registro no provedor dele;
4. o control plane **valida** (resolve o nome e confere que aponta para o Edge);
5. só então emite certificado e ativa a rota.

Validação usa `net.resolve(host)` (existe no Kof: devolve todos os endereços).

## Automação de DNS

Automatizar a criação do registro exige credencial da API do provedor de DNS do
usuário (Cloudflare, Route53, …). Isso **não** faz parte do MVP: o wildcard por
conta elimina a necessidade para o caminho feliz. A automação entra quando o
domínio próprio for suportado, e será um conector por provedor — não um
acoplamento no núcleo.

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
