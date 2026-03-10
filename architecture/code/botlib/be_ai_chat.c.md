# code/botlib/be_ai_chat.c

## File Purpose
Implements the bot chat AI subsystem for Quake III Arena, managing bot console message queues, chat line selection, synonym/random-string expansion, match-template pattern matching, and reply-chat key evaluation. It provides the complete pipeline from raw console input through pattern matching to final chat message construction and delivery.

## Core Responsibilities
- Manage a fixed-size heap of `bot_consolemessage_t` nodes for per-bot console message queues
- Load and parse synonym, random-string, match-template, and reply-chat data files
- Match incoming strings against loaded `bot_matchtemplate_t` patterns and extract named variables
- Select and construct initial chat messages by type, with recency-avoidance logic
- Evaluate reply-chat key sets (AND/NOT/gender/name/string/variable) to choose best-priority reply
- Expand escape-coded chat message templates (`\x01v...\x01`, `\x01r...\x01`) with variable and random substitutions
- Deliver constructed messages via `EA_Command` (say / say_team / tell)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_consolemessage_t` | struct | Doubly-linked node for a single console message visible to a bot |
| `bot_chatstate_t` | struct | Per-bot state: name, gender, client, console message queue, loaded chat data, pending output message |
| `bot_ichatdata_t` | struct | Cache entry pairing a loaded `bot_chat_t *` with its filename and chatname |
| `bot_chat_t` | struct | Root of loaded initial-chat data; owns a linked list of `bot_chattype_t` |
| `bot_chattype_t` | struct | Named category of chat lines (e.g., "death", "kill") with a list of `bot_chatmessage_t` |
| `bot_chatmessage_t` | struct | Single chat message string with last-used timestamp |
| `bot_synonymlist_t` / `bot_synonym_t` | struct | Context-tagged list of weighted synonyms for word substitution |
| `bot_randomlist_t` / `bot_randomstring_t` | struct | Named pool of random replacement strings |
| `bot_matchtemplate_t` | struct | Context-tagged pattern template composed of `bot_matchpiece_t` pieces |
| `bot_matchpiece_t` | struct | One piece of a match template: either `MT_STRING` (one or more alternatives) or `MT_VARIABLE` |
| `bot_replychat_t` / `bot_replychatkey_t` | struct | Priority-ranked reply rule with a key set and associated chat messages |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botchatstates` | `bot_chatstate_t *[MAX_CLIENTS+1]` | global | Per-handle chat state pointers (index 1-based) |
| `ichatdata` | `bot_ichatdata_t *[MAX_CLIENTS]` | global | Cache of loaded initial-chat data to avoid reloading |
| `consolemessageheap` | `bot_consolemessage_t *` | global | Base pointer of the pre-allocated console message pool |
| `freeconsolemessages` | `bot_consolemessage_t *` | global | Head of the free-list within the heap |
| `matchtemplates` | `bot_matchtemplate_t *` | global | Loaded match template list (from `match.c`) |
| `synonyms` | `bot_synonymlist_t *` | global | Loaded synonym list (from `syn.c`) |
| `randomstrings` | `bot_randomlist_t *` | global | Loaded random string pool (from `rnd.c`) |
| `replychats` | `bot_replychat_t *` | global | Loaded reply-chat rules (from `rchat.c`) |

## Key Functions / Methods

### BotSetupChatAI
- **Signature:** `int BotSetupChatAI(void)`
- **Purpose:** Initializes the entire chat AI: loads synonyms, random strings, match templates, and (conditionally) reply chats; initializes console message heap.
- **Inputs:** None (reads LibVars for filenames and flags)
- **Outputs/Return:** `BLERR_NOERROR`
- **Side effects:** Populates all four global data lists; allocates `consolemessageheap` via hunk allocator
- **Calls:** `BotLoadSynonyms`, `BotLoadRandomStrings`, `BotLoadMatchTemplates`, `BotLoadReplyChat`, `InitConsoleMessageHeap`
- **Notes:** Entry point called at botlib initialization.

### BotShutdownChatAI
- **Signature:** `void BotShutdownChatAI(void)`
- **Purpose:** Frees all chat states, cached chat data, the console message heap, and all four global data lists.
- **Side effects:** Zeros all global pointers; frees hunk and heap memory.
- **Calls:** `BotFreeChatState`, `FreeMemory`, `BotFreeMatchTemplates`, `BotFreeReplyChat`

### BotLoadChatFile
- **Signature:** `int BotLoadChatFile(int chatstate, char *chatfile, char *chatname)`
- **Purpose:** Loads initial chat data for a bot state, with caching via `ichatdata[]`.
- **Inputs:** Chat state handle, file path, chat block name
- **Outputs/Return:** `BLERR_NOERROR` or `BLERR_CANNOTLOADICHAT`
- **Side effects:** Modifies `cs->chat`; may allocate `ichatdata` entry
- **Calls:** `BotLoadInitialChat`, `GetClearedMemory`

