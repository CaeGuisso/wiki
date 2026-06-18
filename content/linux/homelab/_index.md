---
title: Homelab
type: docs
---

De uma olhada no meu artigo para detalhes sobre os assuntos: [Proxmox & Homelab](https://guisso.dev/misc/proxmox-debian-day/)

## Infraestrutura Base

- **Processador**: AMD Ryzen 5 5600H
- **Memória**: 12GB RAM
- **Armazenamento**:
    - ~500GB NVMe (sistema)
    - ~1TB HDD para mídia (pool ZFS `media-zfs`, disco único, sem redundância)

Se você estiver interessado, pode encontrar esse mini PC no [AliExpress](https://pt.aliexpress.com/item/1005003443853901.html).

- **Hypervisor**: Proxmox VE 9
- **Método de Deploy**: [helper-scripts.com](https://helper-scripts.com) para automatizar a criação de containers LXC. Todos os guests são LXC, sem VMs QEMU.

![proxmox VE 8 screenshot](pve-2025-fev.png)

## Aplicações em Produção

### Gerenciamento e Rede
- **[AdGuard Home](https://community-scripts.github.io/ProxmoxVE/scripts?id=adguard)**: Servidor DNS com bloqueio de anúncios e rastreadores.

- **[Nginx Proxy Manager](https://community-scripts.github.io/ProxmoxVE/scripts?id=nginxproxymanager)**: Proxy reverso com SSL wildcard via DNS-01 Cloudflare. Roda em duas instâncias, uma interna (LAN) e uma dedicada a projetos externos.

- **[Tianji](https://community-scripts.github.io/ProxmoxVE/scripts?id=tianji)**: Analytics e monitoramento. Similar ao Google Analytics + Uptime Kuma.

- **[Pulse](https://community-scripts.github.io/ProxmoxVE/scripts?id=pulse)**: Dashboard de monitoramento do próprio Proxmox (recursos, containers, status do nó).

### Mídia e Downloads
- **[Jellyfin](https://community-scripts.github.io/ProxmoxVE/scripts?id=jellyfin)**: Servidor de mídia para streaming de filmes, séries e músicas.

- **[Transmission](https://community-scripts.github.io/ProxmoxVE/scripts?id=transmission)**: Cliente torrent.

- **[Prowlarr](https://community-scripts.github.io/ProxmoxVE/scripts?id=prowlarr)**: Gerenciador de indexers para a stack arr.

- **[Radarr](https://community-scripts.github.io/ProxmoxVE/scripts?id=radarr)**: Automação de downloads de filmes.

- **[Sonarr](https://community-scripts.github.io/ProxmoxVE/scripts?id=sonarr)**: Automação de downloads de séries.

- **[Bazarr](https://community-scripts.github.io/ProxmoxVE/scripts?id=bazarr)**: Download automático de legendas.

- **[Jellyseerr](https://community-scripts.github.io/ProxmoxVE/scripts?id=jellyseerr)**: Portal de requisições de mídia integrado ao Jellyfin.

- **[MeTube](https://community-scripts.github.io/ProxmoxVE/scripts?id=metube)**: Download de vídeos do YouTube e outras plataformas.

- **[Lidarr](https://community-scripts.github.io/ProxmoxVE/scripts?id=lidarr)**: Automação de downloads de música.

- **[Navidrome](https://community-scripts.github.io/ProxmoxVE/scripts?id=navidrome)**: Streaming de música pessoal compatível com Subsonic.

### Documentos e Pessoal
- **[Paperless-ngx](https://community-scripts.github.io/ProxmoxVE/scripts?id=paperless-ngx)**: Gerenciamento de documentos digitalizados. Ligado on-demand, só quando preciso indexar ou consultar documentos.

- **[Gramps Web](https://community-scripts.github.io/ProxmoxVE/scripts?id=gramps-web)**: Plataforma de genealogia e árvore genealógica da família.

### Páginas e Projetos Hospedados
- **[gapes](https://github.com/fguisso/gapes)**: Hospedagem self-hosted de páginas HTML estáticas.

- **[hi.events](https://hi.events/)**: Plataforma de gestão de eventos e venda de ingressos.

- **[Formbricks](https://formbricks.com/)**: Formulários e pesquisas self-hosted.

## Acesso Externo

- **VPN**: [**Unifi Teleport**](https://help.ui.com/hc/en-us/articles/4409740063511)
- **Internet**: [**Cloudflare Tunnels**](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

## Não usamos mais

- **[qBittorrent](https://community-scripts.github.io/ProxmoxVE/scripts?id=qbittorrent)**: Substituído pelo Transmission.

- **[Nextcloud](https://community-scripts.github.io/ProxmoxVE/scripts?id=nextcloudpi)**: Usado para fotos e arquivos. Desligado, o número de usuários (2) não justificou manter a complexidade. PhotoPrism cobria melhor o caso de fotos, e as demais features não foram relevantes no uso diário.

- **[PhotoPrism](https://community-scripts.github.io/ProxmoxVE/scripts?id=photoprism)**: Gerenciamento de fotos com IA. Parado, funcionou bem localmente mas teve problemas com o web player de vídeo quando exposto externamente. Em busca de alternativa.

- **[Zitadel](https://community-scripts.github.io/ProxmoxVE/scripts?id=zitadel)**: Provedor de identidade para SSO/OAuth2. Muito bom, usamos num micro SaaS, mas como o projeto não andou pra frente acabamos removendo.

- **[The Lounge](https://community-scripts.github.io/ProxmoxVE/scripts?id=thelounge)**: Cliente IRC web sempre conectado. Muito bom, mas com pouco uso de IRC no dia a dia acabou removido.

- **[SparkyFitness](https://community-scripts.github.io/ProxmoxVE/scripts?id=sparkyfitness)**: Acompanhamento de nutrição e exercícios self-hosted. App muito bom, mas a quantidade de informações e inputs tornou ele pouco prático no dia a dia, removido.

- **[Paperclip](https://community-scripts.github.io/ProxmoxVE/scripts?id=paperclip)**: Plataforma open-source para orquestrar times de agentes de IA como uma empresa (org chart, budgets, governança). Consumia muito token e a experiência foi péssima, removido.

- **[IronClaw](https://community-scripts.github.io/ProxmoxVE/scripts?id=ironclaw)**: Agent OS focado em segurança (Rust, sandbox WASM). Limitava o uso a IA só via API key ou ao SDK do Claude com custo extra, por isso foi removido. Estamos avaliando o [Hermes Agent](https://community-scripts.github.io/ProxmoxVE/scripts?id=hermesagent) com o [Codex](https://github.com/openai/codex) no lugar.

- **DevBox (Debian)**: Container Debian genérico para programar pelo celular ou iPad. Paramos de usar esse setup.
