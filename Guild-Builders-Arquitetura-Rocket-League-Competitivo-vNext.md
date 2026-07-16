# Guild Builders — Arquitetura vNext do Rocket League Competitivo

> Documento técnico de referência da proposta atual.
>
> Este arquivo descreve o estado confirmado do projeto enviado, a arquitetura planejada, os contratos de dados, os fluxos de queue, partidas privadas, torneios, progressão, histórico, punições, logs e a estratégia de migração.
>
> **Status:** especificação de implementação.  
> **Projetos que receberão alterações:** `Guild-Builders-Discord-Bot` e `Guild-Builders-Painel`.  
> **Projetos usados apenas como referência de contrato:** `Guild-Builders-RL-Tracker-Api` e `Guild-Builders-RL-Tracker-Worker`.

---

## 1. Objetivo

Criar no Guild Builders um sistema completo de competições de Rocket League inspirado no fluxo do SixMans, mas mais genérico e configurável, com suporte a:

- queues privadas ranqueadas de `1v1`, `2v2`, `3v3` e `4v4`;
- torneios organizados pelo bot com chaveamento semelhante ao Rocket League;
- entrada individual ou por grupo;
- validação por conta Steam/Epic/PSN/Xbox conectada;
- restrições por rank, tier, divisão ou MMR;
- rating interno separado por modo e categoria;
- XP global dentro de cada guild, dividido em módulos;
- créditos internos da guild;
- histórico de XP, partidas, punições e auditoria;
- conquistas permanentes;
- cargos dinâmicos por posição no ranking;
- confirmação bilateral de resultados;
- resolução administrativa de disputas;
- punições progressivas configuráveis;
- logs configuráveis por evento;
- cache global de MMR compartilhável entre bots;
- preservação dos contratos legados do banco.

O sistema deverá ser modular para permitir, futuramente, que chat, voz e outros recursos usem a mesma infraestrutura de XP sem incrementar incorretamente as categorias do Rocket League.

---

# 2. Estado atual confirmado no ZIP

## 2.1 Projetos existentes

O ZIP possui quatro projetos:

```text
Guild-Builders-Project/
├── Guild-Builders-Discord-Bot
├── Guild-Builders-Painel
├── Guild-Builders-RL-Tracker-Api
└── Guild-Builders-RL-Tracker-Worker
```

A implementação nova deverá modificar somente:

```text
Guild-Builders-Discord-Bot
Guild-Builders-Painel
```

A API e o Worker são fontes de referência para entender os contratos de MMR, mas não devem ser alterados nesta implementação.

---

## 2.2 `IdentityStore` atual

Arquivo principal:

```text
Guild-Builders-Discord-Bot/src/database/modules/identityStore.ts
```

Instância atual:

```ts
export const IdentityStore = new IdentityStoreC(
  MongoConnection,
  Variables.StorageTargets.Identity
);
```

Atualmente o `IdentityStoreC` usa `ENV.BOT_ID` diretamente em várias operações:

```ts
{
  botId: ENV.BOT_ID,
  identifyId: id
}
```

Esse escopo é aplicado em:

- leitura e escrita das seções locais;
- leitura e escrita das seções remotas;
- cache em memória;
- cache em arquivo;
- documentos de `DBArray`;
- consultas do MongoDB;
- pending writes.

Isso significa que o contrato atual representa:

```text
bot atual + identifyId
```

Na maior parte do projeto, o `identifyId` é o ID da guild:

```text
botId = ENV.BOT_ID
identifyId = guildId
```

O `DBArray` também grava automaticamente:

```ts
{
  type: "dbarray",
  dbArrayName,
  botId: ENV.BOT_ID,
  identifyId,
  data,
  updatedAt
}
```

### Limitação

Não é possível criar corretamente um armazenamento:

```text
bot atual + global
```

ou:

```text
todos os bots + global
```

sem reformular a implementação-base, porque o `ENV.BOT_ID` está fixado dentro do serviço.

A solução não deve ser copiar o arquivo inteiro. A implementação-base deve receber um escopo configurável.

---

## 2.3 `DirectDb`

Arquivo:

```text
Guild-Builders-Discord-Bot/src/database/modules/directDb.ts
```

O `DirectDb` permite acesso menos opinativo ao MongoDB e poderá ser usado como base para repositórios atômicos.

O `DBArray` continua adequado para:

- configurações;
- definições de módulos;
- regras administrativas;
- objetos alterados com baixa concorrência.

O `DBArray` não deve ser usado como única proteção para:

- saldo de XP;
- créditos;
- entrada ativa em queue;
- rating;
- recompensas;
- resultado final;
- conquista única;
- punições que dependam de idempotência.

Esses dados exigem operações atômicas, índices únicos e, em alguns casos, transações.

---

## 2.4 Redis atual

Arquivo:

```text
Guild-Builders-Discord-Bot/src/sockets/redisClient.ts
```

Já existem clientes separados para:

- operações gerais;
- leitura bloqueante de streams;
- pub/sub.

O wrapper já suporta, entre outros:

- `get`;
- `set`;
- listas;
- hashes;
- streams;
- pub/sub.

Porém, o `set` atual não recebe TTL, `NX` ou `PX`.

Para a arquitetura nova, serão necessários métodos aditivos como:

```ts
setWithExpiration(key, value, ttlSeconds)
setIfAbsent(key, value, ttlMilliseconds)
delete(key)
ttl(key)
expire(key, ttlSeconds)
```

Esses métodos permitirão:

- cache temporário de MMR;
- locks distribuídos;
- ready check;
- sessões temporárias;
- reserva de jogadores;
- single-flight entre instâncias do bot;
- cooldowns.

---

## 2.5 Contas Rocket League atuais

Arquivo:

```text
Guild-Builders-Discord-Bot/src/services/rocketleague/account.ts
```

Os vínculos são salvos por guild em:

```text
collection: RocketLeague
dbArrayName: Users
```

Contrato atual relevante:

```ts
interface PlayerAccount {
  id: string;
  userId: string;

  preferredPlatform: {
    platform?: GamePlatforms;
  };

  plataforms: Partial<PlataformProfiles>;

  status:
    | "pedding"
    | "linked"
    | "unlinked"
    | "eligible"
    | "error";

  createdAt: number;
  updatedAt: number;
  rankUpdatedAt: number | null;
}
```

