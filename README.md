# Logseq Treecraft Plugin

Treecraft is an experimental Logseq plugin for constructing AI prompts and tree-based problem solving.

Motivation and limitations of linear chat-based interfaces are described in the blog post [Beyond Chat: Tree-based Problem Solving](https://dundalek.com/entropic/tree-based/).
By building on top of existing outliner like Logseq, we leverage existing functionality for manipulating trees and can focus on experimenting with the prompt building features.

It helps to reduce polluting the context window with non-viable branches by allowing you to explore multiple solution paths in a tree structure while keeping only relevant branches in your prompt context.

![Treecraft example](https://github.com/user-attachments/assets/ce2a1b1d-bf94-4a9b-82ca-cd9f1d31a7cc)

**Status**: 🚧 [Scrappy Fiddle](https://dundalek.com/entropic/scrappy-fiddles) 🚧 incomplete proof-of-concept. Prompts have to be copied to clipboard and pasted. If the concept proves useful, tighter integration with LLMs could be implemented.

## Installation

### Manual Installation

1. Download the latest release from the [releases page](https://github.com/dundalek/logseq-treecraft/releases)
2. Unzip the package
3. Open Logseq and go to `Settings` → `Plugins` → `Load unpacked plugin`
4. Select the unzipped folder

## Usage

The panel shows preview of the constructed prompt that can be copied to clipboard.

**Open the panel**

- Toolbar button: Click the tree icon in the toolbar
- Command palette: `Treecraft: Toggle panel`
- Keyboard shortcut: `Cmd+Alt+T` (Mac) or `Ctrl+Alt+T` (Windows/Linux)

**Copy prompt to clipboard**

- From command palette invoke `Treecraft: Copy prompt to clipboard`
- Default keyboard shortcuts are `Cmd+Alt+C` (Mac) or `Ctrl+Alt+C` (Windows/Linux)
- Or click the copy button in the panel

**Insert directives via slash commands**

Type `/` while editing a block to access slash commands that start with `Treecraft: ` as a shorthand to insert directives.

### Basic Prompting

Treecraft collects blocks by traversing up the tree from your cursor position, including:
- All ancestor blocks (from root to cursor)
- Preceding siblings at each level
- Filtered according to directives

**Example:**

When your cursor is here:

```
- a1
- a2
  - b1
  - b2
    - c <cursor-here>
```

The collected prompt will be:

```
a1
a2
b1
b2
c
```

**Example with cursor in the middle:**

```
- a1
- a2
  - b1 <cursor-here>
  - b2
    - c
```

Results in:

```
a1
a2
b1
```

Note that blocks after the cursor (`b2` and `c`) are not included.

**Example with directives:**

You can also use special keywords to control how prompt is constructed like the OPTION directive.

```
- Problem statement
- OPTION Approach A
  - Details of approach A <cursor-here>
- OPTION Approach B
  - Details of approach B
```

Will result in:

```
Problem statement
OPTION Approach A
Details of approach A
```

Choosing the other option:

```
- Problem statement
- OPTION Approach A
  - Details of approach A
- OPTION Approach B
  - Details of approach B <cursor-here>
```

Result:

```
Problem statement
OPTION Approach B
Details of approach B
```

### Treecraft Panel


The panel contains following tabs:

**Prompt Tab**

Shows the constructed prompt for the current cursor position which updates as you navigate blocks.

**Tasks Tab**

Lists actionable items that includes blocks marked with `QUESTION` directive and Logseq `TODO` and `DOING` task markers.

Controls:
- Use the toggle to switch between:
  - Page: Shows all tasks on the current page
  - Thread: Shows tasks within the current thread scope (defined by the closest `THREAD` directive ancestor)
- Toggle filtering of tasks based on `BLOCKER` directives.
  - When filtering is enabled (default): Only unblocked tasks are shown
  - When filtering is disabled: All tasks are shown, including blocked ones
- Click any task to navigate to that block. Hold `Shift` while clicking to open the block in the sidebar.

## Directives

Directives are special keywords that control how blocks are processed. They must be in ALL CAPS at the start of a block's content.

**Available directives:**

- [**THREAD**](#thread) - Creates a scope boundary, stops ancestor traversal at this point
- [**OPTION**](#option) - Marks alternative branches, excluded unless cursor is within
- [**PREPEND**](#prepend) - Moves content to the beginning of the prompt
- [**NOTE**](#note) - Completely excluded from collected prompts
- [**USER**](#user) - Marks user messages, directive stripped from output
- [**ASSISTANT**](#assistant) - Marks assistant messages, directive stripped from output
- [**QUESTION**](#question) - Marks blocks for tracking in Tasks tab
- [**BLOCKER**](#blocker) - Marks blocking items for tracking in Tasks tab

### THREAD

Creates a scope boundary for context collection. When traversing ancestors, collection stops at the closest `THREAD` block (inclusive).

**Use case**: Start a new conversation thread or problem-solving session without including earlier context.

**Example:**
```
- Earlier discussion...
- THREAD New conversation starts here
  - User question
  - Response <cursor-here>
```

Only content from "New conversation starts here" onward is collected.

### OPTION

Marks alternative branches that should be excluded from the prompt unless the cursor is within that branch.

**Use case**: Explore multiple approaches without polluting the context with non-viable alternatives.

**Example:**
```
- Problem statement
- OPTION Approach A
  - Details of approach A
- OPTION Approach B
  - Details of approach B <cursor-here>
```

Only "Approach B" and its details are included in the prompt.

### PREPEND

Content is moved to the beginning of the collected prompt, regardless of its position in the tree.

**Use case**: Add system instructions or context that should always appear first.

**Example:**
```
- Lorem ipsum dolor sit amet...
- PREPEND Summarize article below: <cursor-here>
```

Results in:
```
Summarize article below:
Lorem ipsum dolor sit amet...
```

### NOTE

Blocks marked with `NOTE` are completely excluded from the collected prompt.

**Use case**: Add personal notes, reminders, or meta-commentary that shouldn't be sent to an AI.

**Example:**
```
- Main content
- NOTE Remember to review this later
- More content <cursor-here>
```

The NOTE block is excluded from the output.

```
Main content
More content
```

### USER

Marks the block as a user message. The directive is stripped from the collected content.

**Use case**: Structure chat-style conversations with explicit turn-taking.

**Example:**
```
- USER What is the capital of France?
- ASSISTANT The capital of France is Paris.
- USER What is its population? <cursor-here>
```

### ASSISTANT

Marks the block as an assistant message. The directive is stripped from the collected content.

**Use case**: Record AI responses in a structured format within your notes.

### QUESTION

Marks blocks as questions for tracking in the Tasks tab. The directive is stripped from the output when collecting prompts.

**Use case**: Track unanswered questions or discussion points.

**Example:**
```
- QUESTION What is the best approach here?
- QUESTION Need to verify this assumption
```

### BLOCKER

Marks blocks as blocking items that affect task visibility in the Tasks tab. The directive is stripped from the output when collecting prompts.

**Use case**: Blockers are used to mark that a solution is not viable and other tasks in a tree can be pruned.

**Blocking rules:**  A task (QUESTION, TODO, DOING) is considered blocked when:

1. **Descendant of BLOCKER**: Any task nested under a BLOCKER block is blocked.
   ```
   - BLOCKER Waiting for API access
     - TODO Implement API integration  ← blocked
     - QUESTION Which endpoint to use?  ← blocked
   ```

2. **Sibling of BLOCKER under a task**: When a BLOCKER is a direct child of a task node, it blocks sibling tasks and their descendants.
   ```
   - QUESTION Main question
     - BLOCKER Need clarification first
     - TODO Subtask A  ← blocked (sibling of BLOCKER under task)
       - QUESTION Nested question  ← blocked (descendant of blocked sibling)
   ```

**Exception with OPTION**: When a BLOCKER also has the OPTION directive, it does NOT block siblings. This allows modeling optional blocking scenarios where only one path may be blocked.

```
- QUESTION Choose approach
  - OPTION BLOCKER Path A (blocked by external dependency)
  - OPTION QUESTION Path B  ← NOT blocked (BLOCKER is an OPTION)
```

## Privacy

Treecraft operates entirely locally within Logseq. All your data, prompts, and notes remain on your device and are never transmitted over the network. The plugin does not collect, store, or send any user information to external servers.

When you use the clipboard copy functionality, the prompt content is only placed in your system clipboard for you to manually paste wherever you choose.

## License

Treecraft is currently not open source. It is distributed under [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/).
