---
title: Claude Code no Homelab
type: docs
date: 2026-03-22
description: Usar Claude Code com CLAUDE.md, skills e documentação para gerenciar Proxmox, Unifi e serviços locais via API, sem vazar segredos
---

Rodar o Claude Code diretamente no shell do nó Proxmox, com um repositório de documentação como contexto, transforma o assistente em algo que entende o ambiente antes de agir.

O desafio: o assistente precisa falar com as APIs dos serviços (UniFi, NPM, AdGuard), mas tudo que ele digita, lê ou imprime vira transcript. Credenciais, IPs internos, nomes de vault, nada disso pode aparecer no comando nem no stdout. A solução é separar quem **decide** a chamada (o assistente) de quem **conhece o segredo** (um wrapper).

## Estrutura

```
/home/claude-mox/
├── CLAUDE.md              # contexto principal, carregado automaticamente
├── docs/
│   ├── inventory.md       # containers: VMID, IP, recursos, função
│   ├── network.md         # DNS, proxy, certificados, APIs
│   ├── storage.md         # ZFS pools, datasets, bind mounts
│   └── secrets.md         # arquitetura de segredos (op + wrapper)
├── scripts/
│   └── health.sh          # health check rápido do nó
├── best-practices/
│   └── checklist.md       # boas práticas Proxmox
├── alerts/
│   └── current.md         # problemas ativos, consultado antes de agir
└── .claude/
    └── commands/          # slash commands do projeto (skills)
        ├── pve.md
        ├── lxc.md
        ├── unifi.md
        ├── npm.md
        ├── adguard.md
        ├── net.md
        ├── add-service.md
        ├── remove-service.md
        └── zfs.md
```

## CLAUDE.md

Define o ambiente para o assistente. O Claude Code carrega automaticamente em cada sessão.

```markdown
## Ambiente
- Nó: pve, Proxmox VE 9.x
- IP do Host: <ip_do_host>

## CLIs Disponíveis
| Comando | Uso                                    |
|---------|----------------------------------------|
| pvesh   | API REST do Proxmox via CLI            |
| pct     | Gerenciar containers LXC               |
| zpool   | Gerenciar pools ZFS                    |
| hl      | Wrapper de API com auth via 1Password  |

## Fluxo de Trabalho
1. Verificar alerts/current.md antes de qualquer mudança
2. Consultar docs/inventory.md para entender impacto
3. Documentar resultados após mudanças
```

## Segredos: o assistente nunca vê a credencial

Esta é a parte central. O assistente roda como um processo que loga tudo, então a regra é simples: o segredo nunca entra no comando nem sai no stdout.

### Arquitetura

1. **Gerenciador de segredos** (1Password) como fonte da verdade. Cada credencial é um item referenciável por uma URI estável, no formato `op://<vault>/<item>/<campo>`.
2. **Service Account** com acesso somente ao vault do homelab. O único segredo em disco é o token do Service Account, em arquivo `chmod 600` (root), o bootstrap mínimo.
3. **Wrapper `hl`** (`/usr/local/bin/hl`): recebe o serviço mais a chamada, resolve a `op://` em runtime, injeta no `curl` e devolve só a resposta da API. A credencial existe apenas na memória do processo `curl`.

```
assistente  ──►  hl unifi GET /rest/user   (sem segredo no comando)
                   │
                   ├─ resolve op://<vault>/<item>/<campo>  (runtime)
                   └─ curl ... -H "X-API-KEY: ****"        (segredo só aqui)
                                  │
assistente  ◄──  resposta JSON da API        (sem segredo no stdout)
```

### Regras anti-vazamento

- Referenciar segredo por `op://` ou via wrapper, nunca pelo valor.
- Nunca `op read` numa chamada avulsa: o valor iria pro stdout/transcript. Só o wrapper lê.
- Nunca `cat`/`echo`/abrir o arquivo do token do Service Account.

## Skills (.claude/commands/)

Cada arquivo `.md` em `.claude/commands/` vira um slash command. Invocar com `/<nome>` executa toda a lógica sem re-explicar contexto. As skills não nasceram prontas: cada uma é o resultado de uma dor repetida que valia a pena padronizar. Abaixo, a origem de cada uma, mais do que o conteúdo.

### /pve

O nó só roda LXC, sem VMs QEMU. A skill nasceu da necessidade de listar e inspecionar recursos ao vivo (CPU, RAM, disco por container) sem decorar as flags de `pct` e `pvesh` toda vez. Em vez de confiar num inventário estático, ela sempre consulta o estado real do cluster.

### /lxc

Instalávamos novos containers manualmente com os community helper-scripts, copiando comandos de páginas que envelhecem. A skill passou a buscar o catálogo de scripts ao vivo no GitHub e a checar VMID livre e recursos do host antes de instalar, então nunca depende de uma lista fixa.

### /unifi

Queríamos não só consultar mas também escrever na rede via API. Descobrimos que a API key funciona nos endpoints REST clássicos sem sessão nem CSRF, usando update parcial (só os campos a alterar), o que evita reenviar segredos como passphrases. A skill documenta a diferença entre a API v1 (legacy) e a v2 (moderna) para não errar o path.

### /npm