Campos legados que devem continuar sendo aceitos:

```text
plataforms
pedding
xbox
xbl
```

### Problemas confirmados no normalizador

O normalizador atual usa:

```ts
preferredPlatform: {
  platform: data.platform
}
```

O valor correto está normalmente em:

```ts
data.preferredPlatform.platform
```

Também existe:

```ts
rankUpdatedAt: Number(data.rankUpdatedAt ?? null)
```

Como `Number(null) === 0`, um valor inexistente pode virar zero em vez de permanecer `null`.

O painel possui `preferredPlatform.nickname`, enquanto o tipo correspondente do bot não possui esse campo.

Esses contratos precisam ser normalizados sem remover os nomes legados do banco.

---

## 2.6 MMR atual

Arquivo:

```text
Guild-Builders-Discord-Bot/src/services/rocketleague/mmrDataDb.ts
```

Cada guild salva uma cópia de:

```ts
interface MMRDataEntry {
  userId: string;
  platform: GamePlatforms;
  mmrData: PlayerMMRData | null;
  error?: MMRError;
  fetchedAt: number;
}
```

O armazenamento atual é:

```text
botId + guildId + userId + platform
```

Portanto, se a mesma conta estiver em várias guilds, o objeto grande de MMR pode ser duplicado várias vezes.

### Contrato completo de MMR

O contrato já contém:

```ts
interface PlayerMMRData {
  nickname: string;
  platform: GamePlatforms;
  gameModes: GameModeData[];
  peakMMR: number | null;
  peakByMode: Record<string, number | null>;
  currentMMR: number | null;
  snapshots: PlayerSnapshot[];
  lastUpdated: number;
  fetchedAt: string;
  trackerProfileUrl?: string;
  scrapedPages?: string[];
  globalSeasonHistory?: ModeSeasonHistoryEntry[];
  rawData?: unknown;
}
```

Cada modo possui dados como:

```ts
{
  mode,
  rank,
  mmr,
  peakMmr,
  peakRank,
  wins,
  losses,
  gamesPlayed,
  winPct,
  division
}
```

### Observação importante

`currentMMR` não representa necessariamente o MMR do modo que está sendo validado. As queues devem localizar o item correto de `gameModes`.

---

## 2.7 Configuração da Tracker API atual

Atualmente a configuração da API está dentro do snapshot da guild:

```ts
settings.ApiConfig = {
  Url: string,
  ApiKey: string
}
```

O painel possui campos para editar URL e chave.

Esse dado é infraestrutura do bot e não deveria ser configurável por guild nem enviado ao navegador.

Na arquitetura nova:

```text
RL_TRACKER_API_URL
RL_TRACKER_API_KEY
```

devem ficar no `.env` do bot.

O campo legado `settings.ApiConfig` deve continuar sendo aceito durante a migração, mas deve ser marcado como depreciado e ignorado pela nova implementação.

---

## 2.8 RL Tracker API como fonte persistente

O projeto da API possui índice único para:

```ts
{
  platform: 1,
  nickKey: 1
}
```

A API salva dados globais de Tracker e já possui deduplicação local de consultas simultâneas por:

```text
platform + nickKey
```

Ela deve continuar sendo a fonte persistente dos snapshots de MMR.

O bot adicionará um cache Redis compartilhado na frente da API.

Fluxo pretendido:

```text
Bot
 ↓
Redis compartilhado
 ↓ cache ausente ou vencido
RL Tracker API
 ↓
Mongo da Tracker API
 ↓ dado vencido
Worker
```

---

## 2.9 Automação de cargos atual

Arquivo principal:

```text
Guild-Builders-Discord-Bot/src/services/rocketleague/automation.ts
```

Hoje esse arquivo concentra muitas responsabilidades:

- cliente HTTP da Tracker API;
- resolução de conta;
- escolha de plataforma;
- cache de MMR por guild;
- busca de rank;
- resolução de playlist;
- aplicação de cargos;
- scheduler;
- progresso;
- logs.

A lógica de rank usa uma comparação textual semelhante a:

```ts
mmrData.gameModes.find(gm => gm.mode === playlistName)
```

Essa comparação é frágil.

Na arquitetura nova, os modos devem ter IDs canônicos e aliases.

---

## 2.10 Torneios atuais

Arquivos:

```text
src/services/rocketleague/tournaments/type.ts
src/services/rocketleague/tournaments/normalizer.ts
src/services/rocketleague/tournaments/service.ts
```

O sistema atual possui somente uma base administrativa:

```ts
interface Tournament {
  id: string;
  enabled: boolean;
  status:
    | "draft"
    | "awaitingentry"
    | "waitingcompletion"
    | "finished";

  name: string;
  entryStartDate: number | null;
  entryEndDate: number | null;
  tournamentStartDate: number | null;

  createdBy: string | null;
  createdAt: number;
  updatedAt: number;
}
```

Ainda não existem:

- inscrições;
- rosters;
- equipes;
- parties;
- check-in;
- partidas;
- rodadas;
- bracket real;
- resultado;
- confirmação;
- disputa;
- recompensa.

Os status legados precisam continuar válidos na leitura.

---

## 2.11 Painel atual

Arquivo principal:

```text
Guild-Builders-Painel/client/src/features/bots/tabs/RocketLeagueTab.tsx
```

A tela atual já reúne:

- configurações gerais;
- logs;
- API;
- contas;
- ranks;
- automação;
- torneios.

Ela já ultrapassa muitas responsabilidades e deverá ser dividida em componentes e hooks menores.

O painel usa comandos de socket para conversar com o bot. A nova implementação deve preservar o padrão atual de `sendBotCommand` e adicionar comandos por domínio.

---

# 3. Princípios da arquitetura nova

1. **Compatibilidade antes de limpeza.**  
   Campos e status legados devem continuar sendo aceitos.

2. **Escopo explícito.**  
   Todo dado deve declarar se pertence a:
   - bot + guild;
   - bot + global;
   - ecossistema global.

3. **Redis não é a fonte final de auditoria.**  
   Redis armazena estados temporários e cache.

4. **MongoDB é a fonte de dados persistentes.**

5. **Eventos financeiros e de progressão devem ser idempotentes.**

6. **O XP global da guild é derivado da soma dos módulos.**

7. **Rank oficial e rating interno são conceitos diferentes.**

