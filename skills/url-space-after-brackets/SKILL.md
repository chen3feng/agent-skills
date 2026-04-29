---
name: url-space-after-brackets
description: Always add a space after URL brackets in Markdown to prevent 404 errors with special characters.
tags: [markdown, url, formatting, chinese]
---

# URL space after brackets

## When to use

You're writing or editing Markdown content that contains URLs, especially when the text after the URL contains Chinese characters, asterisks, or other special characters that could be misinterpreted as part of the URL.

## Problem

When generating URLs in Markdown without a trailing space, browsers and parsers may incorrectly include subsequent characters as part of the URL, leading to 404 errors:

```markdown
[中文文档](https://example.com/doc)**重要**
```

In this example, `**重要**` becomes part of the URL: `https://example.com/doc**重要**` which results in a 404 error.

## Solution

Always add a space after the closing bracket of a URL in Markdown:

```markdown
[中文文档](https://example.com/doc) **重要**
```

The space acts as a clear delimiter, ensuring the URL ends where intended.

## Example

**Wrong way** (causes 404):
```markdown
查看[快速入门指南](https://example.com/guide)了解更多细节。
```
Resulting URL: `https://example.com/guide了解更多细节。`

**Right way** (works correctly):
```markdown
查看[快速入门指南](https://example.com/guide) 了解更多细节。
```
Resulting URL: `https://example.com/guide`

## Pitfalls

- This applies to all URL formats in Markdown: `[text](url)`, `![alt](image-url)`, and reference-style links
- The issue is particularly common with Chinese text where spaces are less visually obvious
- Some Markdown parsers are more lenient than others, but for maximum compatibility always include the space
- Don't confuse this with the space *inside* the brackets - that's for readability and doesn't affect URL parsing

## See also

- [chinese-markdown-style](../chinese-markdown-style/SKILL.md)
- [safe-markdown-auto-fix](../safe-markdown-auto-fix/SKILL.md)
