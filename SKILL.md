---
name: shelley-agent
description: Spawn Shelley sessions programmatically - create conversations, send prompts, and get results from the Shelley coding agent.
homepage: https://exe.dev
metadata: {"clawdbot":{"emoji":"🐚"}}
---

# shelley-agent

Invoke Shelley (the coding agent on this VM) programmatically. Create new chat sessions, send prompts, wait for completion, and retrieve results.

## Overview

Shelley is the coding agent running on this exe.dev VM at port 9999. This skill allows Clawdbot to:
- Create new Shelley conversations
- Send prompts/instructions to Shelley
- Poll for completion
- Retrieve the final response
- All sessions are stored in the VM's Shelley database

## API Endpoints

Shelley exposes a REST API at `http://localhost:9999/api/`

### Authentication

All requests require these headers:
```
X-Exedev-Userid: <user-id>
X-Shelley-Request: 1
```

## Commands

### Create a New Conversation with a Prompt

```bash
# Start a new conversation and send initial prompt
curl -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  -H "Content-Type: application/json" \
  "http://localhost:9999/api/conversations/new" \
  -d '{
    "message": "Create a hello world Python script",
    "cwd": "/home/exedev"
  }'

# Response:
# {"conversation_id": "cXXXXXXX", "status": "accepted"}
```

### Check Conversation Status

```bash
# Poll to check if agent is still working
curl -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}" | jq '.agent_working'

# Returns: true (still working) or false (done)
```

### Get Conversation Details

```bash
# Get full conversation with all messages
curl -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}" | jq '{
    status: .agent_working,
    slug: .conversation.slug,
    messages: [.messages[] | {type, id: .message_id}]
  }'
```

### Send Follow-up Message

```bash
# Continue an existing conversation
curl -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  -H "Content-Type: application/json" \
  "http://localhost:9999/api/conversation/{conversation_id}/chat" \
  -d '{"message": "Now add error handling"}'
```

### Stream Conversation (Real-time)

```bash
# Get real-time updates via Server-Sent Events
curl -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}/stream"
```

### Cancel Running Conversation

```bash
curl -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}/cancel"
```

### List All Conversations

```bash
curl -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversations" | jq '.[:5]'
```

### Get Conversation by Slug

```bash
curl -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation-by-slug/{slug}"
```

### Archive/Delete Conversations

```bash
# Archive
curl -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}/archive"

# Unarchive
curl -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}/unarchive"

# Delete permanently
curl -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}/delete"
```

## Extracting Agent Response

The agent's response is in the `messages` array with `type: "agent"`. Parse the `llm_data` JSON:

```bash
# Get the final text response
curl -s -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/{conversation_id}" | \
  jq -r '.messages[] | select(.type=="agent") | .llm_data | fromjson | .Content[] | select(.Type==2) | .Text'
```

Message types in `llm_data.Content`:
- `Type: 2` - Text content
- `Type: 5` - Tool use (function calls)

## Complete Workflow Example

```bash
#!/bin/bash
# shelley-task.sh - Run a task via Shelley and get result

PROMPT="$1"
CWD="${2:-/home/exedev}"

# Create conversation
RESPONSE=$(curl -s -X POST \
  -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  -H "Content-Type: application/json" \
  "http://localhost:9999/api/conversations/new" \
  -d "{\"message\": \"$PROMPT\", \"cwd\": \"$CWD\"}")

CONV_ID=$(echo "$RESPONSE" | jq -r '.conversation_id')
echo "Started conversation: $CONV_ID" >&2

# Poll until complete
while true; do
  STATUS=$(curl -s -H "X-Exedev-Userid: clawdbot" \
    -H "X-Shelley-Request: 1" \
    "http://localhost:9999/api/conversation/$CONV_ID" | jq '.agent_working')
  
  if [ "$STATUS" = "false" ]; then
    break
  fi
  
  echo "Working..." >&2
  sleep 2
done

# Get final response
curl -s -H "X-Exedev-Userid: clawdbot" \
  -H "X-Shelley-Request: 1" \
  "http://localhost:9999/api/conversation/$CONV_ID" | \
  jq -r '.messages | map(select(.type=="agent")) | last | .llm_data | fromjson | .Content[] | select(.Type==2) | .Text'
```

## Viewing Sessions

All Shelley sessions are stored in:
- **Database**: `/home/exedev/.config/shelley/shelley.db`
- **Web UI**: https://clawdbot-sthala.exe.xyz:9999/

Query sessions directly:
```bash
sqlite3 /home/exedev/.config/shelley/shelley.db \
  "SELECT conversation_id, slug, datetime(created_at, 'localtime') FROM conversations ORDER BY created_at DESC LIMIT 10;"
```

## Notes

- Shelley has access to all tools (bash, file editing, browser, etc.)
- Long-running tasks may take minutes - use polling or streaming
- The `cwd` parameter sets the working directory for the session
- Each conversation is independent with its own context
- Conversations auto-generate a slug based on content