8. **Partidas privadas e torneios devem compartilhar o mesmo núcleo de partidas.**

9. **A interface monta árvores; o banco não precisa salvar árvores recursivas.**

10. **A API e o Worker não serão modificados nesta versão.**

---

# 4. Novo sistema de escopos de identidade

## 4.1 Escopos necessários

### Escopo de guild

```text
botId = ENV.BOT_ID
identifyId = guildId
scopeType = guild
```

Usado para:

- configuração da guild;
- contas vinculadas dentro da guild;
- queues;
- torneios;
- XP;
- créditos;
- rating interno;
- punições;
- conquistas;
- cargos;
- logs.

### Global dentro do bot

```text
botId = ENV.BOT_ID
identifyId = "__global__"
scopeType = bot_global
```

Usado para:

- templates do bot;
- configurações compartilhadas entre guilds do mesmo bot;
- migrações do bot;
- logs globais do bot;
- defaults administrativos.

### Global entre todos os bots

```text
botId = "__global__"
identifyId = "__global__"
scopeType = ecosystem_global
```

Usado para:

- metadados realmente compartilhados;
- aliases de plataformas;
- referências de perfis Rocket League;
- estado de migrações compartilhadas;
- dados que todos os bots devem enxergar.

---

## 4.2 Configuração-base

A implementação-base deverá receber algo semelhante a:

```ts
type IdentityScopeType =
  | "guild"
  | "bot_global"
  | "ecosystem_global";

interface IdentityStoreScope {
  scopeType: IdentityScopeType;
  botId: string;
  fixedIdentifyId?: string;
  namespace: string;
}
```

Exemplo:

```ts
export const GuildIdentityStore = createIdentityStore({
  scopeType: "guild",
  botId: ENV.BOT_ID,
  namespace: "identity"
});

export const BotGlobalIdentityStore = createIdentityStore({
  scopeType: "bot_global",
  botId: ENV.BOT_ID,
  fixedIdentifyId: "__global__",
  namespace: "bot-global"
});

export const SharedGlobalIdentityStore = createIdentityStore({
  scopeType: "ecosystem_global",
  botId: "__global__",
  fixedIdentifyId: "__global__",
  namespace: "ecosystem-global"
});
```

Uso:

```ts
const guildDb = GuildIdentityStore.get(guildId);
const botGlobalDb = BotGlobalIdentityStore.get();
const sharedGlobalDb = SharedGlobalIdentityStore.get();
```

Não deve ser possível passar um `identifyId` arbitrário aos stores que possuem `fixedIdentifyId`.

---

## 4.3 Compatibilidade

O export legado deve continuar existindo:

```ts
export const IdentityStore = GuildIdentityStore;
```

Assim, chamadas atuais como:

```ts
IdentityStore.get(guildId)
```

continuam funcionando.

---

## 4.4 Coleções e database

É recomendável separar a coleção dos escopos globais:

```text
Identities
GlobalIdentities
```

Opção preferida:

```text
Identities
→ bot + guild

GlobalIdentities
→ bot + global
→ global + global
```

Todo documento global deve incluir:

```ts
{
  scopeType: "bot_global" | "ecosystem_global"
}
```

Também pode ser configurado um database compartilhado por `.env`, caso diferentes bots usem prefixes diferentes.

Exemplo:

```env
GLOBAL_MONGO_DATABASE=GB_GLOBAL
GLOBAL_MONGO_IDENTITY_COLLECTION=GlobalIdentities
```

---

## 4.5 Cache e pending writes

As chaves de cache em memória e em arquivo devem incluir:

```text
namespace
botId resolvido
identifyId resolvido
database
collection
section/dbArrayName
```

Isso impede colisão entre:

```text
bot global
ecossistema global
guild
```

---

# 5. Arquitetura global de MMR

## 5.1 Decisão

O objeto completo de MMR não deve continuar sendo salvo por guild.

Estado final:

```text
Redis compartilhado
→ cache temporário do bot

RL Tracker API
→ fonte persistente

MMRData legado por guild
→ fallback temporário de migração
```

O `SharedGlobalIdentityStore` pode manter metadados pequenos, mas não precisa duplicar todos os snapshots.

---

## 5.2 Identidade global do perfil

A chave do perfil deve ser baseada na conta externa:

```text
platform + nickname normalizado
```

Exemplos:

```text
epic:goaba
steam:76561198000000000
psn:goaba-psn
xbl:goaba-xbox
```

Contrato:

```ts
interface RocketLeagueProfileIdentity {
  profileKey: string;
  platform: GamePlatforms;
  nickname: string;
  nickKey: string;
}
```

Função:

```ts
resolveProfileKey({
  platform,
  nickname
});
```

A implementação deve normalizar:

- caixa;
- espaços;
- caracteres incompatíveis;
- aliases de plataforma;
- `xbox` para `xbl`.

---

## 5.3 Modos canônicos

Criar IDs estáveis:

```ts
type CompetitiveModeId =
  | "duel_1v1"
  | "doubles_2v2"
  | "standard_3v3"
  | "squad_4v4";
```

Aliases:

```ts
{
  "Ranked Duel 1v1": "duel_1v1",
  "Ranked Doubles 2v2": "doubles_2v2",
  "Ranked Standard 3v3": "standard_3v3"
}
```

Para `4v4`, como não existe um rank competitivo equivalente estável, a configuração da queue deve permitir:

```text
sem rank oficial
usar rank do 3v3
usar maior rank competitivo
usar rating interno 4v4
usar faixa manual de MMR
```

---

## 5.4 Cadeia de resolvers

```text
QueueEligibilityResolver
        ↓
RocketLeagueAccountResolver
        ↓
PlayerProfileResolver
        ↓
PlayerRankResolver
```

### `RocketLeagueAccountResolver`

Entrada:

```ts
{
  guildId,
  userId,
  requestedPlatform?
}
```

Saída:

```ts
{
  userId,
  platform,
  nickname,
  profileKey,
  linkedAccount
}
```

Responsabilidades:

- carregar a conta da guild;
- corrigir aliases;
- definir a plataforma preferida;
- validar se existe nickname;
- nunca confiar em um `userId` arbitrário vindo do navegador quando o comando é público.

### `PlayerProfileResolver`

Ordem:

```text
1. Redis global
2. Tracker API
3. MMRData legado da guild, enquanto a migração estiver ativa
```

