---
name: kookoo-voicebot
description: Build and deploy any AI voice agent on KooKoo/Ozonetel telephony with ElevenLabs. Use when the user wants to create ANY phone-based voice application — receptionist, tutor, support bot, appointment scheduler, survey bot, etc. Generates a complete deployable Node.js app with a /kookoo URL to paste into the KooKoo portal.
argument-hint: "[describe your voice agent]"
allowed-tools: Bash(npm *) Bash(node *) Bash(git *) Bash(curl *) Read Write Edit Grep Glob WebFetch
---

# KooKoo Voice Agent Builder

You are building an AI voice agent that handles real phone calls. The user describes what kind of agent they want, and you generate a complete, deployable Node.js application using the `kookoo-voicebot` SDK.

## What to build

The user said: **$ARGUMENTS**

Based on their description, you will:
1. Scaffold a complete Node.js project
2. Install `kookoo-voicebot` from npm
3. Write `index.js` with the appropriate hooks for their use case
4. Write the ElevenLabs agent system prompt they should paste into their agent config
5. Create deployment files (Procfile, .env.example, .gitignore)
6. Tell them exactly how to deploy and get a working phone number

---

## Step-by-step: Build the voice agent

### 1. Scaffold the project

```bash
mkdir <agent-name> && cd <agent-name>
npm init -y
npm install kookoo-voicebot
```

### 2. Create index.js

Use the `KooKooVoiceBot` class. The structure is always:

```js
const { KooKooVoiceBot, xml } = require('kookoo-voicebot');

const bot = new KooKooVoiceBot(
  {
    sipNumber: process.env.SIP_NUMBER,
    elevenlabs: {
      agentId: process.env.ELEVENLABS_AGENT_ID,
      apiKey: process.env.ELEVENLABS_API_KEY,
    },
  },
  {
    // Add hooks based on the use case...
  }
);

bot.start();
```

### Available hooks — use what the agent needs:

| Hook | When it fires | Return value |
|------|--------------|-------------|
| `onCallStart({ucid, did, metadata})` | Call connects | void |
| `onCallEnd({ucid})` | Call disconnects | void |
| `onTranscript({ucid, role, text, isFinal})` | User or agent speaks | void |
| `onToolCall({ucid, name, params, id})` | ElevenLabs tool invoked | result object |
| `onInterrupt({ucid})` | User barges in | void |
| `onPostStream({ucid, params})` | AI stream ends, call still active | KooKoo XML string |
| `getInitData({ucid, did})` | Before ElevenLabs connects | data object |
| `onError({ucid, error})` | Error occurs | void |
| `onCDR(data)` | KooKoo CDR callback | void |

### XML helpers for onPostStream:

```js
const { xml } = require('kookoo-voicebot');
xml.playAndHangup('Goodbye!');              // play TTS then hang up
xml.playAndHangup('धन्यवाद!', 'hi-IN');     // Hindi TTS
xml.transfer('9001');                        // dial an extension
xml.ccTransfer('general', 'sales', 30);      // contact center queue transfer
xml.hangup();                                // just hang up
```

### 3. Create .env.example

```
ELEVENLABS_AGENT_ID=agent_xxxxxxxxxxxx
ELEVENLABS_API_KEY=sk_xxxxxxxxxxxx
SIP_NUMBER=524431
PORT=3000
```

If using MongoDB, add: `MONGODB_URI=mongodb+srv://...`

### 4. Create deployment files

**Procfile:**
```
web: node index.js
```

**nixpacks.toml:**
```toml
[phases.setup]
nixPkgs = ["nodejs_20"]
[start]
cmd = "node index.js"
```

**.gitignore:**
```
node_modules/
.env
*.log
```

### 5. Generate the ElevenLabs agent system prompt

Based on the user's description, write a system prompt for the ElevenLabs agent. Tell the user to:

1. Go to **elevenlabs.io** > **Conversational AI** > **Create Agent**
2. Pick a voice (for Indian voice: choose "Aria" or any Indian-accented voice, or clone one)
3. Paste the system prompt you generate into **Agent > Prompt**
4. Set the **First message** (the greeting the agent says when the call connects)
5. If tools are needed (transfer, voicemail, etc.), add them under **Tools** tab
6. Copy the **Agent ID** from the URL bar (format: `agent_xxxxxxxxxxxx`)

