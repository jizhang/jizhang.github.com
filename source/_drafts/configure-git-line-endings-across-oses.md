---
title: Configure Git Line Endings Across OSes
tags: [git]
categories: Programming
---

In Linux, lines end with LF (Line Feed, `\n`), while in Windows, CRLF (Carriage Return + Line Feed, `\r\n`). When developers using different operating systems contribute to the same Git project, line endings must be handled correctly, or `diff` and `merge` may break unexpectedly. Git provides several solutions to this problem, including configuration options and file attributes.

## TL;DR

### Approach 1

Set `core.autocrlf` to `input` in **Windows**. Leave Linux/macOS unchanged.

```
git config --global core.autocrlf input
```

### Approach 2

Create `.gitattributes` under the project root, and add the following line:

```
* text=auto eol=lf
```

<!-- more -->
