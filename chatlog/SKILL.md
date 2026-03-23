---
name: chatlog
description: Save the current conversation to a JSON file. Use when the user says "save chatlog", "保存聊天记录", "导出对话", "save conversation", or similar.
---

# Save Chatlog

Extract user questions and assistant answers from the current session into a clean JSON file.

Keep the scope narrow:
- Extract the conversation from the session file and write it to JSON.
- Do not spend time on extra cleanup or special-case filtering unless the user explicitly asks for it.

## Step 1: Detect which tool is running

Check environment variables:
- `CLAUDECODE=1` → **Claude Code**
- Otherwise → **Codex**

## Step 2: Find the current session file and extract

### Claude Code

Session files location:
```
~/.claude/projects/<project-slug>/<session-uuid>.jsonl
```
`<project-slug>` = working directory absolute path with `/` replaced by `-` (e.g., `/home/ubuntu/wb` → `-home-ubuntu-wb`).

Find current session: `ls -t ~/.claude/projects/<project-slug>/*.jsonl | head -1`

JSONL format:
- Each line is a JSON object with `"type"` field being `"user"`, `"assistant"`, or `"file-history-snapshot"`
- User messages: `record.message.content` is a **plain string**
- Assistant messages: `record.message.content` is a **list of items**, extract only items where `item.type == "text"` and take `item.text`
- Consecutive same-role messages should be merged (streaming chunks)
- Skip records where type is not `"user"` or `"assistant"`

Inline extraction script — you can run it directly:

```python
import json, sys
from pathlib import Path

def extract_claude(input_path, output_path):
    messages = []
    with open(input_path, "r", encoding="utf-8") as f:
        for raw in f:
            line = raw.strip()
            if not line:
                continue
            rec = json.loads(line)
            if rec.get("type") not in ("user", "assistant"):
                continue
            msg = rec.get("message", {})
            role = msg.get("role")
            if role not in ("user", "assistant"):
                continue
            content = msg.get("content")
            if isinstance(content, str):
                text = content.strip()
            elif isinstance(content, list):
                text = "\n".join(
                    item["text"] for item in content
                    if item.get("type") == "text" and isinstance(item.get("text"), str)
                ).strip()
            else:
                continue
            if not text:
                continue
            if messages and messages[-1]["role"] == role:
                messages[-1]["content"] += "\n" + text
            else:
                messages.append({"role": role, "content": text})
    Path(output_path).write_text(
        json.dumps({"messages": messages}, ensure_ascii=False, indent=2) + "\n", encoding="utf-8"
    )
    print(f"Extracted {len(messages)} messages -> {output_path}")

if __name__ == "__main__":
    extract_claude(sys.argv[1], sys.argv[2])
```

### Codex

Session files location:
```
~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<uuid>.jsonl
```

Find current session: `find ~/.codex/sessions -name '*.jsonl' -printf '%T@ %p\n' | sort -rn | head -1 | cut -d' ' -f2`

JSONL format:
- Filter lines where `record.type == "response_item"` and `record.payload.type == "message"`
- Role is at `record.payload.role`, keep only `"user"` and `"assistant"`
- Content is at `record.payload.content` (a list), extract items where `item.type` is one of `"input_text"`, `"output_text"`, `"text"` and take `item.text`

Inline extraction script — you can run it directly:

```python
import json, sys
from pathlib import Path

TEXT_TYPES = {"input_text", "output_text", "text"}

def extract_codex(input_path, output_path):
    messages = []
    with open(input_path, "r", encoding="utf-8") as f:
        for raw in f:
            line = raw.strip()
            if not line:
                continue
            rec = json.loads(line)
            if rec.get("type") != "response_item":
                continue
            payload = rec.get("payload", {})
            if payload.get("type") != "message":
                continue
            role = payload.get("role")
            if role not in ("user", "assistant"):
                continue
            content = payload.get("content", [])
            if not isinstance(content, list):
                continue
            text = "\n".join(
                item["text"] for item in content
                if item.get("type") in TEXT_TYPES and isinstance(item.get("text"), str)
            )
            if text:
                messages.append({"role": role, "content": text})
    Path(output_path).write_text(
        json.dumps({"messages": messages}, ensure_ascii=False, indent=2) + "\n", encoding="utf-8"
    )
    print(f"Extracted {len(messages)} messages -> {output_path}")

if __name__ == "__main__":
    extract_codex(sys.argv[1], sys.argv[2])
```

## Step 3: Execute


1. Find the current session jsonl file using the method above
2. You can run the extraction code directly.If it doesn't work, you should write the appropriate extraction script to a temp file (e.g., `/tmp/_chatlog_extract.py`) and then run it.
3. If you want to verify the output file, do it **after** the extraction command finishes. Do not read `chatlog.json` in parallel with the write, or you may hit a race and see stale/missing output.
4. Report the number of extracted messages and the output path to the user