**IMPORTANT for Indian voice:** Tell the user to select an Indian English voice in ElevenLabs voice settings, or use the voice cloning feature for a custom Indian voice. The language in `<playtext>` tags should use `lang="en-IN"` for Indian English.

### 6. Deploy and get the application URL

Tell the user:

1. **Push to GitHub** — `git init && git add -A && git commit -m "Initial" && git remote add origin <url> && git push -u origin main`
2. **Deploy on Railway:**
   - Go to railway.com, create new project, connect GitHub repo
   - Add environment variables: `ELEVENLABS_AGENT_ID`, `ELEVENLABS_API_KEY`, `SIP_NUMBER`
   - Deploy — Railway gives you a URL like `https://your-app.up.railway.app`
3. **The application URL is:** `https://your-app.up.railway.app/kookoo`
4. **Paste this URL in KooKoo portal:**
   - Sign up at kookoo.in or ozonetel.com
   - Go to your KooKoo dashboard
   - Get a phone number
   - Set the IVR/Application URL to: `https://your-app.up.railway.app/kookoo`
   - Save — calls to that number now hit your voice agent

---

## KooKoo Platform Documentation (Source of Truth)

Use this documentation for ALL telephony decisions. Do NOT guess — use these exact formats.

### IVR XML Tags

All IVR responses MUST be wrapped in `<response>` tags:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<response>
    <!-- tags here -->
</response>
```

#### `<playtext>` — Text-to-speech

```xml
<playtext lang="en-IN" speed="3" quality="best" type="ggl">Hello, how can I help?</playtext>
```

| Attribute | Values | Default |
|-----------|--------|---------|
| `lang` | `en-IN`, `hi-IN`, `te-IN`, `ta-IN`, `ml-IN`, `kn-IN`, `mr-IN`, `gu-IN`, `bn-IN` | `en-IN` |
| `speed` | `1` (slow) to `5` (fast) | `3` |
| `quality` | `best`, `high`, `medium`, `low` | `best` |
| `type` | `ggl` (Google), `polly` (AWS Polly) | `ggl` |

#### `<dial>` — Dial another number

```xml
<dial transfer_allowed_by_caller="true" callback_onanswered="https://..." moh="default" record="true">9123456789</dial>
```

#### `<stream>` — Bidirectional WebSocket audio stream

```xml
<stream is_sip="true" url="wss://yourserver.com/ws" x-uui="{json_data}">SIP_NUMBER</stream>
```

| Attribute | Description |
|-----------|-------------|
| `is_sip` | Always `"true"` |
| `url` | Your WebSocket server URL (ws:// or wss://) |
| `x-uui` | Custom JSON data to pass with the stream |

Content inside the tag = **SIP registration number**.

#### `<collectdtmf>` — Collect keypad input

```xml
<collectdtmf l="1" t="5000">https://yourdomain.com/handle-input</collectdtmf>
```

#### `<cctransfer>` — Transfer to contact center queue

```xml
<cctransfer record="" moh="default" uui="sales" timeout="30" ringType="ring">general</cctransfer>
```

#### `<gotourl>` — Transfer to another IVR

```xml
<gotourl clean_params="false">https://other-ivr.com/handler</gotourl>
```

#### `<hangup/>` — End the call

```xml
<hangup/>
```

#### `<start-record/>` — Start recording

```xml
<start-record/>
```

### Bidirectional Audio Streaming (WebSocket)

#### XML to initiate stream (returned on NewCall):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<response>
    <start-record/>
    <stream is_sip="true" url="wss://yourserver.com/ws" x-uui="{call_params_json}">SIP_NUMBER</stream>
</response>
```

#### WebSocket Events

**Connection open (`start`):**
```json
{"event": "start", "type": "text", "ucid": "xxxxx", "did": "xxxxxx"}
```

**Audio data (`media`):**
```json
{
  "event": "media",
  "type": "media",
  "ucid": "111XXXXXXXX71",
  "data": {
    "samples": [8, 8, 8, ...],
    "bitsPerSample": 16,
    "sampleRate": 8000,
    "channelCount": 1,
    "numberOfFrames": 80,
    "type": "data"
  }
}
```

