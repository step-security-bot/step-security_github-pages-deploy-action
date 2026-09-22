# Security Fixes Applied

This document explains the security vulnerabilities found and how they were fixed.

---

## 1. Command Injection via Commit Message (HIGH)

### The Issue
When users provide a commit message, it was directly put into a git command without protection:
```typescript
git commit -m "${commitMessage}"
```

**Problem:** If a user enters a malicious message like `test'; rm -rf /; echo '`, the shell would execute the `rm -rf` command.

### The Fix
We escape single quotes in the message and wrap it in single quotes:
```typescript
const escapedMessage = commitMessage.replace(/'/g, "'\\''")
git commit -m '${escapedMessage}'
```

**Effect on Flow:** NONE. The commit message received by git is identical. Only the shell interpretation changes - it treats the message as literal text, not commands.

---

## 2. Command Injection via Branch Name (HIGH)

### The Issue
Branch names were directly put into git commands without protection:
```typescript
git ls-remote --heads ${action.repositoryPath} refs/heads/${action.branch}
git fetch ${action.repositoryPath} ${action.branch}:${action.branch}
git rebase ${action.branch}
```

**Problem:** A malicious branch name like `main; rm -rf /;` would execute commands.

### The Fix
Created a helper function to escape all branch names:
```typescript
export const escapeShellArg = (arg: string | undefined): string =>
  arg ? `'${arg.replace(/'/g, "'\\''")}'` : ''
```

Then applied it to all branch usages:
```typescript
git ls-remote --heads ${action.repositoryPath} refs/heads/${escapeShellArg(action.branch)}
```

**Effect on Flow:** NONE. Git receives the exact same branch name. Only shell escaping changes.

---

## 3. Command Injection via Folder Path (MEDIUM)

### The Issue
Folder paths were directly put into shell commands:
```typescript
chmod -R +rw ${action.folderPath}
rsync -q -av ... ${action.folderPath}/.
```

**Problem:** A malicious folder path could inject commands.

### The Fix
Applied the same escaping function to folder paths:
```typescript
chmod -R +rw ${escapeShellArg(action.folderPath)}
rsync -q -av ... ${escapeShellArg(action.folderPath)}/.
```

**Effect on Flow:** NONE. The chmod and rsync commands work with the same paths. Only shell safety changes.

---

## 4. SSH Key Exposure in Error Messages (MEDIUM)

### The Issue
If SSH key setup failed, the private SSH key could appear in error messages:
```
Error: ssh-add failed: my-private-key-content-here
```

**Problem:** Error logs could expose the SSH private key to anyone with workflow log access.

### The Fix
Updated the error redaction function to mask SSH keys:
```typescript
const orderedByLength = (
  [
    action.token,
    action.repositoryPath,
    ...(typeof action.sshKey === 'string' ? [action.sshKey] : [])
  ].filter(Boolean) as string[]
).sort((a, b) => b.length - a.length)

for (const find of orderedByLength) {
  value = replaceAll(value, find, '***')
}
```

Now if an error occurs:
```
Before: Error: ssh-add failed: my-private-key-content-here
After:  Error: ssh-add failed: ***
```

**Effect on Flow:** Error messages are slightly less detailed but secure. Developers can't accidentally expose SSH keys in logs.

---

## Summary

| Issue | Severity | Fix | Effect |
|-------|----------|-----|--------|
| Commit message injection | HIGH | Escape quotes in message | None - same result |
| Branch name injection | HIGH | Escape branch names | None - same result |
| Folder path injection | MEDIUM | Escape folder paths | None - same result |
| SSH key exposure | MEDIUM | Redact keys in errors | Errors less detailed but safe |
| Debug mode tokens | MEDIUM | Intentional feature | None - kept as-is |
| Force push default | LOW | Intentional feature | None - kept as-is |

---

## How Escaping Works

All command injection fixes use the same approach:

1. **Take user input** (commit message, branch name, folder path)
2. **Escape single quotes**: `'` becomes `'\''`
3. **Wrap in single quotes**: `'user-input'`
4. **Pass to shell**

Single quotes in shell mean: "treat everything literally, don't interpret anything."

Example:
```
Input:      don't
Escaped:    'don'\''t'
Shell sees: three parts that join to: don't
Git receives: don't (same as input)
```

**No code flow changes** - just shell safety.