Entrada:

```ts
{
  platform,
  nickname,
  freshness: {
    maxAgeMs,
    allowStaleOnError,
    maxStaleAgeMs
  },
  forceRefresh?
}
```

Saída:

```ts
interface ResolvedPlayerProfile {
  profileKey: string;
  platform: GamePlatforms;
  nickname: string;
  mmrData: PlayerMMRData;

  source:
    | "redis"
    | "tracker_api"
    | "legacy_guild_cache";

  stale: boolean;
  fetchedAt: number;
  resolvedAt: number;
}
```

### `PlayerRankResolver`

Entrada:

```ts
{
  profile,
  mode,
  rankSourcePolicy
}
```

Saída:

```ts
{
  mode,
  playlistId,
  mmr,
  tier,
  division,
  rankName
}
```

### `QueueEligibilityResolver`

Entrada:

```ts
{
  guildId,
  userId,
  queue
}
```

Saída:

```ts
{
  eligible,
  reasons,
  account,
  rank,
  snapshotAt,
  rulesVersion
}
```

---

## 5.5 Redis

Namespace sugerido:

```text
gb:rocketleague:profile:v1:<platform>:<nickKey>
gb:rocketleague:profile-lock:v1:<platform>:<nickKey>
gb:rocketleague:error:v1:<platform>:<nickKey>
```

O valor de perfil deve possuir:

```ts
{
  schemaVersion: 1,
  profileKey,
  platform,
  nickKey,
  nickname,
  mmrData,
  fetchedAt,
  source
}
```

---

## 5.6 Single-flight

Devem existir duas proteções:

### Local

```ts
Map<string, Promise<ResolvedPlayerProfile>>
```

Consultas na mesma instância aguardam a mesma Promise.

### Distribuída

```text
SET lockKey lockId NX PX 30000
```

Outras instâncias aguardam o preenchimento do cache ou tentam novamente após o lock expirar.

---

## 5.7 Freshness

A validade depende do uso.

Exemplos iniciais:

```text
visualização de perfil: 30 minutos
queue: 15 minutos
automação de cargos: configurável
cache de erro: 60 segundos
```

Uma queue pode configurar:

```ts
{
  maxRankDataAgeMinutes: 60,
  allowStaleOnApiError: true,
  maximumStaleAgeMinutes: 360
}
```

---

## 5.8 Configuração da API no `.env`

Adicionar ao bot:

```env
RL_TRACKER_API_URL=http://localhost:3001
RL_TRACKER_API_KEY=
RL_TRACKER_API_TIMEOUT_MS=120000
RL_TRACKER_CACHE_TTL_SECONDS=1800
RL_TRACKER_ERROR_CACHE_TTL_SECONDS=60
RL_TRACKER_ALLOW_LEGACY_FALLBACK=true
```

O painel não deve visualizar a chave.

O campo legado deve continuar aceito:

```ts
/**
 * @deprecated
 * A configuração real agora vem do ambiente do bot.
 */
ApiConfig?: {
  Url: string;
  ApiKey: string;
}
```

---

## 5.9 Migração do MMR

### Fase 1 — leitura híbrida

```text
Redis → API → MMRData legado
```

### Fase 2 — parar gravação legada

Nenhuma consulta nova grava o objeto completo por guild.

### Fase 3 — todos os consumidores usam resolver

Migrar:

- automação de cargos;
- comandos de usuário;
- painel;
- queues;
- torneios.

### Fase 4 — remover fallback

Somente após uma versão estável.

### Fase 5 — limpeza opcional

Criar script separado. Não apagar automaticamente durante deploy.

---

# 6. Sistema de XP por guild

## 6.1 Significado de “global”

O XP é global dentro da guild:

```text
botId = ENV.BOT_ID
identifyId = guildId
```

Cada guild possui sua própria progressão.

Se futuramente for necessário um XP real do usuário em todas as guilds, ele poderá ser calculado agregando os resumos de várias guilds.

---

## 6.2 Árvore de módulos

A estrutura conceitual pode ser:

```text
Rocket League
├── Partidas privadas
│   ├── 1v1
│   ├── 2v2
│   ├── 3v3
│   └── 4v4
└── Torneios
    ├── 1v1
    ├── 2v2
    ├── 3v3
    └── 4v4

Chat
Voz
Outros
```

O banco deve salvar definições planas com `parentId`.

```ts
interface ProgressionModuleDefinition {
  id: string;
  parentId: string | null;

  name: string;
  description?: string;

  type: "module" | "activity";

  enabled: boolean;
  order: number;

  schemaVersion: 1;
}
```

Exemplos:

```ts
{
  id: "rocketleague",
  parentId: null,
  name: "Rocket League",
  type: "module"
}
```

```ts
{
  id: "rocketleague.private",
  parentId: "rocketleague",
  name: "Partidas privadas",
  type: "module"
}
```

```ts
{
  id: "rocketleague.private.2v2",
  parentId: "rocketleague.private",
  name: "Privadas 2v2",
  type: "activity"
}
```

A interface monta a árvore recursiva.

---

## 6.3 Saldo por módulo

```ts
interface UserProgressionBalance {
  botId: string;
  identifyId: string;

  userId: string;
  moduleId: string;

  xp: number;

  createdAt: number;
  updatedAt: number;
  schemaVersion: 1;
}
```

Índice único:

```ts
{
  botId: 1,
  identifyId: 1,
  userId: 1,
  moduleId: 1
}
```

Atualização:

```ts
$inc: {
  xp: amount
}
```

Não usar o fluxo não atômico de ler, somar em memória e salvar.

---

## 6.4 Cálculo dos pais

Ao ganhar 100 XP em:

```text
rocketleague.private.2v2
```

Salvar 100 somente naquele módulo.

Não salvar também 100 em todos os pais.

O total é calculado:

```text
Privadas = selfXp + descendentes
Rocket League = selfXp + descendentes
Global da guild = soma de todos os XP diretos
```

A resposta para o painel pode conter:

```ts
{
  id: "rocketleague.private",
  selfXp: 0,
  totalXp: 1400,
  children: []
}
```

---

## 6.5 Histórico de XP