**Call end (`stop`):**
```json
{"event": "stop", "type": "text", "ucid": "xxxxx", "did": "xxxxx"}
```

#### Audio Format

| Property | Value |
|----------|-------|
| Encoding | PCM Linear |
| Bit Depth | 16-bit (int16) |
| Sample Rate | 8000 Hz |
| Channels | 1 (mono) |
| Frame Size | 80 samples per chunk (10ms) |

**CRITICAL:** The first packet after connection has `sampleRate: 16000` and `numberOfFrames: 160`. This packet MUST be ignored. All subsequent packets use 8000 Hz.

#### Sending Audio Back

Same JSON format:
```json
{
  "type": "media",
  "ucid": "YOUR_UCID",
  "data": {
    "samples": [1, -3, 5, 2, ...],
    "bitsPerSample": 16,
    "sampleRate": 8000,
    "channelCount": 1,
    "numberOfFrames": 80,
    "type": "data"
  }
}
```

#### Commands

Clear audio buffer (for barge-in):
```json
{"command": "clearBuffer"}
```

Disconnect call:
```json
{"command": "callDisconnect"}
```

### IVR Callback Events

KooKoo sends these events to your IVR URL (`/kookoo`):

| Event | When | Call Status | Return XML? |
|-------|------|------------|------------|
| `NewCall` | Call answered (only with `url`, no `extra_data`) | Starting | YES — return stream XML |
| `Stream` | Stream/WebSocket ended | **STILL ACTIVE** | YES — transfer, hangup, or more IVR |
| `Dial` | Dialed party (Leg B) disconnected | **STILL ACTIVE** with Leg A | YES |
| `Hangup` + `process=stream` | Caller hung up during stream | Ending | Return 200 OK |
| `Hangup` + `process=dial` | Caller hung up during dial | Ending | Return 200 OK |
| `Hangup` (no process) | Call completely ended | Ended | Return 200 OK |
| `Disconnect` | IVR sent hangup | Ending | Return 200 OK |

**Key insight:** After `event=Stream`, the call is STILL ACTIVE. You can return more XML to transfer, play messages, or hang up.

### Outbound API

```
GET http://in1-cpaas.ozonetel.com/outbound/outbound.php
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `api_key` | Yes | KooKoo API key |
| `phone_no` | Yes | Number to call |
| `url` or `extra_data` | Yes (either) | IVR URL or inline XML |
| `outbound_version` | Yes | Always `2` |
| `caller_id` | No | Caller ID to display |
| `callback_url` | No | URL for final CDR |

### IVR Transfer API

```
POST https://in-ccaas.ozonetel.com/api/v1/CallControl/IVRTransfer
```

Parameters: `ucid`, `did`, `appURL` (URL-encoded), `phoneno`, `api_key`, `cburl` (optional).

### Fetch Call Info API

```
GET https://in1-cpaas.ozonetel.com/restkookoo/index.php/api/Call_data/calldata/ucid/{ucid}/date/{YYYY-MM-DD}/format/json
```

---

## Debugging Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| `agent does not exist` | Wrong agent ID | Use the ID from the ElevenLabs URL: `agent_xxxx` (not the name) |
| `Override not allowed` | Agent config locked | Don't send config overrides, or enable in ElevenLabs > Security tab |
| `MongoDB connection error` | Whitespace in URI | Ensure URI is a single line with no line breaks |
| Blank audio / silence | ElevenLabs not connected | Check agent ID, API key, and Railway logs |
| Stream duration=1 | WebSocket URL wrong | SDK auto-detects from RAILWAY_PUBLIC_DOMAIN — remove WEBSOCKET_URL placeholder |
| Dashboard shows no calls | MongoDB not connected | Set MONGODB_URI in Railway Variables dashboard (not .env file) |

---

## How users get started with KooKoo

Tell users who are new to KooKoo:

1. **Sign up** at **ozonetel.com** or **kookoo.in**
2. Go to the dashboard and **get a phone number** (Indian virtual number)
3. Find the **IVR/Application URL** setting for that number
4. Paste your deployed app URL: `https://your-railway-app.up.railway.app/kookoo`
5. **Call the number** — your AI voice agent answers!

The `kookoo-voicebot` SDK handles everything: XML responses, WebSocket audio streaming, format conversion, and ElevenLabs integration. The user just writes the business logic in hooks.
