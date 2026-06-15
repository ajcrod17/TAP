# TAP — Plano de Trabalho

Lista ordenada de tarefas por pessoa, com caminho do ficheiro.

**Estados:**
- `[ ]` — por fazer
- `[REVIEW]` — referência implementada pelo Claude. **Analisar, perceber e reescrever à nossa maneira** (o subject exige que saibamos explicar o código na defesa). Os comentários em português marcam o que rever.
- `[x]` — feito por nós, validado.

---

## Estado actual

- [x] Estrutura do repositório criada
- [x] Contrato do `internal/protocol` fechado (structs, signatures, error codes)
- [x] `data/world.json` com 8 salas, NPCs, items, quests (faltam descriptions)
- [x] Deviations à RFC documentadas no [README.md](README.md)
- [x] Scaffold do GUI Fyne em [cmd/gui/main.go](cmd/gui/main.go) com assinaturas dos painéis
- [REVIEW] `internal/protocol` implementado como referência (command, response, event) + testes a passar — verificado contra o RFC real (error codes e EVT conformes)

---

## Verificar na escola

Correr a partir da raiz do projeto. "Está tudo OK" = os três sem queixas.

```bash
go test ./...        # corre todos os testes. Bom: linha "ok ...". Mau: aparece "FAIL"
go vet ./...         # análise estática. Bom: não imprime nada
gofmt -l .           # lista ficheiros mal formatados. Bom: não imprime nada
```

Se o `gofmt -l` listar algum ficheiro, corrige tudo de uma vez:

```bash
gofmt -w .           # formata automaticamente
```

Ver os testes em detalhe (caso a caso), só do protocolo:

```bash
go test -v ./internal/protocol
```

Mais tarde, para o servidor (apanha race conditions de concorrência):

```bash
go test -race ./...
```

---

## RUI — Server, Engine & CLI

Ordem recomendada (cada ficheiro depende do anterior estar minimamente funcional).

### Bloco 0 — Estudar (antes de escrever código)

