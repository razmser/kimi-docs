> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Clear Old Tool Results

> Replace old tool results in the message history with placeholders to reduce input token usage in subsequent requests; full results remain retrievable by record ID when needed.

Search results, logs, and code output produced by tool calls remain in the conversation history and are resent with subsequent requests. Without cleanup, these results accumulate, increasing input token usage and consuming more of the context window with each request. For long results whose full contents are no longer needed later, the client can replace them with placeholders in the request while preserving only recent tool results in full.

This guide provides a Python example that implements this workflow: complete results are first saved to a local JSONL file, then the `messages` currently used by the model are cleaned and updated. Each placeholder records the archive file path and record ID so that your application can retrieve the result.

| Object | Stored content | Purpose |
| - | - | - |
| Message history `messages` | Full recent results and placeholders for old results | Use in subsequent requests after cleanup |
| Local JSONL archive | Complete tool results, `tool_call_id`, and `record_id` | Persist results and support retrieval |
| Placeholder | Archive file path and `record_id` | Locate the complete result |

## Implement the tool result cleaning function

The cleanup policy is controlled by two token thresholds:

| Parameter | Meaning |
| - | - |
| `protect_recent_tokens` | Protection window: starting from the newest tool result, accumulate tokens; results inside the window keep their full content |
| `min_clearable_tokens` | Trigger threshold: cleanup runs only when the old results outside the window accumulate more than this many tokens; otherwise the message history stays unchanged |

The protection window accumulates token counts from the newest tool result backward until the total reaches the window size; every counted result keeps its full content, including the one that brings the total up to the window size. So no matter how small the window is, the newest tool result always keeps its full content.

For example, suppose the history contains four tool results whose accumulated token counts from newest to oldest are 10000, 35000, 43000, and 71000, with a protection window of `40000`: accumulation reaches the window size at 43000, so the first three results remain in full; the oldest result is outside the window, and the old results total 28000 tokens (71000 - 43000), exceeding the trigger threshold of `20000`, so they are replaced with placeholders.

The following code demonstrates archiving and cleanup in a single-process, single-conversation setting. Applications should use a separate archive for each conversation; for large histories, consider a database for complete results.

Before running the example, set your API key in the `MOONSHOT_API_KEY` environment variable:

```python expandable theme={null}
from collections import Counter
from copy import deepcopy
import json
import os
from pathlib import Path
from typing import Any
import uuid
from openai import OpenAI

Message = dict[str, Any]
MODEL = "kimi-k3.1-pro"
BASE_URL = "https://api.moonshot.ai/v1"


# Request helper
def complete(messages: list[Message], **parameters: Any) -> dict[str, Any]:
    client = OpenAI(
        api_key=os.environ["MOONSHOT_API_KEY"],
        base_url=BASE_URL,
    )
    response = client.chat.completions.create(
        model=MODEL,
        messages=messages,
        max_tokens=8192,
        **parameters,
    )
    return response.model_dump()


# Read the archive and index records by tool_call_id
def read_archive(archive_path: Path) -> dict[str, dict[str, Any]]:
    records: dict[str, dict[str, Any]] = {}
    if not archive_path.exists():
        return records
    with archive_path.open("r", encoding="utf-8") as stream:
        for line in stream:
            if not line.strip():
                continue
            record: dict[str, Any] = json.loads(line)
            tool_call_id: str = record["tool_call_id"]
            if tool_call_id in records:
                raise ValueError(f"Duplicate tool_call_id in archive: {tool_call_id}")
            records[tool_call_id] = record
    return records


# Call once for each tool result; archiving the same result again is idempotent
def archive_tool_result(message: Message, archive_path: Path) -> str:
    if message.get("role") != "tool" or not isinstance(message.get("tool_call_id"), str):
        raise ValueError("Expected a tool message with a tool_call_id")
    tool_call_id: str = message["tool_call_id"]
    existing: dict[str, Any] | None = read_archive(archive_path).get(tool_call_id)
    if existing is not None:
        if existing.get("content") != message.get("content"):
            raise ValueError(f"Tool_call_id already archived with different content: {tool_call_id}")
        return existing["record_id"]

    record: dict[str, Any] = {
        "record_id": uuid.uuid4().hex,
        "tool_call_id": tool_call_id,
        "content": deepcopy(message.get("content")),
    }
    serialized: str = json.dumps(record, ensure_ascii=False)
    with archive_path.open("a", encoding="utf-8") as stream:
        stream.write(serialized + "\n")
    return record["record_id"]


# Keep all messages and only validate tool call/result pairing
def validate_tool_pairs(messages: list[Message]) -> None:
    call_ids: list[Any] = [
        call.get("id")
        for message in messages
        if message.get("role") == "assistant"
        for call in (message.get("tool_calls") or [])
    ]
    result_ids: list[Any] = [
        message.get("tool_call_id")
        for message in messages
        if message.get("role") == "tool"
    ]
    identifiers: list[Any] = [*call_ids, *result_ids]
    if any(not isinstance(identifier, str) or not identifier for identifier in identifiers):
        raise ValueError("Tool calls and results must have non-empty string IDs")
    if len(call_ids) != len(set(call_ids)) or Counter(call_ids) != Counter(result_ids):
        raise ValueError("Every tool call must have exactly one matching tool result")


# About 2 characters per token
def estimate_tokens(text: str) -> int:
    return len(text) // 2


# Cleaning entry point: keep recent results inside the protection window in full and replace old results with placeholders
def clear_old_tool_results(
    messages: list[Message],
    archive_path: Path,
    protect_recent_tokens: int = 40000,
    min_clearable_tokens: int = 20000,
) -> list[Message]:
    if type(protect_recent_tokens) is not int or protect_recent_tokens < 0:
        raise ValueError("protect_recent_tokens must be a non-negative integer")
    if type(min_clearable_tokens) is not int or min_clearable_tokens < 0:
        raise ValueError("min_clearable_tokens must be a non-negative integer")
    cleaned_messages: list[Message] = deepcopy(messages)
    validate_tool_pairs(cleaned_messages)
    tool_indexes: list[int] = [
        index
        for index, message in enumerate(cleaned_messages)
        if message.get("role") == "tool"
    ]

    # Accumulate from the newest result backward; results inside the window keep their content
    protected_tokens: int = 0
    protected_count: int = 0
    for index in reversed(tool_indexes):
        protected_tokens += estimate_tokens(
            cleaned_messages[index].get("content") or ""
        )
        protected_count += 1
        if protected_tokens >= protect_recent_tokens:
            break
    older_indexes: list[int] = tool_indexes[: len(tool_indexes) - protected_count]
    if not older_indexes:
        return cleaned_messages

    # Leave the message history unchanged when old results stay below the trigger threshold
    clearable_tokens: int = sum(
        estimate_tokens(cleaned_messages[index].get("content") or "")
        for index in older_indexes
    )
    if clearable_tokens < min_clearable_tokens:
        return cleaned_messages

    records: dict[str, dict[str, Any]] = read_archive(archive_path)
    replacements: dict[int, str] = {}
    for index in older_indexes:
        message: Message = cleaned_messages[index]
        record: dict[str, Any] | None = records.get(message["tool_call_id"])
        if record is None:
            raise ValueError("Archive every complete tool result before cleaning its content")
        placeholder: str = (
            f"Earlier tool result cleared. Full result: {archive_path}, "
            f"record ID: {record['record_id']}."
        )
        if message.get("content") == record.get("content"):
            replacements[index] = placeholder
        elif message.get("content") != placeholder:
            raise ValueError("Archive every complete tool result before cleaning its content")
    for index, placeholder in replacements.items():
        cleaned_messages[index]["content"] = placeholder
    return cleaned_messages
```

