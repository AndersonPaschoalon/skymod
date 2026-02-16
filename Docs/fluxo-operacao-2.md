#  Especificação Operacional do `skymod`

---

# 1. Princípios Fundamentais

## 1.1 Princípios de Funcionamento

> 🔒 **Regra de ouro do produto**
>
> * `skymod` **não versiona ambientes**.
> * `skymod` **auxilia na versionamento de artefatos de mod**.
> * `skymod` **cria, isola e gerencia ambientes de desenvolvimento e teste**, separando-os rigidamente do ambiente de gameplay.

Os ambientes possíveis são:

* `vanilla`
* `gaming`
* `dev/*`
* `test/*`

Estes ambientes:

* **Nunca entram no Git**
* **Nunca são manipulados manualmente pelo usuário**
* **Nunca são parcialmente modificados fora das operações definidas**

O Git interage exclusivamente com o **mirror (workspace)**.

Essa separação é estrutural. Qualquer comando que viole essa regra está incorreto por definição.

---

## 1.2 Definições, Estados e Operações Atômicas

### 1.2.1 Ambiente

Define-se como **ambiente** o conjunto completo dos diretórios que compõem a instalação funcional do jogo.

No caso de *Skyrim*, isso inclui:

* Diretório da instalação Steam (`steamapps/common/...`)
* `Documents/My Games/...`
* `AppData/...`

Esses três diretórios formam uma unidade lógica indivisível.
Nenhuma operação pode afetar apenas parte desse conjunto.

---

### 1.2.2 Estados

Um ambiente pode estar em um dos dois estados:

#### Ativado

O ambiente está presente nos diretórios padrão do sistema operacional.
O jogo e o Creation Kit podem utilizá-lo normalmente.

Somente **um ambiente pode estar ativado por vez**.

---

#### Desativado

O ambiente está armazenado sob:

```
.skymod/games/<game>/env/
```

Exemplos:

* `vanilla/v1`
* `live/gaming`
* `dev/mymod`
* `test/run-001`

Ambientes desativados são snapshots íntegros e completos.

---

### 1.2.3 Operações Fundamentais

As seguintes operações são definidas formalmente:

---

#### Está-Ativo(a)

Retorna `true` se o ambiente `a` estiver ativado, `false` caso contrário.

---

#### Listar-Ativo

Retorna o identificador do ambiente atualmente ativado.

Exemplos válidos:

* `live/gaming`
* `dev/mymod`
* `test/run-001`

É **proibido** que um ambiente `vanilla/*` esteja ativado.

Se nenhum ambiente estiver ativado, retorna string vazia.

---

#### Listar-Desativados

Retorna todos os ambientes atualmente armazenados sob `env/`.

---

#### Clonar(a → c)

Cria um novo ambiente `c` a partir de `a`.

Restrições:

* `a` deve ser `vanilla/*`
* `c` deve estar sob `dev/*` ou `test/*`
* `a` deve estar desativado
* Operação realizada por cópia integral

---

#### Ativar(a)

Copia integralmente o conteúdo de `a` para os diretórios padrão do sistema.

Restrições:

* Nenhum outro ambiente pode estar ativado.
* Caso qualquer etapa falhe, o sistema deve permanecer no estado anterior.

---

#### Desativar(a)

Move os diretórios padrão para o diretório snapshot correspondente a `a`.

Restrições:

* O jogo não pode estar em execução.
* A operação deve ser reversível.
* Nenhuma exclusão é permitida fora de `env/test`.

---

#### Swap(a, b)

Operação composta e atômica:

1. Desativar(a)
2. Ativar(b)

`a` deve estar ativado e `b` desativado.

Se qualquer etapa falhar, todas as alterações devem ser revertidas.

---

# 2. Análise do Fluxo Proposto

---

## 2.1 Inicialização Global

```bash
skymod init skyrim
skymod set skyrim
```

### `skymod init skyrim`

Esta operação registra o jogo no sistema `skymod`.

Ela:

* Detecta os diretórios padrão do jogo.
* Cria estrutura interna:

```
.skymod/
└── games/
    └── skyrim/
        ├── config.yaml
        ├── env/
        │   ├── vanilla/
        │   ├── dev/
        │   └── test/
        └── live/
            └── gaming/
```

Não realiza cópias.
Não altera a instalação do jogo.

---

### `skymod set skyrim`

Define o jogo ativo na sessão corrente via variável de ambiente.