**Go geral:** sintaxe, tipos, structs, pointers, interfaces, error handling, packages, modules.
Recurso principal: [A Tour of Go](https://go.dev/tour) + [Effective Go](https://go.dev/doc/effective_go).

**Especificamente para o servidor:**
- [ ] **Goroutines & channels** — `go func(){}`, `chan T`, `select`, padrões "share by communicating" (cap. 8 do livro Donovan & Kernighan — *concurrent chat server* é literalmente este projeto)
- [ ] **Pacote `net`** — `net.Listen`, `net.Conn`, `Accept()` loop
- [ ] **`bufio.Scanner`** — ler linhas de uma `net.Conn`, escolher buffer size, lidar com tokens grandes
- [ ] **`encoding/json`** — `Marshal`/`Unmarshal`, tags nos structs (já vês exemplos em [response.go](internal/protocol/response.go))
- [ ] **`log/slog`** — handler JSON, log levels (INFO/WARN/ERROR), campos estruturados
- [ ] **`context` + `os/signal`** — graceful shutdown com Ctrl+C, `signal.NotifyContext`
- [ ] **Testes em Go** — `testing.T`, table-driven tests (útil para o `protocol`)

### Bloco 1 — Protocolo (desbloqueia tudo o resto)
- [OK] [internal/protocol/command.go](internal/protocol/command.go) — `Parse()` + `String()`
- [OK] [internal/protocol/response.go](internal/protocol/response.go) — `OK()`, `OKf()`, `OKJson()`, `Errf()` (helpers; structs já estavam)
- [OK] [internal/protocol/event.go](internal/protocol/event.go) — as 9 funções de EVT
- [OK] [internal/protocol/command_test.go](internal/protocol/command_test.go) — testes table-driven (Parse, responses, events). Em inglês; serve de molde para os outros pacotes

### Bloco 2 — Game core
- [OK] [internal/game/world.go](internal/game/world.go) — `NewWorld()`, `GetRoom()`, `GetPlayer()`, `AddPlayer()`, `RemovePlayer()`, `MovePlayer()`, `PlayersInRoom()`, `TotalPlayers()`, `NPCInRoom()`
- [OK] [internal/game/item.go](internal/game/item.go) — `TakeItem()`, `DropItem()`, `FindItem()`

### Bloco 3 — Server mínimo (vertical slice)
- [OK] [internal/server/client.go](internal/server/client.go) — `newClient()`, `readPump()`, `writePump()`, `safeSend()`
- [OK] [internal/server/hub.go](internal/server/hub.go) — `NewHub()`, `Run()`, `Accept()`, `broadcast()`, `broadcastAll()`, `removeClient()`, `updatePlayerCount()` (`broadcastGroup` → Bloco 4)
- [OK] [internal/server/dispatch.go](internal/server/dispatch.go) — `dispatch` + 6 handlers: `handleConnect`, `handleQuit`, `handleLook`, `handleMove`, `handleWho`, `handleChat`
- [OK] [cmd/server/main.go](cmd/server/main.go) — `main()` + `listenAndServe()` (+ shutdown gracioso via signal.NotifyContext)
- [OK] [cmd/cli/main.go](cmd/cli/main.go) — `main()` + `readLoop()` + `writeLoop()`
- [OK] [internal/server/server_test.go](internal/server/server_test.go) — teste de integração TCP (vertical slice, presença, chat, nome duplicado); passa com `-race`

> 🚦 **Checkpoint de integração** com o Alexandre. Testar CONNECT → LOOK → MOVE → CHAT → QUIT pelo CLI e pelo GUI.

### Bloco 4 — Sistemas
- [ ] [internal/server/dispatch.go](internal/server/dispatch.go) — handlers de grupo: `handleGroup`, `handleGroupCreate`, `handleGroupInvite`, `handleGroupJoin`, `handleGroupLeave`
- [ ] [internal/server/dispatch.go](internal/server/dispatch.go) — handlers de items: `handleTake`, `handleDrop`, `handleInventory`
- [ ] [internal/server/dispatch.go](internal/server/dispatch.go) — handler de NPCs: `handleTalk`
- [ ] [internal/game/combat.go](internal/game/combat.go) — `Attack()`, `calculateDamage()`, `RespawnPlayer()`
- [ ] [internal/server/dispatch.go](internal/server/dispatch.go) — handlers de combate: `handleAttack`, `handleStatus`
- [ ] [internal/game/quest.go](internal/game/quest.go) — `GetQuestFromNPC()`, `StartQuest()`, `CheckCompletion()`, `CompleteQuest()`, `ListQuests()`
- [ ] [internal/server/dispatch.go](internal/server/dispatch.go) — handlers de quests: `handleQuest`, `handleQuests`

### Bloco 5 — Polish & docs
- [ ] Logging estruturado completo em todos os handlers (slog JSON)
- [ ] Detecção de abuso (rate-limit + log WARN)
- [ ] [README.md](README.md) — secção **Architecture**
- [ ] [README.md](README.md) — secção **Combat System** (preencher fórmula de damage)
- [ ] [README.md](README.md) — secção **Quest System**
- [ ] [README.md](README.md) — secção **Server Logging** (thresholds)
- [ ] [README.md](README.md) — completar **Protocol Implementation** (scanner buffer, max line)

---

## ALEXANDRE — GUI, World & Infra

### Bloco 0 — Estudar (antes de escrever código)

**Go geral:** sintaxe, structs, error handling, packages. Precisas de uma noção básica de concorrência (uma goroutine de leitura + canal para a UI), nada de avançado.
Recurso: [A Tour of Go](https://go.dev/tour) (faz até "Concurrency" inclusive mas focado).

**Especificamente para o cliente GUI + world loader:**
    - [x] **Sanity check do Fyne** — correr [testeFyne/](testeFyne/) num PC da escola, ver se a janela abre (se falhar, decidir plano B antes de escrever código)
    - [x] **Fyne v2** — `app.New()`, `NewWindow()`, `widget.NewLabel/Button/Entry/List`, `container.NewVBox/HBox/Border/AppTabs`, `widget.Refresh()`, `fyne.Do(...)` para atualizar a UI a partir de outra goroutine. Ler o tutorial "Getting started" em [docs.fyne.io](https://docs.fyne.io)
    - [x] **`net.Dial`** — abrir conexão TCP do cliente GUI para o servidor TAP
- [ ] **`bufio.Scanner`** — ler linhas da `net.Conn` (uma resposta/evento por linha)
    - [x] **Goroutines + channels** — uma goroutine de leitura empurra linhas para um `chan string`, a goroutine da UI consome e atualiza widgets
- [ ] **`encoding/json`** — `Unmarshal` para parsear respostas (`LookResponse`, `InventoryItem`, etc.) e para ler `world.json`
- [ ] **`os.ReadFile`** — ler ficheiros de disco

### Bloco 1 — Independente do servidor (podes começar já)
	- [x] [data/world.json](data/world.json) — substituir todos os `"description": "TODO"` por textos reais
	- [x] [internal/worldfile/loader.go](internal/worldfile/loader.go) — `Load()` + `parseJSON()`
	- [x] [internal/worldfile/validate.go](internal/worldfile/validate.go) — `Validate()` + 6 funções `check*`
    - [x] [go.mod](go.mod) — `go get fyne.io/fyne/v2` e apagar o comentário
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — `main()`, `readLoop()`, `sendCommand()` (parte de rede sem UI ainda)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — `buildStatusBar()`, `buildActionBar()` (mais simples, começar por aqui para ter algo a aparecer). Need to test with go version above 1.19.

### Bloco 2 — UI base (não precisa do servidor a correr)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — `buildRoomPanel()` com placeholders (room name, description, exits, items, NPCs)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — `buildChatPanel()` com as 3 tabs (GLOBAL / ROOM / GROUP) + input
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — `buildInventoryPanel()` com lista + botão DROP
	- [x] [cmd/gui/main.go](cmd/gui/main.go) — layout principal que junta os painéis (Border / VBox / HBox)

> 🚦 **Checkpoint de integração** com o Rui. Vertical slice end-to-end.

### Bloco 3 — Resto do `cmd/gui/main.go` (já com o servidor a correr)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — `applyResponse()` e `applyEvent()` a fazer routing por verbo / categoria
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — handlers de LOOK e INVENTORY (parsear `LookResponse` / `InventoryItem` e refrescar widgets)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — handler de STATUS (atualiza HP no status bar)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — handlers de EVT ROOM PRESENCE ENTER/LEAVE
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — append de chat por scope (Global/Room/Group)
    - [x] [cmd/gui/main.go](cmd/gui/main.go) — handler de EVT STATS para o contador global de jogadores
    - [x] Botões de ATTACK / TALK / QUEST / QUESTS ligados a `sendCommand()`

### Bloco 4 — Polish & docs
- [ ] [README.md](README.md) — secção **World Design** (mapa, NPCs, items)
- [ ] [README.md](README.md) — secção **Building and Running** (passos detalhados)
- [ ] [Makefile](Makefile) — rever targets, garantir que `make lint` passa
- [ ] Testar fluxo completo dos exemplos do subject (V.5)

---

## AMBOS — Final

- [ ] [README.md](README.md) — secção **Testing** (multiplayer, combat, quests)
- [ ] [README.md](README.md) — actualizar **Resources** (AI usage com detalhes do que cada um usou IA)
- [ ] [README.md](README.md) — actualizar **Group Contributions** se necessário
- [ ] Peer review cruzado de todo o código
- [ ] Ensaiar a *recode exercise* da avaliação (modificação rápida ao vivo)

---

## Pontos de coordenação (sincronizar com o outro)

| Quando | O quê |
|---|---|
| Antes do Rui implementar o protocolo | Confirmar que o `response.go` está estável |
| Depois do Bloco 3 do Rui | Vertical slice juntos (CLI + GUI a ligar e fazer LOOK) |
| Depois do Bloco 4 do Rui | Integração de combat + quests no GUI do Alexandre |
| Antes da entrega | Peer review cruzado completo |

---

## Recode Exercise — modificações ao vivo para treinar

A correction sheet diz: *"Ask the group to make a small change to one of the systems (e.g., modify NPC dialogue, adjust combat damage, add a simple quest step). The modification should be feasible within a few minutes. Verify that **both group members** understand the codebase and can explain their implementation choices."*

**Ambos** têm de saber fazer estas modificações — não chega ser o dono do componente. Treinem alternando quem implementa.

### ⭐ Exemplos oficiais do eval sheet — treinar SEMPRE

São as três modificações explicitamente citadas no documento. Espera-se que ambos consigam fazê-las sem hesitar.

- [ ] ⭐ **Modify NPC dialogue** → mudar `npc.guard.dialogue` em [data/world.json](data/world.json), verificar com `TALK guard` (1 min)
- [ ] ⭐ **Adjust combat damage** → mudar a fórmula em [internal/game/combat.go](internal/game/combat.go) (`calculateDamage`), recompilar, verificar com `ATTACK` (2-3 min)
- [ ] ⭐ **Add a simple quest step** → ex: mudar o target de `quest.fetch_herbs` de 1 herb para 2, ajustar o `CheckCompletion` se necessário em [internal/game/quest.go](internal/game/quest.go) (5-7 min)

### 🟢 Alta probabilidade — encaixa em "few minutes" (1-3 min)

Variações próximas dos exemplos oficiais. Treinem **todas**.

- [ ] Mudar o **HP de um inimigo** em [data/world.json](data/world.json) → verificar quantos golpes precisa
- [ ] Tornar um NPC **hostile/não-hostile** em [data/world.json](data/world.json) → verificar `ATTACK`
- [ ] Mudar o **start_room** em [data/world.json](data/world.json) → novos players nascem lá
- [ ] Mudar a **% de HP no respawn** em [internal/game/combat.go](internal/game/combat.go) (`RespawnPlayer`)
- [ ] Mudar o **HP inicial do player** em [internal/game/world.go](internal/game/world.go) (`AddPlayer`)
- [ ] Mudar o **valor do greeting** (`proto=1` → `proto=2`) — encontrar onde está hard-coded
- [ ] Mudar a **cor** das mensagens de chat por scope no `appendChatMessage` de [cmd/gui/main.go](cmd/gui/main.go) (usar `canvas.Text` com `Color` diferente por scope)

### 🟡 Borderline — estica os "few minutes" (5-10 min)

Possíveis mas no limite do tempo. Treinem **algumas** para conhecer o fluxo.

- [ ] Adicionar uma **nova sala** ao [data/world.json](data/world.json) com exits bidirecionais
- [ ] Adicionar um **novo item** ([data/world.json](data/world.json): definir em `items` + referenciar em `rooms[].items`) → verificar com `LOOK` + `TAKE`
- [ ] Mudar a **reward de uma quest** ([data/world.json](data/world.json)) → completar a quest para verificar
- [ ] Adicionar um **campo extra à response do STATUS** (ex: `"alive": bool`) → toca em [response.go](internal/protocol/response.go) + [handleStatus](internal/server/dispatch.go)
- [ ] Adicionar um **log novo** num handler à escolha
- [ ] Adicionar um **novo error code** em [errors.go](internal/protocol/errors.go) e usá-lo num handler
- [ ] Adicionar um **novo botão** de ação no GUI (`widget.NewButton` em `buildActionBar` + chamada a `sendCommand`)

### 🔴 Improvável mas just in case (10+ min)

Provavelmente não pedem porque excede "few minutes", mas saberem o caminho de implementação ajuda na *defesa* do código.

- [ ] Adicionar um **novo comando** ao protocolo (ex: `EMOTE <action>`) → toca em [command.go](internal/protocol/command.go), [dispatch.go](internal/server/dispatch.go), [event.go](internal/protocol/event.go)
- [ ] Adicionar um **novo tipo de quest** (ex: `talk` — falar com N NPCs) em [quest.go](internal/game/quest.go)
- [ ] Adicionar um **comando de combate** como `DEFEND` ou `FLEE`
- [ ] Mostrar o **HP como barra** no GUI em vez de número
- [ ] **Contador visual** de quests completadas no GUI

> 💡 **Estratégia:** se conseguirem fazer todas as 🟢 sem hesitar, estão cobertos para o cenário mais provável. As 🟡 são para terem a confiança de que sabem onde tocar quando a modificação envolve mais de um ficheiro. As 🔴 não treinem a fundo — basta saberem explicar *como* abordariam.
