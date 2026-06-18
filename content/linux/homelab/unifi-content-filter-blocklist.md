---
title: Filtro de Conteudo da UniFi via API
type: docs
date: 2026-03-23
description: Como aplicar uma blocklist de DNS no filtro de conteudo da UniFi usando a API
---

{{< callout type="warning" >}}
**Atenção:** eu não gostei de gerenciar os blocks de DNS desta forma, ficou muito trabalhoso. O AdGuard resolve esse ponto de uma forma muito melhor, então voltei meu DNS interno para o AdGuard e hoje prefiro que ele cuide desses blocks. Deixo este artigo como referência de como usar a API do filtro de conteúdo da UniFi, mas não é mais o caminho que eu uso.
{{< /callout >}}

O UniFi OS 4.x introduziu um recurso de filtro de conteúdo (menu Cybersecure) que suporta listas de bloqueio de domínios customizadas por rede. O endpoint da API não é documentado oficialmente, mas funciona com a API key padrão.

## Requisitos

- UniFi OS 4.x (testado no UniFi Express 7, firmware 4.4.x)
- API key com acesso à rede (`Settings → Control Plane → API Keys`)
- O ID do perfil de filtro de conteúdo (veja abaixo)

## Descobrindo o ID do seu perfil de filtro de conteúdo

A forma mais fácil é abrir a interface da UniFi, ir em **Cybersecure → Content Filtering**, ativar uma regra e interceptar a requisição no devtools do navegador (aba Network). Procure por uma requisição PUT para:

```
/proxy/network/v2/api/site/default/content-filtering/<profile-id>
```

Copie o ID do perfil a partir da URL.

Como alternativa, você pode tentar um GET na coleção, embora esse endpoint possa exigir autenticação por cookie dependendo da versão do firmware:

```bash
curl -sk https://<unifi-ip>/proxy/network/v2/api/site/default/content-filtering \
  -H "X-API-KEY: <sua-api-key>"
```

## Estrutura do payload

```json
{
  "_id": "<profile-id>",
  "enabled": true,
  "name": "Default",
  "categories": ["ADVERTISEMENT"],
  "client_macs": [],
  "network_ids": ["<network-id>"],
  "allow_list": [],
  "block_list": ["example.com", "ads.example.com"],
  "safe_search": [],
  "schedule": { "mode": "ALWAYS" }
}
```

**Campos:**
- `categories`: categorias nativas do DNS Shield (ex: `ADVERTISEMENT`, `ADULT`, `GAMBLING`, `MALWARE`)
- `block_list`: array de domínios a bloquear (customizado, aplicado por cima das categorias)
- `allow_list`: domínios sempre permitidos, sobrepondo os bloqueios
- `network_ids`: a quais redes este perfil se aplica
- `client_macs`: limitar a dispositivos específicos (vazio = todos os dispositivos da rede)
- `schedule.mode`: `ALWAYS` ou baseado em horário

## Baixando e processando uma blocklist

Este exemplo usa a [lista de Ad/Tracker do Peter Lowe](https://pgl.yoyo.org/adservers/) (~3,5 mil domínios, formato hosts):

```bash
curl -s "https://pgl.yoyo.org/adservers/serverlist.php?hostformat=hosts&showintro=0&mimetype=plaintext" \
  -o /tmp/blocklist.txt
```

Processe para extrair os domínios puros:

```python
domains = []
with open('/tmp/blocklist.txt') as f:
    for line in f:
        line = line.strip()
        if line.startswith('#') or not line:
            continue
        # formato: "127.0.0.1 domain.com"
        parts = line.split()
        if len(parts) >= 2 and parts[1] != 'localhost':
            domains.append(parts[1])

print(f"Processados {len(domains)} dominios")
```

Outras listas em formato hosts que funcionam com o mesmo parser:
- [Hosts unificado do Steven Black](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts)
- [Hosts do Dan Pollock](https://someonewhocares.org/hosts/hosts)

## Aplicando na UniFi

```python
import json
import subprocess

UNIFI_IP = "<unifi-ip>"
API_KEY = "<sua-api-key>"
PROFILE_ID = "<seu-profile-id>"
NETWORK_ID = "<seu-network-id>"

payload = {
    "_id": PROFILE_ID,
    "enabled": True,
    "name": "Default",
    "categories": ["ADVERTISEMENT"],
    "client_macs": [],
    "network_ids": [NETWORK_ID],
    "allow_list": [],
    "block_list": domains,
    "safe_search": [],
    "schedule": {"mode": "ALWAYS"}
}

with open('/tmp/payload.json', 'w') as f:
    json.dump(payload, f)

result = subprocess.run([
    "curl", "-sk", "-X", "PUT",
    f"https://{UNIFI_IP}/proxy/network/v2/api/site/default/content-filtering/{PROFILE_ID}",
    "-H", "content-type: application/json",
    "-H", f"X-API-KEY: {API_KEY}",
    "-d", "@/tmp/payload.json"
], capture_output=True, text=True)

data = json.loads(result.stdout)
print(f"Aplicados {len(data.get('block_list', []))} dominios")
```

Ou como one-liner em bash:

```bash
UNIFI_IP="<unifi-ip>"
API_KEY="<sua-api-key>"
PROFILE_ID="<seu-profile-id>"

curl -sk -X PUT "https://$UNIFI_IP/proxy/network/v2/api/site/default/content-filtering/$PROFILE_ID" \
  -H "content-type: application/json" \
  -H "X-API-KEY: $API_KEY" \
  -d @/tmp/payload.json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Aplicados:', len(d.get('block_list',[])))"
```

## Notas

- A API aceita a lista inteira em um único PUT, sem necessidade de paginação
- Testado com ~3.500 domínios (~70KB de payload), funcionou sem problemas
- Listas maiores (50 mil+) podem esbarrar em limites de tamanho do payload, teste antes de assumir que funcionam
- O filtro de conteúdo opera no nível da rede, antes de resolvers de DNS como o AdGuard
- A UniFi não expõe logs nem estatísticas dos bloqueios do filtro de conteúdo via API key, apenas pela interface ou por sessões autenticadas no navegador
- As `categories` nativas e a `block_list` customizada são aditivas, ambas se aplicam simultaneamente
