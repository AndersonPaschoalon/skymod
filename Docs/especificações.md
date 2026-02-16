Perfeito. Vamos tratar isso **como especificação de produto**, não como brainstorm.
Vou ser **formal, preciso e didático**, e quando algo for uma **má ideia**, vou dizer explicitamente.

Abaixo está a **especificação v1** dos **artefatos de configuração** do projeto `skymod`, com:

* propósito
* localização
* formato
* campos
* invariantes
* quem lê/escreve
* relação com Git
* erros comuns que **não serão permitidos**

---

# Visão geral dos artefatos

O projeto trabalha com **três domínios distintos**, e isso é fundamental:

| Domínio                | Onde vive                   | Versionado |
| ---------------------- | --------------------------- | ---------- |
| **Host / Engine**      | `~/.skymod/`                | ❌ Nunca    |
| **Ambiente de jogo**   | Steam / Documents / AppData | ❌ Nunca    |
| **Mirror (workspace)** | Diretório do mod            | ✅ Sempre   |

Os arquivos abaixo existem **para garantir determinismo entre esses domínios**.

---

# 1. `manifest.json` — Manifesto do ambiente

## Propósito

> Representar o **estado estrutural e criptográfico** de um ambiente de jogo em um instante no tempo.

Ele é a **fonte de verdade** para:

* vanilla
* dev
* test

Nunca representa o mirror.

---

## Localização

Sempre **dentro de um ambiente**, nunca no Git:

```
env/vanilla/v1/manifest.json
env/dev/mymod/manifest.json
env/test/run-001/manifest.json
```

---

## Quem cria / quem lê

| Ator                      | Ação                    |
| ------------------------- | ----------------------- |
| `skymod vanilla snapshot` | cria                    |
| `skymod dev create`       | cria (copiando vanilla) |
| `skymod test create`      | cria (copiando vanilla) |
| `skymod status`           | lê                      |
| `skymod track`            | lê                      |
| `skymod deploy`           | lê                      |

🚫 **Nunca editável manualmente**
🚫 **Nunca versionado no Git**

---

## Formato (JSON canônico)

```json
{
  "schema": "skymod.manifest.v1",
  "game": "skyrim",
  "environment": {
    "type": "dev",
    "name": "mymod"
  },
  "created_at": "2026-02-03T21:40:12Z",
  "root_folders": [
    "Game",
    "Documents",
    "AppData"
  ],
  "files": {
    "Data/MyMod.esp": {
      "hash": "sha256:abc123...",
      "size": 183424,
      "category": "plugin"
    },
    "Data/scripts/source/MyQuest.psc": {
      "hash": "sha256:def456...",
      "size": 4211,
      "category": "script-source"
    }
  }
}
```

---

## Semântica importante

### `files`

* Lista **todos os arquivos rastreáveis**
* Inclui vanilla + mods
* É a base de comparação para detectar:

  * novos arquivos
  * modificações
  * corrupção

### `category`

Categoria **semântica**, não técnica:

* `plugin` (`.esp`, `.esm`, `.esl`)
* `script-source` (`.psc`)
* `asset`
* `binary` (quase sempre ignorado)
* `unknown`

Essa categorização **alimenta regras hard**.

---

## Invariantes (NÃO negociáveis)

* ❌ Se um arquivo não está aqui, ele **não existia no snapshot**
* ❌ Hash nunca muda sem update explícito
* ❌ Manifest nunca referencia arquivos do mirror
* ✔️ É **imutável após criação**, exceto por operações do `skymod`

---

# 2. `manifest.lock` — Manifesto do mirror

## Propósito

> Representar **exatamente** o que foi extraído do ambiente de jogo e levado para o Git.

Ele é o **equivalente conceitual** de um `package-lock.json` ou `poetry.lock`.

---

## Localização

Sempre no **mirror (workspace Git)**:

```
mymod/.skymod/manifest.lock
```

---

## Quem cria / quem lê

| Ator                 | Ação       |
| -------------------- | ---------- |
| `skymod mirror init` | cria vazio |
| `skymod track`       | atualiza   |
| `skymod status`      | compara    |
| `skymod deploy`      | lê         |

✔️ **Versionado no Git**
✔️ **Auditável**
🚫 **Não editável manualmente**

---

## Formato