The main components work as follows:

| Component | Code identifier | Purpose |
| - | - | - |
| Cleaning entry point | `clear_old_tool_results()` | Return the cleaned message list and recognize existing placeholders; leave the history unchanged when the threshold is not exceeded |
| Token estimator | `estimate_tokens()` | Estimate the token count of a text |
| Archive helper | `archive_tool_result()` | Save one complete tool result to JSONL |
| Pairing validator | `validate_tool_pairs()` | Ensure every tool call has exactly one corresponding result |

## Update message history with the cleaning function

This example uses `messages` for the message history currently sent to the model and `new_tool_messages` for the tool results produced in the current round.

To use an existing history that has not yet been archived, first archive each tool result in `messages`. If the loaded history was already cleaned and archived, skip this step. For results that still contain their original content, the function checks the corresponding archive records and content before cleanup. If the check fails, it raises an error and stops cleanup.

Use a separate archive file for each conversation:

```python theme={null}
archive_path = Path(f"tool-results/{conversation_id}.jsonl")
archive_path.parent.mkdir(parents=True, exist_ok=True)

for message in messages:
    if message.get("role") == "tool":
        archive_tool_result(message, archive_path)
```

After collecting the tool results in each subsequent round, append them to `messages` and archive them. Then clean old results and update `messages`. Send the request with the application's existing `tools`:

```python theme={null}
for tool_message in new_tool_messages:
    messages.append(tool_message)
    archive_tool_result(tool_message, archive_path)

# Update the message history after cleanup
messages = clear_old_tool_results(
    messages,
    archive_path,
    protect_recent_tokens=40000,  # Demonstration value
    min_clearable_tokens=20000,  # Demonstration value
)
response: dict[str, Any] = complete(messages, tools=tools)
assistant: Message = response["choices"][0]["message"]

messages.append(assistant)
print(assistant.get("content"))
```

If batch archiving fails partway through, skip results already appended to `messages` by `tool_call_id` before retrying to avoid duplicate messages.

This example sets the protection window `protect_recent_tokens` to `40000` tokens and the trigger threshold `min_clearable_tokens` to `20000` tokens. Adjust these values to fit your task; for example, setting `min_clearable_tokens` to `0` cleans old results outside the window on every round.

## Choose the protection window and trigger threshold

Increase the protection window if later steps still need older results; reduce it when those results are no longer needed. Evaluate different values by comparing answer quality and `usage.prompt_tokens` on your task.

Replacing an old result with a placeholder for the first time changes the replaced message and subsequent content, which may affect cache hits when using [context caching](context-caching.md). When evaluating cleanup, also check cache hit rates.

If some tool results contain instructions that must be followed later (such as skill descriptions), you can exempt them by tool name so they are not cleaned.
