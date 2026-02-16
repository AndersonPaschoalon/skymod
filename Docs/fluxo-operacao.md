# 1. Princípios fundamentais

## 1.1 Princícios de funcionamento

> 🔒 **Regra de ouro do produto**
> `skymod` **NÃO versiona ambientes**
> `skymod` **ajuda a versionar artefatos de mod.**
> `skymod` **ajuda a criar ambientes para desenvolvimento e  testes de mods, e isolá-los do ambiente de gameplay.**

Ambientes:
* vanilla
* gaming
* dev/*
* test/*

👉 **Nunca entram no Git**
👉 **Nunca são manipulados diretamente pelo usuário**

O Git **só conhece o mirror**.

Isso precisa guiar **todos** os comandos. Se algum comando violar isso, ele está errado.

## 1.2 Definições, Estados e Operações atomicas 

### 1.2.1 Ambiente

* **Ambiente**: um ambiente trata-se do conjuntos das diretórios da intalação padrão. No caso de Skyrim:
  * Steamapps
  * Documents/My Games
  * AppData

### 1.2.2 Estados

* **Ativado**: O ambiente  está disponível tanto para jogar, quanto para o desenvolvimento. Todas as pastas do ambiente estão nas localizações esperadas da instalação, ou seja, nos diretórios **padrão**.
* **Desativado**: O ambiente está localizado em qualquer uma dos diretórios de **snapshot** localizados em `.skymod/games/skrim/`, ou seja Vanilla, gaming, dev e test.

### 1.2.2 Operações

Aqui supomos a existência de dois ambientes `a` e `b`. As seguintes operações são possíveis:
* **Está-Ativo a**: Verifica se o ambiente **a** está ativo (True) ou não (False).
* **Listar-Ativo**: Retorna o nome do ambiente atualmente ativo.
  * Caso não exista ambiente ativo, retorna string vazia;
  * Alguns valores possiveis são:
    * live;gaming
    * dev/mymod
    * test/test_01
  * Deve ser impossivel um ambiente **vanilla** esteja ativo.
* **Listar-Desativados**: lista  ambientes desativados.
  * Alguns valores possiveis são:
    * vanilla/v1
    * live/gaming
    * dev/mymod
    * test/test_01
* **Clonar a c**: Faz uma cópia de um ambiente **a** desativado em outro diretório de **snapshot**, nomeando o novo ambiente como **c**.
  * Operação deve  sempre deve ser realizado a partir de um ambiente **vanilla/** para um ambiente **dev/\*** ou **test/\***
* **Ativar a**: Copiar qualquer o conteúdo de um diretório de **snapshot** para diretórios **padrão**.
  * Operação somente é possivel caso não existá outro ambiente ativado.
  * Somente é possivel 
  * Caso a operação falhe, os diretórios devem parmanecer no diretório de **snapshot**.
* **Desativar a**: Mover os contúdos dos diretórios padrão para os diretórios de **snapshot**.
  * Caso o jogo esteja aberto, a operação deverá falhar;
* **Swap a, b**: esta operação sempre deve ser realizada entre um ambiente **Ativado**(a) e um ambiente **Desativado**(b). As seguintes operações devem ser executadas em ordem, com sucesso:
  * **Desativar a**;
  * **Ativar b**.

Caso qualquer uma das operações falhe durante a execução, a aplicação deverá ser capaz de reverter o sistema para o estado anterior. Ou seja, caso qualquer uma das operações de **move** falhem, todas as operações de **move** anteriormente realizadas deverão ser revertidas.
Através dessas operações, todas as mudanças de ambientes necessárias serão possíveis.

---

# 2. Análise do fluxo proposto 


## 2.1 Inicialização global

```bash
# aqui podemos estar em qualquer pasta do sistema operacional
skymod init skyrim
skymod set skyrim
```

### O que acontece no mundo real

**`skymod init skyrim`**

* Detecta instalações padrão:

  * Steam
  * Documents/My Games
  * AppData
* Cria estrutura **interna e isolada**:

```
.skymod/
└── games/
    └── skyrim/
        ├── config.yaml
        ├── store/
        ├── env/
        │   ├── vanilla/
        │   ├── dev/
        │   └── test/
        └── live/
            └── gaming/   # estado ativo inicial
```

🚫 **Não copia nada ainda**
🚫 **Não toca na instalação do jogo**

---

**`skymod set skyrim`**

* Define `SKYMOD_GAME=skyrim`
* O Objetivo é definir qual jogo será manipulado na **sessão do console**.
* Todos os comandos seguintes falham sem isso


---

## 2.2 Snapshot do vanilla

```bash
skymod vanilla snapshot
```

### O que acontece no mundo real

* Faz um snapshot **atomicamente** da instalação do jogo, e a define como **vanilla**, copiando:
  * pasta do jogo (steamapps)
  * Documents/My Games
  * AppData
  Para:

  ```
  env/vanilla/v1/
  ```
🚫 **Nunca mais toca nesse diretório automaticamente**

* ✔️ Calcula:
  * hash por arquivo
  * manifest global (`manifest.json`)

* ✔️ Marca:
```
env/vanilla/current -> v1
```

#### Obervações

1. O comando somente terá efeito pratico na primeira vez que ser executado. A menos que o usuário remova manualmente os arquivos em `  env/vanilla/v1/`, skymod irá detectar que o **snapshot** do vanilla já foi feito, irá notificar que já foi criado, e sairá. Isso é necessário para evitar multiplos **snapshots** desnecessários. 
2. Caso o usuário deseje criar um novo ambiente vanilla por qualquer razão (ambiente original foi corrompido, insuficiente, etc...), o usuário deverá primeiramente forçar a criação de uma nova versão com a opção `--new`. Em seguida, deverá escolher a versão vanilla default através do comando `set-default`. Poderá visualizar os **snapshots** vanillas através do comando `show`.
```
# Usuario deseja criar um novo snapshot
skymod vanilla snapshot --new
# stdout: Vanilla snapshot `v2` created.

# Usuario deseja visualizar versões dos snapshots criados
skymod vanilla show
# stdout: Tabela mostrando as versões disponiveis e datas de criação

# Usuario deseja selecionar v2 como novo snapshot base
skymod vanilla set-default v2
```


## 2.3 Criação do ambiente de desenvolvimento

```bash
skymod dev create mymod
skymod activate mymod
```

### O que acontece no mundo real

**`skymod dev create mymod`**
* Um ambiente isolado para desenvolvimento chamado *mymod* é criado a partir do *vanilla*.
* Criar arquivo `.skymod-ignore` **NO AMBIENTE DO JOGO**
* criar `manifest.json` baseando-se no ambiente **vanilla** e em `.skymod-ignore`. 

**`skymod activate mymod`**
* Ambiente **mymod** deverá ser ativado:
  * O ammbiente atualmente ativo deverá ser detectado.
  * Caso exista um ambiente ativo, a operação de **Swap <current> dev/mymod** deverá ser realizada;
  * Caso não exista um ambiente ativo, a operação **Ativar dev/mymod** deverá ser realizada.


## 2.4 Inicialização do mirror (workspace Git)

```bash
skymod mirror init
cd mymod
git init
```

### O que acontece no mundo real

**`skymod mirror init`**

* Deve criar o diretório:
```
./mymod/
```
  * **IMPORTANTE**: deve ser executado onde o diretório raiz do repositório deve ser criado.

* Cria arquivos:
```
mymod/
├── .skymod/
│   ├── game = skyrim
│   ├── env = dev/mymod
│   └── manifest.lock
├── .gitignore        # espelhado do ambiente do jogo
└── README.md
```
  🚫 **Não copia nenhum arquivo de mod**
  🚫 **Não toca no Git ainda**
* 👉 O mirror começa **vazio por definição**


## 2.5 Desenvolvimento e tracking

```bash
# editar mod
# creation kit

skymod status
skymod track --all
```

### O que acontece no mundo real

Aqui está o CORAÇÃO do produto.

**`skymod status`**

* Compara:
  * estado atual do ambiente dev
  * vs `manifest.json` original
  * vs último `manifest.lock`
* Classifica:
  * NEW
  * MODIFIED
  * IGNORED (hard rules)
  * IGNORED (user rules)
* 🚫 **Nunca olha o Git**
* 🚫 **Nunca confia em timestamps**
* ✔️ Usa **hashes**


**`skymod track --all`**
  * ✔️ Copia **somente arquivos aprovados**
  * ✔️ Replica **estrutura de diretórios**
  * ✔️ Atualiza:
  ```
  manifest.lock
  ```
  * 🚫 **Nunca copia lixo**
  * 🚫 **Nunca copia vanilla**
  * 🚫 **Nunca copia artefatos proibidos**

### Comandos adicionais

**`skymod untrack <file>`**
* Cria uma regra adicional em `.skymod-ignore`, ignorando o arquivo especificado;
* Regra é refletida automaticamente no mirror `.gitignore`.


## 2.6 Git

```bash
git add .
git commit -m "v1"
```

Aqui está a beleza do design:
* Git não sabe que Skyrim existe
* Git não sabe que existe Creation Kit
* Git vê **um projeto normal**


## 2.7 Ambiente de teste

```bash
skymod test create run-001
skymod activate test/run-001
```

### O que acontece no mundo real

* **`skymod test create run-001`**
* ✔️ Copia `env/vanilla/v1` → `env/test/run-001`
* ✔️ Faz deploy:
  * copia **somente mirror**
  * para dentro do ambiente de teste


* **`skymod activate test/run-001`**
* ✔️ Swap:
```
live/current <-> env/test/run-001
```
  🚫 Não contamina gaming
  🚫 Não contamina dev



## 2.8 Retorno e limpeza

```bash
skymod activate gaming
skymod test destroy run-001
```

### O que acontece no mundo real

**`skymod activate gaming`**
✔️ Swap reversível
✔️ Retorna ao estado do jogador

**`skymod test destroy run-001`**
✔️ Remove **APENAS**:

```
env/test/run-001
```

🚫 Nunca remove:

* gaming
* vanilla
* dev

✔️ Restrição: remover só dentro de env/test