Temos duas instâncias de Nginx Proxy Manager: uma interna (reverse proxy da LAN) e uma externa (atrás de túnel Cloudflare). A skill separa as duas com wrappers distintos e inclui um fallback: quando a API cai, o estado pode ser lido direto do sqlite via `pct exec`, útil para auditoria.

### /adguard

O AdGuard Home é o DNS da rede. Precisávamos consultar estatísticas (queries, taxa de bloqueio) e gerenciar rewrites de DNS ao vivo, sem abrir a interface web. A skill embute os endpoints e deixa a auth com o wrapper.

### /net

As três fontes (UniFi, NPM, AdGuard) divergiam na prática: proxy host apontando para IP sem reserva DHCP, rewrite de DNS órfão de um domínio que não existe mais, DNS do DHCP apontando para o lugar errado. A skill cruza os três ao vivo e reporta as inconsistências, em vez de cada um manter sua própria verdade.

### /add-service

Expor um serviço interno era um ritual repetitivo: reservar IP no UniFi, criar o proxy host no NPM, atualizar o inventário. A skill padronizou o pipeline e incluiu a convenção do subdomínio curto (3 a 4 letras), sempre confirmado com o usuário antes de aplicar. O wildcard de DNS já cobre os novos nomes, então o AdGuard não precisa mudar.

### /remove-service

Simétrica à `/add-service`. Ao destruir um container, proxy hosts, rewrites de DNS e reservas DHCP que apontavam para ele ficavam órfãos. A skill levanta a identidade do container (MAC, IP, hostname, mountpoints) antes de destruir, porque depois o `pct config` para de funcionar, e então varre NPM, UniFi e AdGuard limpando as referências.

### /zfs

O pool `media-zfs` é de disco único, sem redundância. Isso torna o monitoramento de SMART, scrub e erros de checksum (CKSUM) crítico, não opcional. A skill concentra os comandos de saúde do pool para detectar degradação cedo.

## APIs úteis (via wrapper)

Com o wrapper, o assistente faz chamadas legíveis e auditáveis, sem nunca manipular a credencial.

### Proxmox

```bash
pvesh get /cluster/resources --output-format json
pct exec <vmid> -- bash -c "comando"
```

### UniFi (REST)

```bash
# clientes ativos, o wrapper injeta a X-API-KEY
hl unifi GET /stat/sta

# reservas DHCP
hl unifi GET /rest/user

# configurar DNS do DHCP em uma rede
hl unifi PUT /rest/networkconf/<id> \
  -d '{"dhcpd_dns_enabled": true, "dhcpd_dns_1": "<ip_do_adguard>", "dhcpd_dns_2": "1.1.1.1"}'
```

### Nginx Proxy Manager (REST)

```bash
# listar proxy hosts, o wrapper resolve login e Bearer
hl npm GET /nginx/proxy-hosts

# criar proxy host com SSL
hl npm POST /nginx/proxy-hosts -d '{
    "domain_names": ["servico.interno.seudominio.com"],
    "forward_scheme": "http",
    "forward_host": "<ip_do_servico>",
    "forward_port": 8096,
    "certificate_id": 2,
    "ssl_forced": true,
    "allow_websocket_upgrade": true,
    "enabled": true,
    "meta": {}, "locations": [], "access_list_id": 0,
    "advanced_config": "", "http2_support": false,
    "hsts_enabled": false, "caching_enabled": false
  }'
```

### AdGuard Home (REST)

```bash
# adicionar rewrite DNS, o wrapper injeta o Basic auth
hl adguard POST /control/rewrite/add \
  -d '{"domain": "*.interno.seudominio.com", "answer": "<ip_do_npm>"}'
```

## HTTPS interno com wildcard

Para HTTPS nos domínios internos sem expor serviços individualmente:

1. Use um subdomínio real: `*.interno.seudominio.com`
2. No NPM, emitir certificado via DNS-01 com token Cloudflare (permissão `Zone → DNS → Edit`)
3. Nenhuma porta precisa estar aberta para a internet
4. Nos CT logs aparece apenas `*.interno.seudominio.com`

O token Cloudflare também vive no gerenciador de segredos. Ele é consumido pelo serviço de renovação de certificado dentro do container, não pelo assistente. A `op://` é a fonte da verdade para rotação; o deploy no container é à parte.

```bash
# emitir cert wildcard via NPM API (wrapper injeta o token Cloudflare na credencial DNS)
hl npm POST /nginx/certificates -d '{
    "provider": "letsencrypt",
    "nice_name": "*.interno.seudominio.com",
    "domain_names": ["*.interno.seudominio.com", "interno.seudominio.com"],
    "meta": {
      "dns_challenge": true,
      "dns_provider": "cloudflare",
      "propagation_seconds": 60
    }
  }'
```

Fluxo completo:

```
AdGuard: *.interno.seudominio.com → IP do NPM
NPM: cert wildcard + proxy host por serviço
Unifi DHCP: distribui AdGuard como DNS para a rede
```

Qualquer dispositivo que pegar DHCP resolve os domínios internos com HTTPS automaticamente.

## Por que isso importa

O ganho não é só conveniência. Ao separar **decisão** de **segredo**:

- O transcript do assistente é seguro para revisar, compartilhar ou auditar, porque não há credencial nele.
- Rotação é centralizada: troca no gerenciador de segredos, nada muda nas skills.
- O blast radius é o vault do homelab e nada além: o Service Account não enxerga o resto.