```ts
interface ProgressionEvent {
  id: string;
  idempotencyKey: string;

  botId: string;
  identifyId: string;

  userId: string;
  moduleId: string;

  operation:
    | "gain"
    | "remove"
    | "adjust"
    | "reversal";

  xpDelta: number;

  source: {
    type:
      | "private_match"
      | "tournament_match"
      | "tournament_prize"
      | "chat"
      | "voice"
      | "admin";

    matchId?: string;
    tournamentId?: string;
    reason?: string;
  };

  createdAt: number;
  createdBy?: string;
  schemaVersion: 1;
}
```

Índice único:

```text
idempotencyKey
```

Exemplo:

```text
match:<matchId>:user:<userId>:xp:v1
```

A interface deve listar:

- últimos 5 por padrão;
- opção de 20;
- paginação para histórico completo.

O histórico não deve ser truncado no banco.

Para chat e voz, eventos podem ser agregados por janela para evitar um documento por mensagem.

---

## 6.6 Resumo global da guild

```ts
interface UserProgressionSummary {
  botId: string;
  identifyId: string;
  userId: string;

  globalXp: number;
  globalLevel: number;

  updatedAt: number;
  schemaVersion: 1;
}
```

Serve para:

- perfil;
- leaderboard;
- top global.

Pode ser reconstruído a partir dos balances.

---

# 7. Créditos

Os créditos são globais dentro da guild nesta fase.

```ts
interface CreditTransaction {
  id: string;
  idempotencyKey: string;

  botId: string;
  identifyId: string;
  userId: string;

  amount: number;
  operation: "credit" | "debit" | "reversal";

  reason:
    | "private_match_win"
    | "private_match_loss"
    | "tournament_prize"
    | "achievement"
    | "admin"
    | "shop_purchase";

  provider: "internal" | "ay_payments";

  matchId?: string;
  tournamentId?: string;

  createdAt: number;
  schemaVersion: 1;
}
```

O saldo deve ser atômico e auditável.

Quando AY Payments estiver pronta:

```text
provider = ay_payments
```

poderá ser integrado sem substituir o contrato de recompensa.

O XP pode ser agregado em vários níveis de visualização. O crédito deve ser entregue uma única vez por evento.

---

# 8. Rating competitivo interno

O rank oficial serve para:

- elegibilidade;
- seed inicial;
- restrição de queue;
- balanceamento inicial opcional.

O rating interno serve para:

- matchmaking;
- leaderboard;
- evolução nas privadas;
- ranking por modo;
- temporadas.

```ts
interface CompetitionRating {
  botId: string;
  identifyId: string;

  userId: string;
  category: "private_ranked" | "tournament";
  mode: "1v1" | "2v2" | "3v3" | "4v4";
  seasonId: string;

  rating: number;
  gamesPlayed: number;
  wins: number;
  losses: number;

  updatedAt: number;
  schemaVersion: 1;
}
```

Índice único:

```text
botId + identifyId + userId + category + mode + seasonId
```

---

# 9. Queues privadas ranqueadas

## 9.1 Modos

```text
1v1 → 2 jogadores
2v2 → 4 jogadores
3v3 → 6 jogadores
4v4 → 8 jogadores
```

---

## 9.2 Configuração

```ts
interface PrivateQueueDefinition {
  id: string;
  enabled: boolean;

  name: string;
  description?: string;

  mode: "1v1" | "2v2" | "3v3" | "4v4";
  teamSize: number;
  totalPlayers: number;

  eligibility: {
    access:
      | "everyone"
      | "linked_account"
      | "valid_rank"
      | "rank_range"
      | "mmr_range"
      | "roles";

    requireConnectedAccount: boolean;
    requireValidRankData: boolean;

    rankSource:
      | "same_mode"
      | "standard_3v3"
      | "highest_competitive"
      | "internal_rating"
      | "none";

    minTier?: number | null;
    maxTier?: number | null;
    minMMR?: number | null;
    maxMMR?: number | null;

    requiredRoleIds?: string[];
    blockedRoleIds?: string[];

    maxRankDataAgeMinutes: number;
    allowStaleOnApiError: boolean;
    maximumStaleAgeMinutes: number;
  };

  partyPolicy: {
    allowPremadeParties: boolean;
    allowPartialParties: boolean;
    keepPartyTogether: boolean;
    maximumPartySize: number;
  };

  matchmaking: {
    type:
      | "random"
      | "official_rank"
      | "internal_rating"
      | "hybrid";

    initialRatingDifference: number;
    expandDifferenceEverySeconds: number;
    maximumDifference: number;
  };

  readyCheck: {
    enabled: boolean;
    timeoutSeconds: number;
  };

  rewards: {
    winnerXp: number;
    loserXp: number;
    winnerCredits: number;
    loserCredits: number;
  };

  resultPolicyId: string;
  punishmentPolicyId: string;

  schemaVersion: 1;
}
```

---

## 9.3 Entrada na queue

A entrada não deve contar imediatamente.

Estados:

```ts
type QueueEntryStatus =
  | "resolving"
  | "eligible"
  | "rejected"
  | "resolution_failed"
  | "reserved"
  | "matched"
  | "left"
  | "expired";
```

Fluxo:

```text
Usuário entra
  ↓
resolving
  ↓
resolver conta
  ↓
resolver cache/API
  ↓
validar rank e punições
  ↓
eligible ou rejected
```

Somente `eligible` conta para completar a queue.

A interface pode exibir:

```text
✅ @Jogador — Champion I
⏳ @Jogador — Verificando rank...
❌ @Jogador — Rank fora da faixa
```

---

## 9.4 Snapshot de elegibilidade

```ts
interface EligibilitySnapshot {
  profileKey: string;
  platform: GamePlatforms;

  mode: string;
  playlistId?: number;

  mmr?: number | null;
  tier?: number | null;
  division?: number | null;

  officialRankName?: string | null;
  internalRating?: number | null;

  profileFetchedAt: number;
  resolvedAt: number;
  rulesVersion: number;
}
```

A listagem normal da queue usa o snapshot. Ela não deve chamar a API para todos a cada renderização.

Antes de criar a partida, atualizar somente snapshots vencidos.

---

## 9.5 Concorrência

Garantias necessárias:

- uma entrada ativa por usuário e guild;
- usuário não pode estar em duas queues incompatíveis;
- party não pode ser reservada duas vezes;
- jogadores reservados não podem formar duas partidas;
- criação da partida deve ser idempotente.

Usar:

- índice único parcial;
- lock Redis;
- operação condicional no Mongo;
- chave de idempotência.

