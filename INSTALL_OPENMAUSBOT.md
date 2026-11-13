# 📦 Guia de Instalação — OpenMausBot + OmniRoute (NVIDIA)

Este documento descreve passo a passo como instalar e configurar o **OpenMausBot** com o **OmniRoute** como provedor principal de modelos (incluindo os modelos NVIDIA NIM), baseado na instalação realizada neste ambiente.

---

## 1. Pré-requisitos

| Item | Versão mínima | Como verificar |
|------|---------------|----------------|
| Node.js | 24+ | `node --version` |
| pnpm | 10+ | `pnpm --version` |
| Git | qualquer | `git --version` |
| Docker (opcional, para OmniRoute) | qualquer | `docker --version` |

> Este guia foi testado em **Ubuntu 26.04 x86_64** com Node v24.22.3 e pnpm 12.9.1.

---

## 2. Clonar o repositório

```sh
git clone https://github.com/dockeraj-beep/OpenMausBot.git
cd OpenMausBot
```

---

## 3. Instalar dependências

```sh
pnpm install
```

> Durante a instalação, alguns scripts de build (`core-js`, `workerd`) podem ser ignorados. Isso é esperado. Você pode rodar `pnpm approve-builds` se precisar deles.

---

## 4. Configurar o OmniRoute (gateway de modelos NVIDIA)

O OmniRoute atua como gateway entre o OpenMausBot e os provedores de modelos (NVIDIA NIM, OpenRouter, etc.).

### 4.1. Instalar o OmniRoute

```sh
npm install -g omniroute
```

Ou use a instalação via Docker (recomendado para produção):

```sh
docker run -d \
  --name omniroute \
  -p 20128:20128 \
  -v $(pwd)/data:/app/data \
  diegosouzapw/omniroute:latest
```

### 4.2. Configurar senha de administrador

```sh
omniroute setup --non-interactive --password "SUA_SENHA_ADMIN"
```

### 4.3. Criar uma chave de API para o OpenMausBot

Acesse `http://127.0.0.1:20128/dashboard/api-manager` ou crie via banco de dados:

```sh
# Exemplo direto no SQLite (ajuste o caminho conforme sua instalação)
sqlite3 /caminho/para/storage.sqlite "INSERT INTO api_keys ..."
```

Ou gere uma chave e configure manualmente. Neste ambiente, a chave gerada foi:

```
sk-omb--dvGnfNGuK5vWUCuziYWQUnQvJYcJHTz
```

### 4.4. Adicionar o provedor NVIDIA

```sh
omniroute providers add nvidia \
  --credential "SUA_CHAVE_NVIDIA_NIM_AQUI" \
  --default-model "nemotron"
```

> Substitua `SUA_CHAVE_NVIDIA_NIM_AQUI` pela sua chave real da NVIDIA NIM (ex: `nvapi-...`).

### 4.5. Verificar os modelos disponíveis

```sh
curl -s -H "Authorization: Bearer SUA_CHAVE_OMNIROUTE" http://127.0.0.1:20128/v1/models
```

Modelos NVIDIA encontrados neste ambiente:

| Modelo | ID do modelo |
|--------|-------------|
| Nemotron 3.5 Super VL 120B | `nvidia/nvidia/nemotron-3.5-super-vl-120b-a12b` |
| Nemotron 3 Ultra 550B | `nvidia/nvidia/nemotron-3-ultra-550b-a55b` |
| Nemotron 3.5 Lightning 30B | `nvidia/nvidia/nemotron-3.5-lightning-30b-a3b` |

---

## 5. Configurar o OpenMausBot para usar o OmniRoute

### 5.1. Criar o diretório de configuração

```sh
mkdir -p ~/.openmausbot
chmod 700 ~/.openmausbot
```

### 5.2. Criar o arquivo `config.json`

Crie o arquivo `~/.openmausbot/config.json` com o seguinte conteúdo:

```json
{
  "instances": {
    "omniroute": {
      "driver": "openai-compat",
      "displayName": "OmniRoute (NVIDIA)",
      "icon": { "kind": "preset", "preset": "openai" },
      "environment": { "OMNIROUTE_API_KEY": "SUA_CHAVE_OMNIROUTE_AQUI" },
      "config": {
        "url": "http://127.0.0.1:20128/v1",
        "apiKeyEnv": "OMNIROUTE_API_KEY",
        "model": "agent-default"
      }
    },
    "omniroute-nemotron-super": {
      "driver": "openai-compat",
      "displayName": "OmniRoute — Nemotron 3.5 Super VL 120B",
      "icon": { "kind": "preset", "preset": "openai" },
      "environment": { "OMNIROUTE_API_KEY": "SUA_CHAVE_OMNIROUTE_AQUI" },
      "config": {
        "url": "http://127.0.0.1:20128/v1",
        "apiKeyEnv": "OMNIROUTE_API_KEY",
        "model": "nvidia/nvidia/nemotron-3.5-super-vl-120b-a12b"
      }
    },
    "omniroute-nemotron-ultra": {
      "driver": "openai-compat",
      "displayName": "OmniRoute — Nemotron 3 Ultra 550B",
      "icon": { "kind": "preset", "preset": "openai" },
      "environment": { "OMNIROUTE_API_KEY": "SUA_CHAVE_OMNIROUTE_AQUI" },
      "config": {
        "url": "http://127.0.0.1:20128/v1",
        "apiKeyEnv": "OMNIROUTE_API_KEY",
        "model": "nvidia/nvidia/nemotron-3-ultra-550b-a55b"
      }
    },
    "omniroute-nemotron-lightning": {
      "driver": "openai-compat",
      "displayName": "OmniRoute — Nemotron 3.5 Lightning 30B",
      "icon": { "kind": "preset", "preset": "openai" },
      "environment": { "OMNIROUTE_API_KEY": "sk-omb--dvGnfNGuK5vWUCuziYWQUnQvJYcJHTz" },
      "config": {
        "url": "http://127.0.0.1:20128/v1",
        "apiKeyEnv": "OMNIROUTE_API_KEY",
        "model": "nvidia/nvidia/nemotron-3.5-lightning-30b-a3b"
      }
    }
  }
}
```

