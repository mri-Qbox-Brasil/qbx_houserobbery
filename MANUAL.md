# qbx_houserobbery — Manual

Roubo a residências: o jogador arromba a porta com lockpick durante a madrugada, entra num interior IPL isolado por routing bucket e saqueia pontos de loot e props.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Fluxo do roubo](#fluxo-do-roubo)
5. [Interiores, casas e loot](#interiores-casas-e-loot)
6. [Integrações](#integrações)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Localização](#localização)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | `GetPlayer`, `Notify`, `SetPlayerBucket`, `GetDutyCountType('leo')` |
| `ox_lib` | Sim | Points, skillcheck, progress circle, callbacks, locale |
| `ox_inventory` | Sim | `exports.ox_inventory:Items()` para labels e entrega das recompensas |
| Recurso de lockpick que dispare `lockpicks:UseLockpick` | Sim | Único gatilho de entrada. `qbx_lockpick` / `ox_inventory` lockpick |
| `qbx_interior` (ou outro que ouça `qb-interior:client:screenfade`) | Não | Fade de tela na entrada e saída da casa |
| `qbx_policejob` | Não | Recebe `police:server:policeAlert` e `evidence:server:CreateFingerDrop` |

---

## Instalação

1. Copie a pasta `qbx_houserobbery` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_houserobbery
   ```
3. Garanta que os itens usados existam no `ox_inventory`:
   - Itens de entrada: `advancedlockpick`, `screwdriverset`
   - Recompensas dos pickups: `small_tv`, `toaster`, `microwave`
   - Recompensas de loot (config padrão): `metalscrap`, `plastic`, `copper`, `iron`, `aluminum`, `steel`, `glass`, `diamond_ring`, `goldchain`, `rolex`, `10kgoldchain`, `bandage`, `repairkit`, `cleaningkit`, `weed_white-widow`, `weed_skunk`, `weed_purple-haze`, `weed_og-kush`, `weed_amnesia`, `weed_ak47`
4. Os routing buckets `600`–`621` (um por casa) ficam reservados para este recurso. Se outro recurso usar esses buckets, mude `routingbucket` em `config/shared.lua`.

---

## Configuração

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `startHours` | number | Sim | Hora (in-game) a partir da qual o arrombamento é permitido (padrão `22`) |
| `endHours` | number | Sim | Hora até a qual o arrombamento é permitido (padrão `5`). A checagem passa quando `hora >= startHours` ou `hora <= endHours` |
| `fingerprintChance` | number | Sim | Chance (0–100) de deixar digital ao saquear. Não deixa digital se o jogador estiver de luvas |
| `useDrawText` | bool | Sim | `true` usa texto 3D no mundo; `false` usa o `lib.showTextUI` |

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `minimumPolice` | number | Sim | Policiais em serviço (`leo`) necessários para arrombar |
| `notEnoughCopsNotify` | bool | Sim | Notifica o jogador quando não há policiais suficientes |
| `requiredItems` | array de string | Sim | `{'advancedlockpick', 'screwdriverset'}`. Com lockpick comum, o jogador também precisa do item de índice 2 (chave de fenda) |
| `rewards` | array de pools | Sim | Pools de recompensa de loot. Ver abaixo |

#### Formato de um pool de `rewards`

```lua
{items = {'metalscrap', 'plastic'}, togive = {min = 2, max = 5}, toget = {min = 2, max = 5}}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `items` | array de string | Itens do pool. A lista é embaralhada na inicialização do recurso |
| `togive` | `{min, max}` | Quantos itens **distintos** do pool o jogador recebe |
| `toget` | `{min, max}` | Quantidade de cada item entregue |

O índice do pool no array `rewards` é o número usado no campo `pool` de cada ponto de loot em `config/shared.lua`.

### `config/shared.lua`

#### `interiors[n]`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `exit` | `vec4` | Sim | Ponto de saída dentro do interior (também é o destino da entrada) |
| `skillcheck` | array de dificuldades | Sim | Sequência do `lib.skillCheck` (`easy`, `medium`, `hard`) para arrombar |
| `callCopsTimeout` | ms | Sim | Atraso entre a entrada e o alerta à polícia |
| `loot` | array de `{ coords, pool }` | Sim | Pontos de busca. `pool` é a lista de índices de `rewards` sorteáveis nesse ponto |
| `pickups` | array de `{ coords, model, reward }` | Sim | Props do IPL que podem ser pegos. `model` é o prop, `reward` é o item entregue |

#### `houses[n]`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `routingbucket` | number | Sim | Bucket exclusivo dessa casa. Mantenha único por casa |
| `interior` | number | Sim | Índice do interior usado (referência a `interiors`) |
| `opened` | bool | Sim | Estado inicial (mantenha `false`). Vira `true` após o arrombamento |
| `coords` | `vec3` | Sim | Porta da casa no mundo |
| `setup.loot` | `{min, max}` | Sim | Quantos pontos de loot do interior são ativados nessa casa |
| `setup.pickups` | `{min, max}` | Sim | Quantos pickups do interior são ativados nessa casa |
| `loot` / `pickups` | table | Sim | Tabelas vazias preenchidas em runtime pelo servidor |

---

## Fluxo do roubo

1. O jogador usa um lockpick na porta de uma casa (raio de 3m do `coords`). O recurso ouve `lockpicks:UseLockpick`.
2. O servidor valida: casa ainda não aberta, `screwdriverset` no inventário (se o lockpick não for avançado), `minimumPolice` em serviço e o horário (`startHours`/`endHours`).
3. Passa um `lib.skillCheck` com a sequência de dificuldade do interior. Falhou, nada acontece; acertou, a casa é marcada como aberta para **todos os jogadores** e o jogador é teleportado ao interior no routing bucket da casa.
4. Após `callCopsTimeout` ms, o alerta é enviado à polícia.
5. Dentro do interior o jogador procura nos pontos de loot (progress de 4 a 8 segundos) e pega os props configurados. Cada ação pode deixar uma digital, salvo se estiver de luvas.
6. Sair pelo ponto de `exit` devolve o jogador ao bucket `0`, na porta da casa.

Uma casa aberta continua aberta: qualquer jogador pode entrar sem lockpick até o recurso reiniciar. Cada ponto de loot só rende recompensa uma vez.

---

## Interiores, casas e loot

O config padrão traz **3 interiores IPL** e **22 casas** espalhadas pelo mapa, cada uma com um routing bucket próprio (`600`–`621`). Casas diferentes podem reaproveitar o mesmo interior — o isolamento é feito pelo bucket.

Na inicialização o servidor:

- embaralha as listas de loot, de pickups e os itens de cada pool de recompensa;
- sorteia, por casa, quantos pontos de loot (`setup.loot`) e quantos pickups (`setup.pickups`) ficam ativos;
- sincroniza o estado resultante para todos os clientes (e para cada jogador que entra no servidor).

Reiniciar o recurso re-sorteia tudo e reseta as casas para fechadas.

---

## Integrações

### qbx_policejob

O alerta de roubo é enviado via `police:server:policeAlert`. As digitais deixadas nos pontos de loot usam `evidence:server:CreateFingerDrop`.

### qbx_interior

Entrada e saída disparam `qb-interior:client:screenfade` no cliente, produzindo o fade de tela durante o teleporte.

### Recurso de lockpick

A entrada depende inteiramente do evento `lockpicks:UseLockpick(playerSource, isAdvanced)`. Sem um recurso que dispare esse evento no servidor, não é possível arrombar nenhuma casa.

---

## Entrypoints para outros recursos

### Evento de entrada (obrigatório)

Emitido pelo recurso de lockpick, no servidor:

```lua
TriggerEvent('lockpicks:UseLockpick', playerSource, isAdvanced)
```

`isAdvanced = true` dispensa a checagem do `screwdriverset`.

### Eventos de servidor

```lua
-- Entrar numa casa já aberta (valida proximidade e estado)
TriggerServerEvent('qbx_houserobbery:server:enterHouse', houseIndex)

-- Sair do interior (valida proximidade do ponto de saída)
TriggerServerEvent('qbx_houserobbery:server:leaveHouse')
```

### Eventos de cliente

```lua
-- Sincroniza o estado das casas. Sem index, substitui a tabela inteira
TriggerClientEvent('qbx_houserobbery:client:syncconfig', source, data, index)
```

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`cs`, `en`, `nl`, `pt-br`

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_houserobbery/
├── client/
│   └── main.lua          — points de entrada/saída, loot, pickups, skillcheck, digitais
├── server/
│   └── main.lua          — validações, teleporte, routing bucket, sorteio e entrega de loot, alerta policial
├── config/
│   ├── client.lua        — janela de horário, chance de digital, modo de exibição do prompt
│   ├── server.lua        — polícia mínima, itens de entrada, pools de recompensa
│   └── shared.lua        — interiores (loot, pickups, skillcheck) e as 22 casas
├── locales/              — cs, en, nl, pt-br (.json)
└── fxmanifest.lua
```
