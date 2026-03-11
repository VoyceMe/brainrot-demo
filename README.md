# Brainrot Demo — Roblox + Character Server Integration

## Overview
A gacha-style Roblox demo where players pull AI-powered NPCs, chat with them via Nick's Character Server API, buy them stateful items, and build trust through conversation.

**API:** `https://demo-framework-production.up.railway.app`  
**Characters:** patrick, rachel, zyxthara, nick

## Roblox Script Locations

| Script | Location | Type | Purpose |
|--------|----------|------|---------|
| `BrainrotServer` | ServerScriptService | Script | All server logic: pulls, chat, items, trust |
| `BrainrotHUD` | StarterGui | LocalScript | Top bar (coins/income), pull button, reveal animation |
| `BrainrotDialogue` | StarterGui | LocalScript | Chat UI, shop bar, message bubbles |
| `BrainrotProximity` | StarterPlayer > StarterPlayerScripts | LocalScript | ProximityPrompt hookup, signal to dialogue |
| `BrainrotConfig` | ReplicatedStorage | ModuleScript | All configuration (API URL, characters, items, colors) |
| `BrainrotRemotes` | ReplicatedStorage | ModuleScript | RemoteFunction/RemoteEvent creation |

## How It Works

### Chat Flow
1. Player presses E → `BrainrotProximity` fires `InteractNPC:InvokeServer(sceneName, "start")`
2. Server fetches greeting from `GET /character/:name` and returns it
3. Player types message → `SendMessage:FireServer(sceneName, msg)`
4. Server calls `POST /chat` with `{message, character, uid, history}`
5. Server parses SSE response, sends text back via `ReceiveMessage:FireClient`
6. Trust bumps based on simple sentiment (nice = +5 to +12, mean = -5 to -15)

### Item Buy Flow
1. Player clicks shop button → `BuyItem:InvokeServer(sceneName, itemIndex)`
2. Server deducts coins, bumps trust by item's `trustBoost`
3. Server calls `POST /items/:characterName/create` with `{templateId}` to create real API item
4. On next dialogue open, if player bought items this session, server fires async `POST /chat` with hint asking character to reference items and upsell

## ⚠️ KNOWN ISSUE — Item Hallucination

**Nick's diagnosis: "It's hallucinating them. It's not even looking at the objects."**

**What's happening:** The character talks ABOUT items ("I love my deck of cards!") but isn't actually USING the item system. When Patrick says "I drew a card," he's making it up — no actual `draw` action was called on the card deck.

**Root cause:** The Roblox client sends a plain `POST /chat` with just the message text. It does NOT:
1. Process `tool_call` events from the SSE stream (the API returns these when the AI wants to use an item)
2. Execute `tool_result` responses back to the API
3. Connect to `GET /events/:characterName` for the persistent SSE event bus
4. Handle `item_update` events that fire when an item's state changes

**What the API actually supports (from the docs):**

The `/chat` endpoint returns SSE events including:
- `text` — streamed AI text (we handle this ✅)
- `tool_call` — AI wants to use an item (e.g. draw a card) ❌ NOT HANDLED
- `tool_result` — result of tool execution ❌ NOT HANDLED
- `done` — stream complete (we handle this ✅)

The items have real actions:
- Playing Cards: `draw` (draw a card), `shuffle` (shuffle deck)
- Magic 8-Ball: `shake` (get a fortune)
- Notebook: `write` (write an entry), `read` (read entries)
- Coin: `flip` (heads or tails)

**What needs to happen:**
1. Parse `tool_call` events from the chat SSE stream
2. When AI invokes a tool, the Roblox server needs to... (this is where we need Nick's guidance — does the server auto-execute tools, or does the client need to call an endpoint?)
3. Feed `tool_result` back into the conversation
4. Optionally connect to `/events/:characterName` for real-time item state updates

**Current workaround:** We inject text hints like "[You own: Deck of Cards]" into chat prompts so the AI knows about items, but it's faking the interactions rather than using the actual tool system.

## Dependencies
- Roblox Studio (any recent version)
- HTTP Service enabled (Game Settings → Security → Allow HTTP Requests)
- Nick's API running at the configured ENDPOINT_URL

## Setup
1. Open the `.rbxl` file in Roblox Studio
2. Verify `BrainrotConfig.ENDPOINT_URL` points to the correct API
3. Enable HTTP requests in Game Settings
4. Press Play

## Files to Export from .rbxl
If setting up fresh, the 6 scripts listed above need to be placed in their correct locations. The rest of the place (room geometry, BrainrotMachine model, etc.) is part of the base template.