### BotInitialChat
- **Signature:** `void BotInitialChat(int chatstate, char *type, int mcontext, char *var0..var7)`
- **Purpose:** Selects a chat message of the given type, packs up to 8 variable strings, and constructs the final output in `cs->chatmessage`.
- **Side effects:** Updates `cs->chatmessage`; stamps chosen message with recency time.
- **Calls:** `BotChooseInitialChatMessage`, `BotConstructChatMessage`

### BotReplyChat
- **Signature:** `int BotReplyChat(int chatstate, char *message, int mcontext, int vcontext, char *var0..var7)`
- **Purpose:** Evaluates all reply-chat rules against the incoming message, selects the highest-priority matching rule, and constructs the reply.
- **Outputs/Return:** `qtrue` if a reply was constructed, else `qfalse`
- **Side effects:** Updates `cs->chatmessage`; stamps chosen message with recency time; reads `bot_testrchat` libvar for debug output
- **Calls:** `StringsMatch`, `StringContains`, `StringContainsWord`, `BotConstructChatMessage`, `BotRemoveTildes`

### StringsMatch
- **Signature:** `int StringsMatch(bot_matchpiece_t *pieces, bot_match_t *match)`
- **Purpose:** Attempts to match `match->string` against a sequence of `MT_STRING`/`MT_VARIABLE` pieces, recording variable offsets and lengths.
- **Outputs/Return:** `qtrue` on full match
- **Calls:** `StringContains`
- **Notes:** Adjacent variables are forbidden at load time; empty-string alternatives allow optional matches.

### BotFindMatch
- **Signature:** `int BotFindMatch(char *str, bot_match_t *match, unsigned long int context)`
- **Purpose:** Tries all loaded match templates in the given context; populates `match->type/subtype` and variable slots.
- **Outputs/Return:** `qtrue` on first match found
- **Calls:** `StringsMatch`

### BotExpandChatMessage
- **Signature:** `int BotExpandChatMessage(char *outmessage, char *message, unsigned long mcontext, bot_match_t *match, unsigned long vcontext, int reply)`
- **Purpose:** Single-pass expansion of escape-coded message: substitutes `\x01vN\x01` variable references and `\x01rNAME\x01` random strings.
- **Outputs/Return:** `qtrue` if any random expansion occurred (triggers re-expansion loop)
- **Side effects:** Writes to `outmessage`; calls synonym replacement on result
- **Calls:** `RandomString`, `BotReplaceReplySynonyms`, `BotReplaceSynonyms`, `BotReplaceWeightedSynonyms`

### BotEnterChat
- **Signature:** `void BotEnterChat(int chatstate, int clientto, int sendto)`
- **Purpose:** Issues the constructed chat message as a game command (`say`, `say_team`, or `tell`) and clears `cs->chatmessage`.
- **Side effects:** Calls `EA_Command`; clears pending message buffer

### Notes (minor helpers)
- `IsWhiteSpace`, `UnifyWhiteSpaces`, `BotRemoveTildes` — string normalization utilities
- `StringContains`, `StringContainsWord`, `StringReplaceWords` — substring search/replace primitives used by synonym logic
- `AllocConsoleMessage` / `FreeConsoleMessage` — O(1) heap slab allocator/deallocator
- `BotDump*` functions — debug-only logging to the log file pointer

## Control Flow Notes
- **Init:** `BotSetupChatAI` is called once at botlib startup, loading all data files into global lists.
- **Per-bot init:** `BotAllocChatState` + `BotLoadChatFile` + `BotSetChatName/Gender` configure a new bot's chat state.
- **Per-frame (reactive):** The game calls `BotQueueConsoleMessage` to feed console text; `BotNextConsoleMessage` / `BotRemoveConsoleMessage` drain the queue.  The game then calls `BotFindMatch` → `BotReplyChat` or `BotInitialChat` to generate a response, and `BotEnterChat` to deliver it.
- **Shutdown:** `BotShutdownChatAI` is called at botlib shutdown to release all resources.

## External Dependencies
- `l_memory.h` — `GetMemory`, `GetClearedMemory`, `GetHunkMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_libvar.h` — `LibVarValue`, `LibVarString`, `LibVarGetValue`
- `l_script.h` / `l_precomp.h` — `source_t`, `token_t`, `LoadSourceFile`, `FreeSource`, `PC_ReadToken`, `PC_ExpectToken*`, `PC_CheckToken*`, `SourceError`, `SourceWarning`, `StripDoubleQuotes`
- `l_log.h` — `Log_FilePointer`, `Log_Write` (debug dump functions)
- `be_interface.h` — `botimport` (global import table for `Print`); `bot_developer` flag
- `be_aas.h` / `be_aas_funcs.h` — `AAS_Time()` (used for message recency timestamps)
- `be_ea.h` — `EA_Command` (delivers the final say/tell command to the game)
- `be_ai_chat.h` — public API declarations (`bot_match_t`, `MAX_MATCHVARIABLES`, `MAX_MESSAGE_SIZE`, gender constants, `BLERR_*`)
- `botlib.h` — `botimport_t` structure definition
- `q_shared.h` — `qboolean`, `MAX_CLIENTS`, `MAX_QPATH`, `Com_Memset`, `Com_Memcpy`, `Q_stricmp`, `Q_strncpyz`, `va`