---

# 10. Parties e grupos

## 10.1 Contrato

```ts
interface CompetitionParty {
  id: string;

  botId: string;
  guildId: string;

  mode: "1v1" | "2v2" | "3v3" | "4v4";

  leaderId: string;

  members: Array<{
    userId: string;
    status:
      | "invited"
      | "accepted"
      | "declined"
      | "left";
  }>;

  status:
    | "forming"
    | "ready"
    | "queued"
    | "in_match"
    | "disbanded"
    | "expired";

  createdAt: number;
  updatedAt: number;
  expiresAt?: number;

  schemaVersion: 1;
}
```

Quem cria é o líder inicial.

O líder pode:

- convidar;
- cancelar convite;
- remover membro;
- sair;
- transferir liderança;
- entrar na queue;
- retirar da queue.

A transferência exige aceite do novo líder.

---

## 10.2 Parties parciais

Em uma queue 3v3, podem existir:

```text
party de 3
party de 2 + solo
3 solos
```

Se `keepPartyTogether` estiver ativo, o balanceador não separa a party.

---

## 10.3 Manter time após a partida

Configuração:

```ts
{
  enabled: true,
  timeoutSeconds: 45,

  exactTeamRequiresUnanimous: true,

  allowPartialPartyFromYesVotes: true,
  minimumPartialPartySize: 2
}
```

### Dupla

Dois votos “sim” são obrigatórios.

### Trio/quarteto

Todos devem votar “sim” para manter exatamente o time.

Se apenas parte votar “sim” e parties parciais forem permitidas, criar um novo grupo somente com os jogadores que aceitaram.

Ninguém pode ser forçado a continuar por decisão da maioria.

---

# 11. Capitães

Party leader e capitão são conceitos diferentes:

```text
Party leader
→ gerencia o grupo antes da partida

Team captain
→ representa o time na partida
```

Políticas:

```ts
type CaptainSelectionMode =
  | "party_leader"
  | "vote"
  | "highest_official_rank"
  | "highest_internal_rating"
  | "random"
  | "volunteer";
```

Configuração:

```ts
{
  primary: "vote",
  timeoutSeconds: 60,

  fallbackOrder: [
    "highest_internal_rating",
    "highest_official_rank",
    "random"
  ]
}
```

Votação:

```text
maioria absoluta
↓
se não houver até o timeout:
mais votos
↓
empate:
fallback
```

Em party pronta, o líder pode ser capitão por padrão.

---

# 12. Núcleo de partidas

## 12.1 Origem

```ts
type MatchSource =
  | "private_ranked"
  | "tournament";
```

---

## 12.2 Estados

```ts
type CompetitionMatchStatus =
  | "forming"
  | "validating_players"
  | "ready_check"
  | "team_selection"
  | "lobby_ready"
  | "in_progress"
  | "result_pending"
  | "disputed"
  | "admin_review"
  | "completed"
  | "cancelled"
  | "expired";
```

---

## 12.3 Contrato

```ts
interface CompetitionMatch {
  id: string;

  botId: string;
  guildId: string;

  source: MatchSource;
  mode: "1v1" | "2v2" | "3v3" | "4v4";

  queueId?: string;
  tournamentId?: string;
  tournamentMatchId?: string;

  status: CompetitionMatchStatus;

  blueTeam: MatchTeam;
  orangeTeam: MatchTeam;

  lobby?: {
    name: string;
    password: string;
  };

  resultReports: MatchResultReport[];

  result?: {
    winner: "blue" | "orange";
    blueScore?: number;
    orangeScore?: number;

    confirmationMethod: string;
    confirmedAt: number;
    confirmedBy?: string;
  };

  createdAt: number;
  startedAt?: number;
  completedAt?: number;

  schemaVersion: 1;
}
```

---

## 12.4 Histórico

O documento final da partida deve ser mantido.

O perfil poderá mostrar:

```text
Vitória — Privada 2v2 — 3 x 1
Derrota — Torneio 3v3 — 1 x 2
```

O histórico permite calcular:

- partidas;
- vitórias;
- derrotas;
- win rate;
- sequência;
- desempenho por modo;
- desempenho por categoria;
- adversários recorrentes;
- torneios vencidos.

---

# 13. Resultado e confirmação

## 13.1 Políticas

```ts
type ResultConfirmationMode =
  | "dual_captain_submission"
  | "captain_report_opponent_confirm"
  | "player_majority"
  | "authorized_roles"
  | "authorized_users"
  | "hybrid";
```

---

## 13.2 Padrão recomendado para privadas

```text
dual_captain_submission
```

Cada capitão envia independentemente:

```ts
{
  team: "blue",
  submittedBy: "userId",
  winner: "blue",
  blueScore: 3,
  orangeScore: 1,
  submittedAt: number
}
```

Comparação:

```text
mesmo vencedor + mesmo placar
→ completed

mesmo vencedor + placar diferente
→ conflito de placar

vencedores diferentes
→ disputed

um não responde
→ timeout
```

---

## 13.3 Staff autorizado

Configuração:

```ts
{
  allowedRoleIds: [],
  allowedUserIds: [],
  allowAdministrators: true,
  staffCanOverride: true
}
```

Staff pode:

- informar;
- confirmar;
- corrigir;
- resolver disputa;
- aplicar forfeit;
- cancelar;
- reverter recompensa;
- remover punição.

---

## 13.4 Timeout

```ts
{
  reportTimeoutSeconds: 600,
  reminderIntervalsSeconds: [120, 300, 480],

  timeoutAction:
    | "send_to_admin_review"
    | "cancel_match"
    | "accept_existing_report"
    | "apply_double_forfeit"
}
```

Padrão seguro:

```text
punir ausência + enviar para revisão
```

Não aceitar automaticamente um único report como padrão.

---

# 14. Punições

## 14.1 Infrações

```ts
interface CompetitionInfraction {
  id: string;

  botId: string;
  guildId: string;
  userId: string;

  reason:
    | "ready_check_timeout"
    | "match_no_show"
    | "match_abandonment"
    | "result_not_submitted"
    | "false_result"
    | "repeated_dispute"
    | "unsportsmanlike_behavior"
    | "admin_action";

  points: number;

  matchId?: string;
  tournamentId?: string;

  createdAt: number;
  expiresAt?: number;

  revokedAt?: number;
  revokedBy?: string;
  revokeReason?: string;

  schemaVersion: 1;
}
```

