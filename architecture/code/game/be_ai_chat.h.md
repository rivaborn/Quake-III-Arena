# code/game/be_ai_chat.h

## File Purpose
Declares the public interface for the bot chat AI subsystem, defining data structures and function prototypes used to manage bot console message queues, pattern-based chat matching, and chat message generation/delivery.

## Core Responsibilities
- Define constants for message size limits, gender flags, and chat target types
- Declare the console message linked-list node structure for per-bot message queues
- Declare match variable and match result structures for template-based message parsing
- Expose lifecycle functions for the chat AI subsystem (setup/shutdown, alloc/free state)
- Expose functions for queuing, retrieving, and removing console messages
- Expose functions for selecting, composing, and sending chat replies
- Expose utility functions for string matching, synonym replacement, and whitespace normalization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bot_consolemessage_t` | struct (typedef) | Linked-list node holding a single console message: handle, timestamp, type, text buffer, and prev/next pointers |
| `bot_matchvariable_t` | struct (typedef) | Describes a captured variable within a match: byte offset and length into the matched string |
| `bot_match_t` | struct (typedef) | Result of a template match: full matched string, type/subtype classification, and up to `MAX_MATCHVARIABLES` captured variables |

## Global / File-Static State
None.

## Key Functions / Methods

### BotSetupChatAI / BotShutdownChatAI
- Signature: `int BotSetupChatAI(void)` / `void BotShutdownChatAI(void)`
- Purpose: One-time initialization and teardown of the entire chat AI subsystem (likely loads synonym/match template data).
- Inputs: None
- Outputs/Return: Setup returns int (0 = failure, non-zero = success inferred).
- Side effects: Global subsystem state allocation/deallocation (defined elsewhere).
- Calls: Not inferable from this file.

### BotAllocChatState / BotFreeChatState
- Signature: `int BotAllocChatState(void)` / `void BotFreeChatState(int handle)`
- Purpose: Allocate or release a per-bot chat state slot; returns an opaque integer handle.
- Inputs: `handle` — index into the chat state pool.
- Outputs/Return: Alloc returns handle (≥1) or 0 on failure.
- Side effects: Modifies global chat state pool (defined elsewhere).

### BotQueueConsoleMessage / BotRemoveConsoleMessage / BotNextConsoleMessage / BotNumConsoleMessages
- Purpose: Manage a per-bot FIFO queue of incoming console messages. `Next` pops the oldest entry into a caller-supplied `bot_consolemessage_t`; `Remove` deletes by handle.
- Notes: Together these form the message-inbox abstraction consumed by reply logic.

### BotInitialChat
- Signature: `void BotInitialChat(int chatstate, char *type, int mcontext, char *var0…var7)`
- Purpose: Selects a canned chat message of the given named type, substituting up to 8 variable strings into placeholders.
- Inputs: `chatstate` handle, `type` string key, `mcontext` context mask, `var0`–`var7` substitution values.
- Side effects: Writes selected message into the chat state's output buffer.

### BotReplyChat
- Signature: `int BotReplyChat(int chatstate, char *message, int mcontext, int vcontext, char *var0…var7)`
- Purpose: Searches reply templates for a match against `message`, selects and prepares a response with variable substitution.
- Outputs/Return: Non-zero if a matching reply was found and selected.

### BotEnterChat
- Signature: `void BotEnterChat(int chatstate, int clientto, int sendto)`
- Purpose: Transmits the currently selected/prepared chat message to the game; `sendto` selects `CHAT_ALL`, `CHAT_TEAM`, or `CHAT_TELL`.
- Side effects: Issues a game syscall to deliver the message over the network.

### BotFindMatch / BotMatchVariable
- Signature: `int BotFindMatch(char *str, bot_match_t *match, unsigned long int context)` / `void BotMatchVariable(bot_match_t *match, int variable, char *buf, int size)`
- Purpose: Pattern-match an incoming string against loaded templates filtered by `context`; extract individual captured variables by index into a caller buffer.

### Notes
- `StringContains`, `UnifyWhiteSpaces`, `BotReplaceSynonyms` are string utility helpers used internally by matching and reply composition.
- `BotLoadChatFile`, `BotSetChatGender`, `BotSetChatName` handle per-bot chat personality initialization after state allocation.

## Control Flow Notes
Called during **bot initialization** (`BotSetupChatAI`, `BotAllocChatState`, `BotLoadChatFile`, `BotSetChatGender/Name`). Each **server frame**, the game module feeds incoming console messages via `BotQueueConsoleMessage`, then the AI logic calls `BotNextConsoleMessage` → `BotFindMatch` → `BotReplyChat` → `BotEnterChat` to produce responses. `BotInitialChat` / `BotEnterChat` are also invoked for proactive chat events (kills, pickups, etc.).

## External Dependencies
- No includes visible in this header; implementation resides in `botlib/be_ai_chat.c`.
- `MAX_MESSAGE_SIZE`, `MAX_MATCHVARIABLES`, gender/target constants are self-contained in this file.
- All function bodies are **defined elsewhere** (botlib shared library, linked via `botlib_export_t` function table).