Sem esse comando, todas as operações falham explicitamente com mensagem diagnóstica clara.

A validade é restrita à sessão do terminal.

---

## 2.2 Snapshot do Vanilla

```bash
skymod vanilla snapshot
```

Esta operação cria o snapshot base.

Ela:

1. Copia integralmente os diretórios padrão.
2. Armazena sob:

```
env/vanilla/v1/
```

3. Calcula hash SHA-256 por arquivo.
4. Gera `manifest.json`.
5. Define `v1` como default.

Após criado, o snapshot não é modificado automaticamente.

---

### Política de múltiplos snapshots

Por padrão, apenas um snapshot é criado.

Para criar novo:

```bash
skymod vanilla snapshot --new
```

Listagem:

```bash
skymod vanilla show
```

Definir padrão:

```bash
skymod vanilla set-default v2
```

Essa política evita explosão de snapshots.

---

## 2.3 Criação de Ambiente de Desenvolvimento

```bash
skymod dev create mymod
skymod activate mymod
```

### `skymod dev create mymod`

* Clona `vanilla/default`
* Cria `env/dev/mymod`
* Gera `.skymod-ignore`
* Gera `manifest.json`

Esse ambiente nasce desativado.

---

### `skymod activate mymod`

* Detecta ambiente ativo atual
* Executa `Swap(current, dev/mymod)` se necessário
* Caso nenhum ativo, executa `Ativar(dev/mymod)`

Essa operação é sempre atômica.

---

## 2.4 Inicialização do Mirror (Workspace Git)

```bash
skymod mirror init
cd mymod
git init
```

`skymod mirror init` deve ser executado no diretório onde o repositório será criado.

Ele:

* Cria diretório `./mymod/`
* Cria:

```
mymod/
├── .skymod/
│   ├── config.yaml
│   └── manifest.lock
├── .gitignore
└── README.md
```

Não copia arquivos do jogo.
Não executa comandos Git.
Não toca no ambiente.

O mirror nasce vazio por definição.

---

## 2.5 Desenvolvimento e Tracking

```bash
skymod status
skymod track --all
```

### `skymod status`

Compara:

* Ambiente dev atual
* `manifest.json`
* `manifest.lock`

Classifica arquivos em:

* NEW
* MODIFIED
* IGNORED (hard rules)
* IGNORED (user rules)

Nunca usa timestamps.
Nunca consulta Git.
Baseia-se exclusivamente em hash criptográfico.

---

### `skymod track --all`

* Copia apenas arquivos permitidos.
* Replica estrutura.
* Atualiza `manifest.lock`.

Nunca copia arquivos vanilla.
Nunca copia binários proibidos.
Nunca copia ruído do Creation Kit.

---

### `skymod untrack <file>`

* Adiciona regra a `.skymod-ignore`
* Atualiza `.gitignore` no mirror

---

## 2.6 Integração com Git

```bash
git add .
git commit -m "v1"
```

O Git enxerga um projeto comum.

Ele não tem conhecimento sobre:

* Skyrim
* Creation Kit
* Ambientes

Essa separação é intencional.

---

## 2.7 Ambiente de Teste

```bash
skymod test create run-001
skymod activate test/run-001
```

### `skymod test create run-001`

* Clona `vanilla/default`
* Copia arquivos do mirror
* Gera ambiente isolado

---

### `skymod activate test/run-001`

Executa swap atômico:

```
live/current <-> env/test/run-001
```

Não contamina `gaming`.
Não contamina `dev`.

---

## 2.8 Retorno e Limpeza

```bash
skymod activate gaming
skymod test destroy run-001
```

### `skymod activate gaming`

Executa swap reversível.

Restaura o ambiente do jogador.

---

### `skymod test destroy run-001`

Remove exclusivamente:

```
env/test/run-001
```

Restrições:

* Só permitido sob `env/test`
* Nunca remove `gaming`
* Nunca remove `vanilla`
* Nunca remove `dev`

---

# Considerações Analíticas

Seu modelo é sólido e demonstra pensamento de isolamento forte e atomicidade.

Os pontos fortes são:

* Separação estrutural entre ambiente e artefato
* Uso de hash em vez de timestamp
* Proibição de manipulação parcial
* Modelo de swap atômico

Os pontos que exigirão cuidado na implementação real são:

* Atomicidade cross-filesystem
* Garantia de rollback real
* Performance em snapshot inicial
* Detecção de corrupção silenciosa

