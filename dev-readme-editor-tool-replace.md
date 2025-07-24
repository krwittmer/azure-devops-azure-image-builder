
# 🔧 Placeholder Replacement Guide

This guide shows how to replace template placeholders such as `<subscriptionId>`, `<identityRg>`, etc., using different tools and environments.

---

## ✅ 1. Visual Studio Code (VS Code)

### 🧰 Method: Multi-Cursor Find & Replace

1. Open the file in VS Code.
2. Press `Ctrl + H` (or `Cmd + H` on macOS) to open the **Find and Replace** panel.
3. In the "Find" field, enter the placeholder (e.g., `<subscriptionId>`).
4. In the "Replace" field, enter the real value (e.g., `12345678-1234-1234-1234-123456789abc`).
5. Press `Enter` or click "Replace All" (`Alt + Enter` to preview).
6. Repeat for each placeholder.

💡 You can use **regular expressions** by clicking the `.*` icon in the Find bar:
```regex
<([^>]+)>
```
This matches all placeholders surrounded by angle brackets.

---

## ✅ 2. WSL (Windows Subsystem for Linux)

### 🧰 Method: `sed` Command Line Substitution

```bash
# Replace <subscriptionId>
sed -i 's|<subscriptionId>|12345678-1234-1234-1234-123456789abc|g' imageTemplate.json

# Replace additional placeholders
sed -i 's|<identityRg>|aib-identities|g' imageTemplate.json
sed -i 's|<stagingRg>|aib-staging-rg|g' imageTemplate.json
```

- `-i` edits the file in place.
- `|` is used as a delimiter to avoid escaping `/`.

---

## ✅ 3. Windows CMD (Command Prompt)

### 🧰 Method: Using PowerShell via CMD

```cmd
powershell -Command "(Get-Content imageTemplate.json) -replace '<subscriptionId>', '12345678-1234-1234-1234-123456789abc' | Set-Content imageTemplate.json"
```

For multiple replacements:

```cmd
powershell -Command "(Get-Content imageTemplate.json) -replace '<identityRg>', 'aib-identities' -replace '<stagingRg>', 'aib-staging-rg' | Set-Content imageTemplate.json"
```

---

## 🧪 Best Practices

- Use **scripts** for repeated replacements.
- Keep a `template.json` version intact and generate a `rendered.json` for deployment.
- Consider using **ARM parameter files** or switching to **Bicep** for maintainability.

Let me know if you'd like a cross-platform reusable script to automate this process!