```json
{
  "schema": "skymod.manifest-lock.v1",
  "game": "skyrim",
  "env": "dev/mymod",
  "tracked_at": "2026-02-04T01:12:55Z",
  "files": {
    "Data/MyMod.esp": {
      "hash": "sha256:abc123...",
      "source": "Game/Data/MyMod.esp"
    },
    "Data/scripts/source/MyQuest.psc": {
      "hash": "sha256:def456...",
      "source": "Game/Data/scripts/source/MyQuest.psc"
    }
  }
}
```

---

## Semântica

* Representa **o que está no Git**
* Não representa o estado atual do ambiente
* Permite detectar:

  * arquivos removidos do jogo
  * arquivos alterados fora do Git
  * divergência dev ↔ mirror

---

## Invariantes

* ❌ Um arquivo no Git **sem entrada aqui é inválido**
* ❌ Um hash diferente indica **estado sujo**
* ✔️ É a única ponte formal entre jogo e Git

---

# 3. `.skymod-ignore` — Regras de exclusão do ambiente

Aqui você tocou num ponto crítico. Vou ser direto:

> 👉 **Sua ideia é BOA, mas precisa de limites claros.**

---

## Propósito

> Declarar **explicitamente** o que **NUNCA** deve ser considerado para versionamento.

Ele vive **no ambiente do jogo**, não no Git.

---

## Localização

```
env/dev/mymod/.skymod-ignore
```

🚫 Nunca no vanilla
🚫 Nunca em gaming
✔️ Sempre criado em `dev create`

---

## Formato

✔️ **Idêntico ao `.gitignore`**, deliberadamente.

Exemplo:

```gitignore
# Hard rules (não removíveis)
*.pex
*.dll
*.log

# Creation Kit noise
Data/Source/Temp/**
Data/Cache/**

# User rules
Data/scripts/source/debug/**
```

---

## Regras de ouro

### 1. Existem **regras hard** (embutidas)

* Nunca removíveis
* Nunca sobrescritas
* Ex: `.dll`, `.pex`, `.bsa`

### 2. Existem **regras soft** (user)

* Adicionadas via `skymod untrack`
* Persistentes
* Versionáveis (ver abaixo)

---

## Relação com `.gitignore`

### Decisão correta (importante):

> ❌ **NÃO usar o `.skymod-ignore` diretamente como `.gitignore`**

Motivos:

* Escopos diferentes
* Riscos semânticos
* Git não entende categorias

### O que fazer então?

* `.gitignore` no mirror é **gerado**
* Ele é uma **tradução segura** do `.skymod-ignore`
* Ambos são versionados no Git:

  * `.skymod-ignore` → **fonte**
  * `.gitignore` → **derivado**

✔️ Isso permite:

* auditoria
* reconstrução
* entendimento humano

👉 **Sua exigência de versionar o `.skymod-ignore` é correta**.

---

# 4. `.skymod/config.yaml` — Configuração do mirror

## Propósito

> Declarar o **vínculo explícito** entre o workspace Git e um ambiente.

Sem isso, o mirror é inválido.

---

## Localização

```
mymod/.skymod/config.yaml
```

---

## Formato

```yaml
schema: skymod.config.v1
game: skyrim
environment: dev/mymod
vanilla: v1
created_at: 2026-02-03T22:01:00Z
```

---

## Invariantes

* ❌ Um mirror sem esse arquivo é inválido
* ❌ Um mirror apontando para env inexistente falha
* ✔️ Usado por todos os comandos `track`, `status`, `deploy`

---

# 5. Arquivos auxiliares (v1)

### `.skymod/state.json` (host-side)

* Guarda:

  * ambiente ativo
  * último swap
* Vive em `~/.skymod/`
* Nunca versionado
* Essencial para rollback

---

# 6. O que você **não esqueceu**, mas ainda não nomeou

### ❗ `ruleset.json` (embutido no binário)

* Define categorias
* Define hard-ignore
* Define extensões perigosas
* Nunca editável

Sem isso, o produto vira um “rsync glorificado”.

---

# Veredito final (sem floreios)

* Seu modelo está **correto em essência**

* A separação:

  * `manifest.json` (ambiente)
  * `manifest.lock` (Git)
  * `.skymod-ignore` (intenção)

  é **excelente** e rara de ver em ferramentas de modding.

* O único ponto que **exigiu correção** foi o uso direto do `.gitignore`.

Se quiser, o **próximo passo natural** é:

> **Especificar o algoritmo exato de `status` e `track` (com pseudocódigo e invariantes)**

Isso fecha o core lógico da ferramenta.