---

## 14.2 Ações

```text
aviso
pontos
cooldown
suspensão de queue
suspensão de torneio
suspensão de todo o competitivo
perda de créditos
perda de XP
derrota por abandono
redução opcional de rating
remoção de cargos
timeout Discord
kick
ban
```

Timeout, kick e ban devem ficar desativados por padrão e exigir confirmação explícita no painel.

---

## 14.3 Escalonamento

```ts
{
  windowDays: 30,
  resetAfterCleanDays: 14,

  thresholds: [
    {
      points: 2,
      action: {
        type: "queue_cooldown",
        durationSeconds: 3600
      }
    },
    {
      points: 4,
      action: {
        type: "competition_suspension",
        durationSeconds: 86400
      }
    },
    {
      points: 6,
      action: {
        type: "competition_suspension",
        durationSeconds: 604800
      }
    }
  ]
}
```

Punição disciplinar e resultado competitivo devem ser separados.

---

# 15. Torneios

## 15.1 Modos

```text
1v1
2v2
3v3
4v4
```

---

## 15.2 Primeira versão

Suportar:

- eliminação simples;
- equipe pronta;
- inscrição individual;
- modo híbrido;
- check-in;
- seed aleatório;
- seed por rank oficial;
- seed por rating interno;
- BO1, BO3, BO5 e BO7;
- premiação por colocação;
- staff autorizado;
- chaveamento visual.

Depois:

- eliminação dupla;
- grupos;
- round robin;
- lower bracket;
- torneios recorrentes.

---

## 15.3 Inscrição

```ts
type TournamentRegistrationMode =
  | "premade_only"
  | "solo_only"
  | "hybrid";
```

Uma inscrição deve manter snapshot do roster e da elegibilidade.

---

## 15.4 Partidas de bracket

Cada partida deve ser um documento separado:

```ts
interface TournamentBracketMatch {
  id: string;
  tournamentId: string;

  round: number;
  position: number;

  blueEntryId?: string;
  orangeEntryId?: string;

  nextMatchId?: string;
  nextSlot?: "blue" | "orange";

  bestOf: 1 | 3 | 5 | 7;

  competitionMatchId?: string;

  status:
    | "pending"
    | "ready"
    | "in_progress"
    | "completed"
    | "cancelled";

  schemaVersion: 1;
}
```

O vencedor deve avançar atomicamente.

---

## 15.5 Compatibilidade de status

Status legados:

```text
draft
awaitingentry
waitingcompletion
finished
```

Devem continuar aceitos.

Internamente, podem ser mapeados para fases canônicas:

```text
draft
registration_open
checkin
running
finished
cancelled
```

Não reescrever documentos antigos automaticamente.

---

# 16. Logs

## 16.1 Event bus

Serviços não devem enviar diretamente para canais.

Eles emitem eventos:

```ts
competitionEvents.emit("match.completed", payload);
```

Um router decide:

- se salva;
- se envia;
- qual canal;
- qual template;
- qual nível.

---

## 16.2 Eventos

```text
queue.player_joined
queue.player_left
queue.player_rejected
queue.player_resolution_failed
queue.ready_check_failed
queue.match_found

party.created
party.invited
party.member_joined
party.member_left
party.disbanded

match.created
match.started
match.result_submitted
match.result_confirmed
match.result_disputed
match.completed
match.cancelled

tournament.registration_created
tournament.team_registered
tournament.checkin_failed
tournament.match_completed
tournament.champion_defined

punishment.created
punishment.escalated
punishment.expired
punishment.revoked

progression.xp_awarded
progression.credits_awarded
progression.achievement_unlocked

system.error
```

---

## 16.3 Rotas

```ts
interface RocketLeagueLogsConfig {
  enabled: boolean;

  defaults: {
    auditChannelId?: string;
    errorChannelId?: string;
    queueChannelId?: string;
    matchChannelId?: string;
    tournamentChannelId?: string;
    punishmentChannelId?: string;
    progressionChannelId?: string;
  };

  routes: Array<{
    event: string;
    enabled: boolean;
    channelId?: string;
  }>;
}
```

Prioridade:

```text
rota específica
↓
canal padrão da categoria
↓
auditChannelId
↓
sem mensagem no Discord
```

Mesmo sem canal, eventos importantes continuam no banco.

Erros sem guild usam configuração do bot global ou `.env`.

---

# 17. Conquistas e cargos

## 17.1 Conquistas permanentes

Exemplos:

```text
primeira vitória
10 vitórias 2v2
100 partidas privadas
campeão de torneio 3v3
5 torneios vencidos
```

```ts
interface AchievementDefinition {
  id: string;
  name: string;
  description: string;

  condition: AchievementCondition;

  reward?: {
    xp?: number;
    credits?: number;
    roleId?: string;
  };

  enabled: boolean;
}
```

Índice único para desbloqueio:

```text
guild + user + achievementId
```

---

## 17.2 Cargos dinâmicos

Exemplos:

```text
Top 1 global
Top 10 2v2
Top 25 torneios
```

Esses cargos são recalculados e removidos quando o jogador deixa a posição.

---

# 18. Estrutura de pastas proposta

## 18.1 Bot

```text
src/
├── database/
│   ├── modules/
│   │   ├── identityStore.ts
│   │   ├── identityScope.ts
│   │   └── directDb.ts
│   └── repositories/
│       └── scopedAtomicRepository.ts
│
├── services/
│   ├── progression/
│   │   ├── modules/
│   │   ├── balances/
│   │   ├── events/
│   │   ├── summaries/
│   │   ├── credits/
│   │   ├── achievements/
│   │   └── progression.service.ts
│   │
│   └── rocketleague/
│       ├── index.ts
│       │
│       ├── config/
│       │   ├── rocket-league-config.service.ts
│       │   └── tracker-api-config.resolver.ts
│       │
│       ├── accounts/
│       │   ├── account.repository.ts
│       │   ├── account.service.ts
│       │   ├── account.normalizer.ts
│       │   └── account.resolver.ts
│       │
│       ├── profiles/
│       │   ├── profile.types.ts
│       │   ├── profile-key.resolver.ts
│       │   ├── player-profile.resolver.ts
│       │   ├── tracker-api.client.ts
│       │   ├── redis-profile-cache.repository.ts
│       │   └── legacy-mmr.repository.ts
│       │
│       ├── ranks/
│       │   ├── rank.repository.ts
│       │   ├── playlist.resolver.ts
│       │   ├── player-rank.resolver.ts
│       │   └── rank-range.resolver.ts
│       │
│       ├── automation/
│       │   ├── role-automation.service.ts
│       │   ├── rank-sync.service.ts
│       │   ├── rank-sync.scheduler.ts
│       │   └── rank-sync-progress.service.ts
│       │
│       ├── parties/
│       ├── queues/
│       ├── matchmaking/
│       ├── matches/
│       ├── results/
│       ├── punishments/
│       ├── logs/
│       ├── achievements/
│       └── tournaments/
│
└── types/
    ├── progression/
    └── features/rocketleague/
```