> **Importante:** substitua `SUA_CHAVE_OMNIROUTE_AQUI` pela chave gerada no passo 4.3.  
> Defina permissões restritas: `chmod 600 ~/.openmausbot/config.json`

### 5.3. Reinicie o OpenMausBot

As instâncias são lidas no boot do servidor. Reinicie o servidor para aplicar.

---

## 6. Iniciar o OpenMausBot

Você precisará de três terminais (ou use `tmux`/`screen`):

### Terminal 1 — Servidor de harness

```sh
cd /caminho/para/OpenMausBot
pnpm dev:server
# Roda em http://127.0.0.1:8799
```

### Terminal 2 — App (interface web)

```sh
cd /caminho/para/OpenMausBot
pnpm dev
# Roda em http://127.0.0.1:5199
```

### Terminal 3 — Shell Electron (opcional, desktop app)

```sh
cd /caminho/para/OpenMausBot
pnpm dev:desktop
```

---

## 7. Acessar a interface

Abra no navegador:

```
http://127.0.0.1:5199/
```

Ou:

```
http://localhost:5199/
```

---

## 8. Configurar um bot para usar o OmniRoute

1. Abra o OpenMausBot no navegador.
2. Crie um novo bot ou selecione um existente.
3. No **model picker** (seletor de modelo), escolha uma das instâncias OmniRoute:
   - **OmniRoute (NVIDIA)**
   - **OmniRoute — Nemotron 3.5 Super VL 120B**
   - **OmniRoute — Nemotron 3 Ultra 550B**
   - **OmniRoute — Nemotron 3.5 Lightning 30B**
4. Selecione o modelo desejado e comece a conversar.

> ⚠️ **O OpenMausBot não possui um "engine padrão global".** Cada bot escolhe seu engine individualmente no model picker. Você precisa selecionar OmniRoute para cada bot que quiser usar com os modelos NVIDIA.

---

## 9. Comandos úteis

| Comando | Descrição |
|---------|-----------|
| `pnpm typecheck` | Verifica tipos (app + server) |
| `pnpm test` | Roda os testes |
| `pnpm build` | Build de produção |
| `pnpm lint` | Executa o linter |
| `pnpm dev:server` | Inicia apenas o servidor de harness |
| `pnpm dev` | Inicia apenas a interface web |
| `pnpm dev:desktop` | Inicia o shell Electron |

### Parar os serviços

```sh
pkill -f "server/dev-server.ts"
pkill -f "vite"
```

---

## 10. Arquivos de configuração importantes

| Caminho | Descrição |
|---------|-----------|
| `~/.openmausbot/config.json` | Instâncias de engines do OpenMausBot (inclui OmniRoute) |
| `~/.openmausbot/` | Diretório de dados locais do OpenMausBot |
| `/root/.omniroute/` | Dados do OmniRoute (se instalado globalmente) |
| `/home/.../OmniRoute/data/` | Dados do OmniRoute (se rodando via Docker) |

---

## 11. Segurança

- O `config.json` contém chaves de API em texto simples. Mantenha permissões `0600`.
- Não commite o `config.json` no repositório Git.
- Use chaves de API com escopo restrito quando possível.
- O OmniRoute armazena credenciais criptografadas no SQLite (`storage.sqlite`).

---

## 12. Solução de problemas

| Problema | Solução |
|----------|---------|
| `Authentication required` no OmniRoute | Verifique se a chave API está correta e se `REQUIRE_API_KEY` está configurado |
| Modelo não aparece no picker | Reinicie o servidor do OpenMausBot após editar `config.json` |
| OmniRoute não responde | Verifique se o container/processo está rodando na porta 20128 |
| `pnpm install` falha | Verifique se Node >= 24 e pnpm >= 10 estão instalados |

---

## 13. Referências

- [OpenMausBot README](README.md)
- [Docs: Bring your own engine](docs/custom-engines.md)
- [OmniRoute CLI Tools](https://github.com/diegosouzapw/OmniRoute/blob/release/v3.8.52/docs/reference/CLI-TOOLS.md)
- [OmniRoute website](https://omniroute.online/)

---

*Este guia foi gerado com base na instalação realizada em Ubuntu 26.04 com OpenMausBot v0.1.155 e OmniRoute v3.8.53.*
