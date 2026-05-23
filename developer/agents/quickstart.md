# Python Quickstart

Create an agent and chat with it using the [iblai/quickstarts](https://github.com/iblai/quickstarts) Python client.

---

## Overview

This quickstart shows how to programmatically create an AI Agent, start a chat session, and send messages over WebSocket — all from a Python script. You can also connect to an existing agent and session.

---

## Prerequisites

- Python 3.10+
- An ibl.ai platform API key
- A tenant and username on the platform

Install dependencies:

```bash
pip install requests websockets
```

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `IBL_TENANT` | Yes | Your tenant key |
| `IBL_USERNAME` | Yes | Your platform username |
| `IBL_PLATFORM_API_KEY` | Yes | Platform API key for authentication |
| `IBL_MENTOR_ID` | No | Existing agent unique ID (skips creation) |
| `IBL_SESSION_ID` | No | Existing session ID (requires `IBL_MENTOR_ID`) |
| `IBL_MANAGER_URL` | No | Manager API URL (defaults to `https://base.manager.iblai.app`) |
| `IBL_ASGI_URL` | No | WebSocket URL (defaults to `wss://asgi.data.iblai.app`) |

---

## Usage

```bash
# Create a new agent and chat
export IBL_TENANT="my-tenant"
export IBL_USERNAME="my-username"
export IBL_PLATFORM_API_KEY="my-api-key"
python quickstart.py "Explain quantum computing"

# Chat with an existing agent (creates a new session)
export IBL_MENTOR_ID="existing-agent-unique-id"
python quickstart.py "What is machine learning?"

# Continue an existing session
export IBL_MENTOR_ID="existing-agent-unique-id"
export IBL_SESSION_ID="existing-session-id"
python quickstart.py "Tell me more about that"
```

---

## Source Code

### quickstart.py

The entry point. Creates an agent (or uses an existing one), starts a session, and sends a prompt.

```python
from __future__ import annotations

import asyncio
import logging
import os
import random
import sys

import api

logging.basicConfig(level=logging.INFO, format="%(message)s")

log = logging.getLogger("quickstart")

TENANT = os.getenv("IBL_TENANT", "")
USERNAME = os.getenv("IBL_USERNAME", "")
PLATFORM_API_KEY = os.getenv("IBL_PLATFORM_API_KEY", "")
MENTOR_ID = os.getenv("IBL_MENTOR_ID", "")
SESSION_ID = os.getenv("IBL_SESSION_ID", "")


def run(prompt: str, mentor_unique_id: str = "", session_id: str = "") -> None:
    """Chat with an agent using an existing agent/session or create a new one.

    - Provide a mentor_unique_id to create a new session with an existing agent.
    - Provide a mentor_unique_id and session_id to continue an existing session.
    """
    if not mentor_unique_id:
        mentor_name = f"Agent-{random.randint(0, 100)}"
        log.info("Creating a new agent: %s", mentor_name)
        mentor_settings = {
            "new_mentor_name": mentor_name,
            "template_name": "ai-agent",
            "display_name": mentor_name,
            "description": "An agent used to explain difficult concepts in simple terms",
            "system_prompt": "Explain it like the user is a 5-year-old",
        }
        agent = api.create_mentor(
            api_token=PLATFORM_API_KEY,
            tenant=TENANT,
            username=USERNAME,
            settings=mentor_settings,
        )
        mentor_unique_id = agent["unique_id"]
        log.info("Created Agent: %s", mentor_unique_id)
    else:
        log.info("Using existing Agent: %s", mentor_unique_id)

    if not session_id:
        log.info("Creating a new session for agent: %s", mentor_unique_id)
        session_id = api.create_chat_session(
            username=USERNAME,
            tenant=TENANT,
            mentor_unique_id=mentor_unique_id,
        )
        log.info("Created Session ID: %s", session_id)
    else:
        log.info("Using existing Session ID: %s", session_id)

    asyncio.run(
        api.chat_with_websocket(
            prompt=prompt,
            session_id=session_id,
            agent=mentor_unique_id,
            tenant=TENANT,
            username=USERNAME,
            api_key=PLATFORM_API_KEY,
        )
    )


if __name__ == "__main__":
    api.validate_env("IBL_TENANT", "IBL_USERNAME", "IBL_PLATFORM_API_KEY")
    log.info("Using tenant=%s, username=%s", TENANT, USERNAME)
    log.info("Using Manager URL: %s", api.MANAGER_URL)
    log.info("Using Asgi URL: %s", api.ASGI_URL)
    if len(sys.argv) < 2:
        log.error('Usage: python quickstart.py "<prompt>"')
        sys.exit(1)
    prompt = sys.argv[1]

    if SESSION_ID and not MENTOR_ID:
        log.error("Error: If IBL_SESSION_ID is set, IBL_MENTOR_ID must also be set.")
        sys.exit(1)
    # - Provide a mentor_unique_id to create a new session with an existing agent.
    # - Provide a mentor_unique_id and session_id to continue an existing session.
    run(prompt, mentor_unique_id=MENTOR_ID, session_id=SESSION_ID)
```

### api.py

The API client module. Handles agent creation, session management, and WebSocket communication.

```python
from __future__ import annotations

import asyncio
import json
import logging
import os
import sys
import typing as t

import requests
from websockets.client import connect

log = logging.getLogger("api")

DEFAULT_MANAGER_URL = "https://base.manager.iblai.app"
DEFAULT_ASGI_URL = "wss://asgi.data.iblai.app"
MANAGER_URL = os.getenv("IBL_MANAGER_URL", DEFAULT_MANAGER_URL) or DEFAULT_MANAGER_URL
ASGI_URL = os.getenv("IBL_ASGI_URL", DEFAULT_ASGI_URL) or DEFAULT_ASGI_URL


def create_mentor(
    api_token: str, tenant: str, username: str, settings: dict[str, str]
) -> dict[str, t.Any]:
    """Create a new agent with the provided settings."""
    headers = {"Authorization": f"Api-Token {api_token}"}
    resp = requests.post(
        f"{MANAGER_URL}/api/ai-agent/orgs/{tenant}/users/{username}/agent-with-settings/",
        headers=headers,
        json=settings,
    )
    if not resp.ok:
        log.error(
            "Failed to create agent (status=%s): %s",
            resp.status_code,
            resp.text,
        )
        sys.exit(1)
    return resp.json()


def create_chat_session(username: str, tenant: str, mentor_unique_id: str) -> str:
    """Create a new chat session for the given user and agent."""
    resp = requests.post(
        f"{MANAGER_URL}/api/ai-agent/orgs/{tenant}/users/{username}/sessions/",
        json={"agent": mentor_unique_id},
    )
    if not resp.ok:
        log.error(
            "Failed to create chat session (status=%s): %s",
            resp.status_code,
            resp.json(),
        )
        sys.exit(1)
    session_id = resp.json().get("session_id")
    return session_id


async def chat_with_websocket(
    prompt: str, session_id: str, agent: str, tenant: str, username: str, api_key: str
):
    data = {
        "flow": {
            "name": agent,
            "tenant": tenant,
            "username": username,
            "pathway": agent,
        },
        "session_id": session_id,
        "token": api_key,
        "prompt": prompt,
    }

    ws = await connect(f"{ASGI_URL}/ws/langflow/")
    log.info("Connected to Agent")
    print("Response: ", end="")
    await ws.send(json.dumps(data))
    eos = False
    while not eos:
        try:
            data = await asyncio.wait_for(ws.recv(), timeout=10)
            msg: dict[str, str] = json.loads(data)
            if "error" in msg:
                log.error("Error from server: %s", msg["error"])
                await ws.close()
                sys.exit(1)

            if "data" in msg:
                print(msg["data"], end="", flush=True)

            if msg.get("eos"):
                eos = True
                print("")

        except asyncio.TimeoutError:
            log.warning("Timeout while waiting for response. Exiting...")
            await ws.close()
            sys.exit(1)

    await ws.close()


def validate_env(*args: str) -> None:
    """Validate that all required environment variables are set."""
    for key in args:
        if not os.getenv(key, None):
            log.error(f"Environment variable {key} is not set.")
            sys.exit(1)
```

---

## Repository

- **GitHub**: [iblai/quickstarts](https://github.com/iblai/quickstarts)