Não criar uma única pasta global `src/resolvers` com todos os domínios misturados.

---

## 18.2 Painel

```text
client/src/features/rocketleague/
├── api/
├── components/
│   ├── settings/
│   ├── accounts/
│   ├── automation/
│   ├── queues/
│   ├── parties/
│   ├── matches/
│   ├── tournaments/
│   ├── progression/
│   ├── punishments/
│   └── logs/
│
├── hooks/
├── pages/
├── schemas/
├── types/
└── utils/
```

A `RocketLeagueTab.tsx` deve virar um componente de composição, não continuar concentrando todas as regras.

---

# 19. Persistência e índices

Coleções sugeridas:

```text
ProgressionModules
ProgressionBalances
ProgressionEvents
ProgressionSummaries
CreditBalances
CreditTransactions

RocketLeagueQueues
RocketLeagueQueueEntries
RocketLeagueParties
RocketLeagueMatches
RocketLeagueMatchReports
RocketLeagueRatings
RocketLeagueInfractions
RocketLeaguePunishmentStates

RocketLeagueTournamentEntries
RocketLeagueTournamentMatches

AchievementDefinitions
UserAchievements

RocketLeagueAuditEvents
```

Índices mínimos:

```text
uma entrada ativa por usuário/guild
um rating por usuário/categoria/modo/temporada
um saldo de XP por usuário/módulo
um evento por idempotencyKey
uma transação de crédito por idempotencyKey
uma recompensa por partida/usuário/tipo
uma conquista por guild/usuário/conquista
um resultado final por partida
```

---

# 20. Segurança

1. Comandos públicos não devem confiar em `userId` enviado pelo navegador.
2. O servidor/painel deve vincular a operação à sessão autenticada.
3. Comandos administrativos continuam exigindo `ensureAdmin`.
4. Chaves da Tracker API não chegam ao cliente.
5. Ações destrutivas devem ser auditadas.
6. Staff override deve registrar autor, data e motivo.
7. Reversões de XP, crédito e resultado devem gerar novos eventos, não apagar silenciosamente o histórico.

---

# 21. Estratégia de implementação

## Fase 0 — testes de regressão

Antes da refatoração:

- normalização de conta;
- IdentityStore;
- DBArray;
- configuração;
- torneios legados;
- automação de cargo.

## Fase 1 — escopos e contratos

- IdentityStore configurável;
- stores globais;
- compatibilidade do export;
- correções de conta;
- aliases;
- schemaVersion.

## Fase 2 — resolver global de MMR

- cliente da API;
- `.env`;
- Redis TTL/NX;
- profile resolver;
- rank resolver;
- fallback legado.

## Fase 3 — migração da automação

- automação usa resolver;
- parar gravação de MMR por guild;
- painel remove API config;
- comandos usam resolver.

## Fase 4 — progressão

- módulos;
- balances;
- eventos;
- histórico;
- resumos;
- créditos;
- painel.

## Fase 5 — núcleo competitivo

- party;
- queue;
- elegibilidade;
- matchmaking;
- partida;
- ready check;
- capitão;
- resultado;
- disputa;
- punições.

## Fase 6 — torneios

- inscrição;
- check-in;
- bracket;
- partidas;
- premiação.

## Fase 7 — conquistas, cargos e temporadas

- conquistas;
- leaderboard;
- cargos dinâmicos;
- temporada.

## Fase 8 — limpeza

- remover uso do MMR legado;
- script opcional;
- remover UI antiga;
- documentação final.

---

# 22. Por que a implementação é grande

O projeto não está apenas adicionando um botão de “entrar na fila”.

A implementação envolve:

- refatorar o armazenamento sem quebrar dados existentes;
- remover duplicação de MMR;
- compartilhar cache entre bots;
- proteger operações concorrentes;
- criar progressão auditável;
- criar um núcleo de partidas;
- decidir resultados de forma segura;
- lidar com abandono e fraude;
- montar parties e capitães;
- criar torneios e bracket;
- criar dezenas de configurações administrativas;
- dividir uma tela grande do painel;
- manter compatibilidade com bancos já em uso.

Fazer essas etapas de forma gradual reduz o risco de:

- perder dados;
- duplicar recompensas;
- iniciar duas partidas com os mesmos jogadores;
- aplicar cargo errado;
- quebrar vínculos existentes;
- expor chaves;
- deixar resultados sem auditoria.

---

# 23. Defaults recomendados

```text
Escopo do XP:
bot + guild

MMR:
Redis compartilhado + Tracker API

Queue:
snapshot de rank de até 60 minutos

Resultado de privada:
dois capitães enviam independentemente

Resultado de torneio:
capitães + override de staff

Timeout:
10 minutos

Ação no timeout:
punição + revisão administrativa

Capitão de time aleatório:
votação, depois maior rating interno

Stay together:
unanimidade para time completo

Party parcial:
permitida somente se configurada

Kick/ban Discord:
desativado por padrão

Histórico visível:
5 itens, opção para 20 e paginação
```

---

# 24. Resultado esperado

Ao concluir a arquitetura, o Guild Builders terá:

```text
um único núcleo competitivo
├── privadas ranqueadas
└── torneios

progressão por guild
├── XP modular
├── créditos
├── histórico
└── conquistas

dados globais de Rocket League
├── cache Redis compartilhado
└── Tracker API persistente

administração
├── logs por evento
├── punições
├── staff
├── regras de acesso
└── auditoria
```

O sistema continuará compatível com os dados antigos enquanto migra gradualmente para os contratos novos.
