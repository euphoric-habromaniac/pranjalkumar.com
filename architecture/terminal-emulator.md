# Terminal Emulator

> The command pipeline, type system, tab completion, history persistence and SPA navigation.

---

## What it is

A structured command interpreter, not a styled `<textarea>`. It has a typed command pipeline, 26 commands, tab completion with argument awareness, typo suggestions, persistent history, SPA navigation, async support, and man pages.

---

## Component architecture

| File | Responsibility |
|---|---|
| `TerminalMode.tsx` | Full-screen overlay with AnimatePresence. Wires `useTerminal` to input/output. |
| `TerminalBoot.tsx` | Boot sequence animation — simulated kernel messages. |
| `TerminalInput.tsx` | Input line, blinking cursor, ghost suggestions, keyboard handling. |
| `TerminalOutput.tsx` | Scrollable output buffer with auto-scroll. |
| `TerminalLine.tsx` | Line renderer — maps `TerminalLineType` to color tokens. |
| `useTerminal.ts` | State hook — owns lines, history, submission pipeline. |
| `commands/index.ts` | Command registry, alias resolution, `processCommand`. |
| `commands/suggest.ts` | Closest-match search behind the "did you mean" hint. |

---

## The type system

### TerminalLine — the output unit

```typescript
interface TerminalLine {
  id: string;                            // nanoid — unique React key
  type: TerminalLineType;                // 'output' | 'command' | 'error' | 'success' | 'info' | 'boot' | 'label' | 'sys' | 'proj'
  content: string | TerminalBlock[];     // plain text or rich colored segments
  isCommand?: boolean;
}

interface TerminalBlock {
  text: string;
  color?: 'mint' | 'lavender' | 'peach' | 'sky' | 'dim' | 'default';
  bold?: boolean;
  run?: string;                          // if set, clicking the text runs this command
}
```

### ProcessResult — the return type

```typescript
type ProcessResult =
  | TerminalLine[]           // normal output
  | 'CLEAR'                  // clear the screen
  | 'EXIT'                   // close the terminal
  | `NAVIGATE:${string}`     // SPA navigation
  | Promise<TerminalLine[]>  // async output (API calls)
```

The result is a flat union, and the template literal `NAVIGATE:${string}` is the decision that matters.

#### Why `NAVIGATE:${string}` instead of `{ type: 'navigate', to: string }`?

Using a tagged string kept `ProcessResult` as a flat union. Changing to a discriminated union object would have required refactoring 20+ files. The template literal provides type safety — TypeScript enforces the `NAVIGATE:` prefix — without object overhead.

The handler is two lines:

```typescript
if (typeof result === 'string' && result.startsWith('NAVIGATE:')) {
  onExit();
  onNavigate(result.slice('NAVIGATE:'.length));
  return;
}
```

---

## Command pipeline

```
Input → trim → split → resolve aliases → registry lookup → handler(args) → ProcessResult
```

1. **Alias resolution** — `cls` → `clear`, `certs` → `cert`, `work` → `internship`, `hire` → `tldr`, `experience` → `exp`, `posts` → `blog`, `?` → `help`
2. **Special cases** — `exit`/`quit` return `'EXIT'`. `man <cmd>` invokes `showManPage()`.
3. **Not found** — typo suggestions: the closest command names appear as "did you mean: ...", and each one is clickable
4. **Async handling** — Promise results show `querying...` loading line, replaced when resolved

---

### Clickable output

A text block can carry a `run` string. Clicking it submits that command as if it had been typed, so project titles, suggestions and follow-up hints in the output are links into the terminal itself.

### Consent before network calls

`whoami` and `scan` look up the visitor's public IP through a third-party service. By default they do nothing except print what they would do. Running them again with `--yes` (the hint is clickable) goes ahead.

---

## All 26 commands

| Command | Description | Type |
|---|---|---|
| `help` | Available commands | sync |
| `tldr` | The 20-second version: status, top projects, contact. Alias `hire` | sync |
| `exp` | Experience timeline. Alias `experience` | sync |
| `blog` | List or open blog posts. Alias `posts` | sync/navigate |
| `theme` | Switch light, dark or system theme | sync |
| `about` | Who I am | sync |
| `projects` | List projects. `--details <id>` prints a breakdown, `--open <id>` opens the on-site page, `--repo <id>` opens GitHub | sync/navigate |
| `skills` | Tech skills grid | sync |
| `contact` | Contact info | sync |
| `internship` | Current internship | sync |
| `whoami` | IP, location, ISP via ipapi.co, after `--yes` | **async** |
| `scan` | Simulated port scan of your own IP, after `--yes` | **async** |
| `ping` | Simulated ping | **async** |
| `open` | Open social profile | sync |
| `ls` | List filesystem | sync |
| `cat` | Read file (about.txt, resume.pdf, etc.) | sync/navigate |
| `sudo` | Always fails sarcastically | sync |
| `cert` | Certifications | sync |
| `history` | Command history | sync |
| `date` | Current date/time | sync |
| `echo` | Print text | sync |
| `uname` | System info | sync |
| `neofetch` | ASCII system info | sync |
| `clear` | Clear terminal | sync |
| `design` | Navigate to /design | navigate |
| `resume` | Open the résumé page | navigate |
| `exit`/`quit` | Close terminal (handled outside the registry, not counted above) | exit |

---

## Tab completion

Context-aware. `getArgCompletions(cmd, priorArgs)` returns different lists per command:

```typescript
case 'open':      → ['github', 'linkedin', 'email']
case 'projects':  → ['--open', '--details', '--repo'] → then project IDs
case 'help':      → all command names
case 'cat':       → ['about.txt', 'skills.txt', 'README.md', 'resume.pdf', ...]
case 'ls':        → ['-la', '-l', '-a']
```

Tab cycles through candidates. Hint row shows all matches below the prompt.

---

## History persistence

Stored in `localStorage` as JSON under `terminal_history`. Arrow Up/Down navigates. Survives reloads, browser restarts, terminal close/reopen. Private browsing handled with `try/catch`.

---

## Man pages

Every command has a `detail` field. `man <cmd>` or `help <cmd>` renders a formatted page with description and usage.
