# Brainrot Demo — Roblox + Character Server Integration

## Overview
A gacha-style Roblox demo where players pull AI-powered NPCs, chat with them via the Character Server API, buy them stateful items, and build trust through conversation. NPCs stream responses in real time, use tools to inspect game state and interact with items, and maintain persistent internal state (mood, health, memories) that renders in a schema-driven UI.

**API:** `https://demo-framework-production.up.railway.app`
**Local dev:** `http://localhost:3101`
**Characters:** patrick, rachel, zyxthara, nick

## Roblox Script Locations

| Script | Location | Type | Purpose |
|--------|----------|------|---------|
| `BrainrotServer` | ServerScriptService | Script | HTTP communication, streaming, tool execution, state polling |
| `BrainrotHUD` | StarterGui | LocalScript | Top bar (coins/income), pull button, reveal animation |
| `BrainrotDialogue` | StarterGui | LocalScript | Chat UI, streaming display, tool activity, state bar |
| `BrainrotProximity` | StarterPlayerScripts | LocalScript | ProximityPrompt hookup, signal to dialogue |
| `BrainrotConfig` | ReplicatedStorage | ModuleScript | Configuration (API URL, characters, items, colors) |
| `BrainrotRemotes` | ReplicatedStorage | ModuleScript | RemoteFunction/RemoteEvent creation |

## How It Works

### Chat Flow (Streaming + Tool Calling)

1. Player presses E near an NPC -> `BrainrotProximity` fires `InteractNPC:InvokeServer(sceneName, "start")`
2. Server returns greeting, client opens dialogue and requests initial state (`__request_state__`)
3. Player types a message -> `SendMessage:FireServer(sceneName, msg)`
4. Server sends `POST /chat` with a unique `chatId` for poll-based streaming
5. Server polls `GET /chat/poll/:chatId` every 150ms, forwarding events to the client as they arrive:
   - `stream_start` — shows typing indicator
   - `stream_chunk` — progressively updates the response bubble
   - `tool_activity` — displays what tool the AI is using (purple indicator in chat)
   - `stream_end` — clears typing state
6. If the AI calls a **server tool** (e.g., `look_around`, `use_object`), the backend executes it automatically
7. If the AI calls a **client tool** (e.g., `check_player_coins`), the backend pauses and sends a `client_tool_request`
8. The Roblox server executes the tool locally using live game state, then resumes via `POST /chat/tool-result`
9. The AI continues with the tool results — this can loop up to 3 rounds of client tool calls

### Why Polling Instead of SSE?

Roblox's `HttpService:RequestAsync` is synchronous — it blocks until the entire response body is received. This means true SSE streaming is impossible. The workaround:

- The backend buffers all SSE events keyed by `chatId`
- The Roblox server polls `/chat/poll/:chatId` every 150ms
- Each poll returns new events since the last cursor position
- This achieves near-real-time progressive text display

### Client-Side Tools

These tools execute on the Roblox server using live game state, because the backend has no access to in-game data:

| Tool | What It Does |
|------|-------------|
| `check_player_coins` | Returns the player's current coin balance |
| `check_player_trust` | Returns trust level (0-100) with the current NPC |
| `get_nearby_objects` | Lists NPCs and objects near the player |
| `get_brainrot_collection` | Lists collected characters with rarity and trust |

The AI decides when to call these — e.g., if you ask "can I afford that?", it might call `check_player_coins` before answering.

### Server-Side Tools

These execute on the Character Server automatically:

| Tool | What It Does |
|------|-------------|
| `look_around` | Character sees their location and inventory |
| `check_time` | Current date and time |
| `search_inventory` | Search possessions by keyword |
| `use_object` | Interact with a stateful item (8-ball, notebook, coin, etc.) |

### State Display

A schema-driven state bar sits between the chat header and message area. It renders the NPC's internal state:

- **Health bars** — Physical, mental, energy (colored progress bars)
- **Mood chip** — Color-coded by sentiment (green = positive, red = negative)
- **Emotion tags** — Purple chips for current emotions
- **Status badge** — Current status (healthy, sleepy, excited, etc.)
- **Possessions** — Teal chips for stateful items, dark chips for plain items

State updates automatically after each conversation (the backend runs a state analysis in the background) and is polled every 5 seconds during active conversations.

### Item Buy Flow

1. Player clicks shop button -> `BuyItem:InvokeServer(sceneName, itemIndex)`
2. Server deducts coins, bumps trust by item's `trustBoost`
3. Server calls `POST /items/:characterName/create` with `{templateId}` to create a real API item
4. The AI can now **actually use** the item via the `use_object` tool — no more hallucination

### Item System (Resolved)

Previously, NPCs would talk about items without actually using them. This is now fully fixed:

- The backend's `use_object` tool lets the AI interact with stateful items
- Items have real actions powered by JSONata reducers (e.g., Magic 8-Ball: `ask`, Notebook: `write`/`read`, Coin: `flip`)
- Item state persists across conversations
- The `use_object` tool is dynamically injected only when the character has stateful items
- Plain string items auto-materialize into stateful instances on first use

## Dependencies
- Roblox Studio (any recent version)
- HTTP Service enabled (Game Settings > Security > Allow HTTP Requests)
- Character Server running at the configured `ENDPOINT_URL`

## Setup

1. Open the `.rbxl` file in Roblox Studio
2. Set `BrainrotConfig.ENDPOINT_URL` to your server:
   - Local: `http://localhost:3101`
   - Production: `https://demo-framework-production.up.railway.app`
3. Enable HTTP requests in Game Settings > Security
4. Press Play

## Architecture

```
Player Input
    |
    v
BrainrotDialogue (LocalScript)     BrainrotServer (ServerScript)
    |  FireServer(msg)                  |
    |---------------------------------->|
    |                                   |  POST /chat { chatId }
    |                                   |-------------------> Character Server
    |                                   |                         |
    |                                   |  GET /chat/poll/:chatId | (every 150ms)
    |                                   |<------------------->    |
    |  FireClient(stream_chunk)         |                         |
    |<----------------------------------|                         |
    |  FireClient(tool_activity)        |                         |
    |<----------------------------------|                         |
    |                                   |                         |
    |                          [client_tool_request?]             |
    |                                   |                         |
    |                          executeClientTool()                |
    |                                   |                         |
    |                                   |  POST /chat/tool-result |
    |                                   |------------------------>|
    |                                   |                         |
    |  FireClient(stream_chunk)         |  (continues streaming)  |
    |<----------------------------------|<------------------------|
    |                                   |
    |                                   |  GET /state/:character (every 5s)
    |  FireClient(character_state)      |<----------------------->|
    |<----------------------------------|
    |
    v
State Bar + Chat Bubbles
```

## Files to Export from .rbxl
If setting up fresh, the 6 scripts listed above need to be placed in their correct locations. The rest of the place (room geometry, BrainrotMachine model, etc.) is part of the base template.
