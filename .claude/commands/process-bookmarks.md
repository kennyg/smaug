# /process-bookmarks

Process prepared Twitter bookmarks into a markdown archive with rich analysis and optional filing to a knowledge library.

## Before You Start

### Multi-Step Parallel Protocol (CRITICAL)

**Create todo list IMMEDIATELY after reading bookmark count.** This ensures final steps never get skipped.

**For 1-2 bookmarks (sequential):**
```javascript
TodoWrite({ todos: [
  {content: "Read pending bookmarks", status: "pending", activeForm: "Reading pending bookmarks"},
  {content: "Process bookmark 1", status: "pending", activeForm: "Processing bookmark 1"},
  {content: "Process bookmark 2", status: "pending", activeForm: "Processing bookmark 2"},
  {content: "Clean up pending file", status: "pending", activeForm: "Cleaning up pending file"},
  {content: "Commit and push changes", status: "pending", activeForm: "Committing changes"},
  {content: "Return summary", status: "pending", activeForm: "Returning summary"}
]})
```

**For 3+ bookmarks (MUST use parallel subagents with batch files):**
```javascript
TodoWrite({ todos: [
  {content: "Read pending bookmarks", status: "pending", activeForm: "Reading pending bookmarks"},
  {content: "Spawn subagents to write batch files", status: "pending", activeForm: "Spawning subagents"},
  {content: "Wait for all subagents to complete", status: "pending", activeForm: "Waiting for subagents"},
  {content: "Merge batch files into bookmarks.md", status: "pending", activeForm: "Merging batch files"},
  {content: "Clean up batch and pending files", status: "pending", activeForm: "Cleaning up files"},
  {content: "Commit and push changes", status: "pending", activeForm: "Committing changes"},
  {content: "Return summary", status: "pending", activeForm: "Returning summary"}
]})
```

**Execution rules:**
- Mark each step `in_progress` before starting
- Mark `completed` immediately after finishing (no batching)
- Only ONE task `in_progress` at a time
- Never skip final steps (commit, summary)

**CRITICAL for 3+ bookmarks:** Spawn ALL subagents in ONE message, each writing to a batch file:
```javascript
// Send ONE message with multiple Task calls - they run in parallel
// Use model="haiku" for cost-efficient parallel processing (~50% cost savings)
// Each subagent writes to .state/batch-N.md, NOT to bookmarks.md!
Task(subagent_type="general-purpose", model="haiku", prompt="Process batch 0: write to .state/batch-0.md: {json for bookmarks 0-4}")
Task(subagent_type="general-purpose", model="haiku", prompt="Process batch 1: write to .state/batch-1.md: {json for bookmarks 5-9}")
Task(subagent_type="general-purpose", model="haiku", prompt="Process batch 2: write to .state/batch-2.md: {json for bookmarks 10-14}")
// ... all batches in the SAME message
```

After ALL subagents complete, merge batch files into bookmarks.md in chronological order.

**DO NOT:**
- Have subagents write directly to bookmarks.md (race conditions!)
- Process 3+ bookmarks sequentially (too slow)
- Send Task calls in separate messages (defeats parallelism)
- Skip the merge step

### Setup

**Get today's date (friendly format):**
```bash
date +"%A, %B %-d, %Y"
```

Use this format for date section headers (e.g., "Thursday, January 2, 2026").

**Load paths and categories from config:**
```bash
cat ./smaug.config.json | jq '{archiveFile, pendingFile, stateFile, categories, obsidian}'
```

This gives you:
- `archiveFile`: Where to write the bookmark archive (e.g., `~/Obsidian_Vaults/.../bookmarks.md`)
- `pendingFile`: Where pending bookmarks are stored
- `stateFile`: Where processing state is tracked
- `categories`: Custom category definitions
- `obsidian`: Obsidian vault settings (if enabled)

**IMPORTANT:** Use these paths throughout. The `~` will be the user's home directory.
If no custom categories, use the defaults from `src/config.js`.

**Check for Obsidian integration:**
If `obsidian.enabled` is true (or `OBSIDIAN_VAULT_PATH` env var is set), you'll also export to Obsidian vault. See the "Obsidian Integration" section below.

## Input

Prepared bookmarks are in the `pendingFile` path from config (typically `./.state/pending-bookmarks.json` or a custom path).

Each bookmark includes:
- `id`, `author`, `authorName`, `text`, `tweetUrl`, `date`
- `tags[]` - folder tags from bookmark folders (e.g., `["ai-tools"]`)
- `links[]` - each with `original`, `expanded`, `type`, and `content`
  - `type`: "github", "article", "video", "tweet", "media", "image"
  - `content`: extracted text, headline, author (for articles/github)
- `isReply`, `replyContext` - parent tweet info if this is a reply
- `isQuote`, `quoteContext` - quoted tweet info if this is a quote tweet

## Categories System

Categories define how different bookmark types are handled. Each category has:
- `match`: URL patterns or keywords to identify this type
- `action`: What to do with matching bookmarks
  - `file`: Create a separate markdown file in the folder
  - `capture`: Just add to bookmarks.md
  - `transcribe`: Flag for future transcription, add to bookmarks.md with transcript note
- `folder`: Where to save files (for `file` action)
- `template`: Which template to use (`tool`, `article`, `podcast`, `video`)

**Default categories:**
| Category | Match Patterns | Action | Folder |
|----------|---------------|--------|--------|
| github | github.com | file | ./knowledge/tools |
| article | medium.com, substack.com, dev.to, blog | file | ./knowledge/articles |
| podcast | podcasts.apple.com, spotify.com/episode, overcast.fm | transcribe | ./knowledge/podcasts |
| youtube | youtube.com, youtu.be | transcribe | ./knowledge/videos |
| video | vimeo.com, loom.com | transcribe | ./knowledge/videos |
| tweet | (fallback) | capture | - |

## Workflow

### 1. Read the Prepared Data

Read from the `pendingFile` path specified in config. If the path starts with `~`, expand it to `$HOME`:
```bash
# Get pendingFile from config and expand ~
PENDING_FILE=$(cat ./smaug.config.json | jq -r '.pendingFile' | sed "s|^~|$HOME|")
cat "$PENDING_FILE"
```

### 2. Process Bookmarks (Parallel for 3+)

**IMPORTANT: If there are 3 or more bookmarks, you MUST use parallel processing:**

```
Use the Task tool to spawn multiple subagents simultaneously.
Each subagent processes a batch of ~5 bookmarks.
Example: 20 bookmarks → spawn 4 subagents (5 each) in ONE message with multiple Task calls.
```

This is critical for performance. Do NOT process bookmarks sequentially when there are 3+.

For each bookmark (or batch):

#### a. Determine the best title/summary

Don't use generic titles like "Article" or "Tweet". Based on the content:
- GitHub repos: Use the repo name and brief description
- Articles: Use the article headline or key insight
- Videos: Note for transcript, use tweet context
- Quote tweets: Capture the key insight being highlighted
- Reply threads: Include parent context in the summary
- Plain tweets: Use the key point being made

#### b. Categorize using the categories config

Match each bookmark's links against category patterns (check `match` arrays). Use the first matching category, or fall back to `tweet`.

**For each action type:**
- `file`: Create a separate file in the category's folder using its template
- `capture`: Just add to bookmarks.md (no separate file)
- `transcribe`: Add to bookmarks.md with a "Needs transcript" flag, optionally create placeholder in folder

**Special handling:**
- Quote tweets: Include quoted tweet context in entry
- Reply threads: Include parent context in entry

#### c. Write bookmark entry

Add to the `archiveFile` path from config (expand `~` to home directory):

**CRITICAL ordering rules for bookmarks.md:**

The file must be in **descending chronological order** (newest dates at TOP, oldest at BOTTOM).

1. **Read the existing file structure first** - note all existing date sections and their positions
2. Use each bookmark's `date` field (already formatted as "Weekday, Month Day, Year")
3. **For each bookmark's date:**
   - If that date section already exists: insert the entry immediately AFTER the `# Date` header (above other entries in that section)
   - If no section exists for that date: create a new `# Weekday, Month Day, Year` section at the **correct chronological position** (NOT always at top!)
4. **Chronological positioning for new date sections:**
   - Find where the date belongs chronologically among existing sections
   - Insert BEFORE any older dates, AFTER any newer dates
   - Example: If file has "Jan 3" then "Jan 1", and you need "Jan 2", insert between them
5. Do NOT create duplicate date sections - always search the entire file first
6. Separate date sections with `---`

**Processing order:** Bookmarks in pending-bookmarks.json are sorted oldest-first. Process them in order so that when each is inserted at the top of its date section, the final result has correct ordering within each day.

**Header hierarchy:**
- `# Thursday, January 2, 2026` - Date headers (h1)
- `## @author - title` - Individual bookmark entries (h2)

**Standard entry format:**
```markdown
## @{author} - {descriptive_title}
> {tweet_text}

- **Tweet:** {tweet_url}
- **Link:** {expanded_url}
- **Tags:** [[tag1]] [[tag2]] (if bookmark has tags from folders)
- **Filed:** [{filename}](./knowledge/tools/{slug}.md) (if filed)
- **What:** {1-2 sentence description of what this actually is}
```

**Tags format:** Use wiki-link style `[[TagName]]` for each tag. Only include the **Tags:** line if the bookmark has tags in its `tags` array (from folder configuration). Example: `- **Tags:** [[AI]] [[Coding]]`

**For quote tweets, include the quoted content:**
```markdown
## @{author} - {descriptive_title}
> {tweet_text}
>
> *Quoting @{quoted_author}:* {quoted_text}

- **Tweet:** {tweet_url}
- **Quoted:** {quoted_tweet_url}
- **Tags:** [[tag1]] [[tag2]] (if bookmark has tags)
- **What:** {description}
```

**For replies, include parent context:**
```markdown
## @{author} - {descriptive_title}
> *Replying to @{parent_author}:* {parent_text}
>
> {tweet_text}

- **Tweet:** {tweet_url}
- **Parent:** {parent_tweet_url}
- **Tags:** [[tag1]] [[tag2]] (if bookmark has tags)
- **What:** {description}
```

Separate entries with `---` only between different dates, not between entries on the same day.

### 3. Clean Up Pending File

After successfully processing, remove the processed bookmarks from the pending file (use `pendingFile` path from config, expanding `~`):

```javascript
const pendingPath = config.pendingFile.replace(/^~/, process.env.HOME);
const pending = JSON.parse(fs.readFileSync(pendingPath, 'utf8'));
const processedIds = new Set([/* IDs you processed */]);
const remaining = pending.bookmarks.filter(b => !processedIds.has(b.id));
pending.bookmarks = remaining;
pending.count = remaining.length;
fs.writeFileSync(pendingPath, JSON.stringify(pending, null, 2));
```

### 4. Commit and Push Changes

After all bookmarks are processed and filed, commit the changes:

```bash
# Get today's date for commit message
DATE=$(date +"%b %-d")

# Stage all bookmark-related changes (use archiveFile path from config)
git add "$ARCHIVE_FILE"  # The archiveFile path from config
git add knowledge/

# Commit with descriptive message
git commit -m "Process N Twitter bookmarks from $DATE

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"

# Push immediately
git push
```

Replace "N" with actual count. If any knowledge files were created, mention them in the commit message body.

### 5. Return Summary

```
Processed N bookmarks:
- @author1: Tool Name → filed to knowledge/tools/tool-name.md
- @author2: Article Title → filed to knowledge/articles/article-slug.md
- @author3: Plain tweet → captured only

Committed and pushed.
```

## Frontmatter Templates

### Tool Entry (`./knowledge/tools/{slug}.md`)

```yaml
---
title: "{tool_name}"
type: tool
date_added: {YYYY-MM-DD}
source: "{github_url}"
tags: [{relevant_tags}, {folder_tags}]
via: "Twitter bookmark from @{author}"
---

{Description of what the tool does, key features, why it was bookmarked}

## Key Features

- Feature 1
- Feature 2

## Links

- [GitHub]({github_url})
- [Original Tweet]({tweet_url})
```

### Article Entry (`./knowledge/articles/{slug}.md`)

```yaml
---
title: "{article_title}"
type: article
date_added: {YYYY-MM-DD}
source: "{article_url}"
author: "{article_author}"
tags: [{relevant_tags}, {folder_tags}]
via: "Twitter bookmark from @{author}"
---

{Summary of the article's key points and why it was bookmarked}

## Key Takeaways

- Point 1
- Point 2

## Links

- [Article]({article_url})
- [Original Tweet]({tweet_url})
```

### Podcast Entry (`./knowledge/podcasts/{slug}.md`)

```yaml
---
title: "{episode_title}"
type: podcast
date_added: {YYYY-MM-DD}
source: "{podcast_url}"
show: "{show_name}"
tags: [{relevant_tags}, {folder_tags}]
via: "Twitter bookmark from @{author}"
status: needs_transcript
---

{Brief description from tweet context}

## Episode Info

- **Show:** {show_name}
- **Episode:** {episode_title}
- **Why bookmarked:** {context from tweet}

## Transcript

*Pending transcription*

## Links

- [Episode]({podcast_url})
- [Original Tweet]({tweet_url})
```

### Video Entry (`./knowledge/videos/{slug}.md`)

```yaml
---
title: "{video_title}"
type: video
date_added: {YYYY-MM-DD}
source: "{video_url}"
channel: "{channel_name}"
tags: [{relevant_tags}, {folder_tags}]
via: "Twitter bookmark from @{author}"
status: needs_transcript
---

{Brief description from tweet context}

## Video Info

- **Channel:** {channel_name}
- **Title:** {video_title}
- **Why bookmarked:** {context from tweet}

## Transcript

*Pending transcription*

## Links

- [Video]({video_url})
- [Original Tweet]({tweet_url})
```

## Parallel Processing (REQUIRED for 3+ bookmarks)

**CRITICAL: Subagents must NOT write directly to bookmarks.md** - this causes race conditions and scrambled ordering.

### Two-Phase Approach:

**Phase 1: Parallel batch processing (subagents write to temp files)**

Spawn multiple Task subagents in ONE message. Each writes to a separate temp file:

```
Task 1: model="haiku", "Process batch 0" → writes to .state/batch-0.md
Task 2: model="haiku", "Process batch 1" → writes to .state/batch-1.md
Task 3: model="haiku", "Process batch 2" → writes to .state/batch-2.md
Task 4: model="haiku", "Process batch 3" → writes to .state/batch-3.md
```

**Subagent prompt template:**
```
Process these bookmarks and write ONLY the markdown entries (no date headers) to .state/batch-{N}.md

Bookmarks to process (in order - oldest first):
{JSON array of 5-10 bookmarks}

For each bookmark, write an entry in this format:
---
DATE: {bookmark.date}
## @{author} - {title}
> {tweet text}

- **Tweet:** {url}
- **Tags:** [[tag1]] [[tag2]] (if tags exist)
- **What:** {description}

Also create knowledge files (./knowledge/tools/*.md, ./knowledge/articles/*.md) as needed.
DO NOT touch bookmarks.md - only write to .state/batch-{N}.md
```

**Phase 2: Sequential merge (main agent combines batches)**

After ALL subagents complete:
1. Read all .state/batch-*.md files in order (batch-0, batch-1, batch-2...)
2. Parse each entry (separated by `---`) and extract the DATE line
3. Insert each entry into bookmarks.md at the correct chronological position
4. Delete the temp batch files

**Merge logic for bookmarks.md:**
- File is descending order (newest dates at top)
- For each entry from batch files (processed in order):
  - Find or create the date section at correct position
  - Insert entry at TOP of that date section
- Since batches are oldest-first, entries end up in correct order

**DO NOT:**
- Have subagents write directly to bookmarks.md (causes race conditions)
- Process all bookmarks sequentially (too slow)
- Skip the merge step

## Example Output

```
Processed 4 bookmarks:

1. @tom_doerr: Whisper-Flow (Real-time Transcription)
   → Tool: github.com/dimastatz/whisper-flow
   → Filed: knowledge/tools/whisper-flow.md

2. @simonw: Gist Host Fork for Rendering GitHub Gists
   → Article about GitHub Gist rendering
   → Filed: knowledge/articles/gisthost-gist-rendering.md

3. @michael_chomsky: ResponsiveDialog Component Pattern
   → Quote tweet endorsing @jordienr's UI pattern
   → Captured with quoted context

4. @CasJam: Claude Code Video Post-Production
   → Plain tweet (video content)
   → Captured only, flagged for transcript
```

## Obsidian Integration

When `obsidian.enabled` is true (or `OBSIDIAN_VAULT_PATH` env var is set), export bookmarks to an Obsidian vault in addition to the local archive.

### Check Obsidian Config

```bash
# Check if Obsidian is enabled
OBSIDIAN_ENABLED=$(cat ./smaug.config.json | jq -r '.obsidian.enabled // false')
OBSIDIAN_VAULT=$(echo "${OBSIDIAN_VAULT_PATH:-$(cat ./smaug.config.json | jq -r '.obsidian.vaultPath // empty')}" | sed "s|^~|$HOME|")

if [ "$OBSIDIAN_ENABLED" = "true" ] || [ -n "$OBSIDIAN_VAULT" ]; then
  echo "Obsidian export enabled to: $OBSIDIAN_VAULT"
fi
```

### Obsidian Folder Structure

Within the vault, create this structure:
```
{vaultPath}/
├── {bookmarksFolder}/           # e.g., "Twitter Captures"
│   ├── bookmarks.md             # Main archive (same format, Obsidian-enhanced)
│   ├── 2026-01-04.md            # Daily note (if using daily notes style)
│   └── Knowledge/               # {knowledgeFolder}
│       ├── Tools/
│       │   └── whisper-flow.md
│       └── Articles/
│           └── gisthost-rendering.md
```

### Obsidian Entry Format

Use Obsidian-specific formatting features:

**1. YAML Frontmatter** (when `obsidian.frontmatter` is true):
```yaml
---
title: "Whisper-Flow - Real-time Transcription Tool"
date: 2026-01-04
type: twitter-bookmark
author: "@tom_doerr"
tweet_url: "https://x.com/tom_doerr/status/123456"
tags:
  - twitter
  - tool
  - ai
aliases:
  - whisper-flow
---
```

**2. Wikilinks** (when `obsidian.wikilinks` is true):
- Link to knowledge files: `[[whisper-flow|Whisper-Flow]]`
- Link to related notes: `[[AI Tools]]`, `[[GitHub Projects]]`
- Author tags as wikilinks: `[[People/@tom_doerr|@tom_doerr]]`

**3. Hashtags** (when `obsidian.hashtags` is true):
- Convert folder tags to hashtags: `#ai-tools`, `#coding`
- Add type hashtags: `#twitter/bookmark`, `#tool`, `#article`

### Obsidian Bookmark Entry Template

```markdown
---
title: "{descriptive_title}"
date: {YYYY-MM-DD}
type: twitter-bookmark
author: "@{author}"
source: "{tweet_url}"
links:
  - "{expanded_url}"
tags:
  - twitter/bookmark
  - {category}
  - {folder_tags}
---

## @{author} - {descriptive_title}

> {tweet_text}

### Details

- **Tweet:** [{tweet_url}]({tweet_url})
- **Link:** [{domain}]({expanded_url})
- **Tags:** #twitter #{category} #{folder_tags}
- **Filed:** [[{knowledge_file}|{title}]]
- **What:** {1-2 sentence description}

### Related

- [[{category} Notes]]
- [[Twitter Bookmarks]]
```

### Obsidian Knowledge File Template

```markdown
---
title: "{tool_name}"
date: {YYYY-MM-DD}
type: tool
source: "{github_url}"
via: "@{twitter_author}"
stars: {star_count}
language: "{primary_language}"
tags:
  - tool
  - {language}
  - {topics}
aliases:
  - {repo_name}
---

# {tool_name}

{Description of what the tool does}

## Key Features

- Feature 1
- Feature 2

## Links

- **GitHub:** [{owner}/{repo}]({github_url})
- **Via:** [[@{author}]] - [[{bookmark_title}|Original Tweet]]

## Related

- [[Tools]]
- [[{language} Projects]]
```

### Export Workflow

After processing each bookmark for the local archive:

1. **Check if Obsidian is enabled:**
   ```javascript
   const obsidianVault = process.env.OBSIDIAN_VAULT_PATH || config.obsidian?.vaultPath;
   const obsidianEnabled = config.obsidian?.enabled || !!obsidianVault;
   ```

2. **Create Obsidian folders if needed:**
   ```bash
   VAULT="${OBSIDIAN_VAULT_PATH:-$HOME/path/to/vault}"
   BOOKMARKS_FOLDER="${OBSIDIAN_BOOKMARKS_FOLDER:-Twitter Captures}"
   mkdir -p "$VAULT/$BOOKMARKS_FOLDER/Knowledge/Tools"
   mkdir -p "$VAULT/$BOOKMARKS_FOLDER/Knowledge/Articles"
   ```

3. **Write Obsidian-formatted entry:**
   - Add YAML frontmatter
   - Use [[wikilinks]] for internal references
   - Add #hashtags for tags
   - Mirror the entry to `{vaultPath}/{bookmarksFolder}/bookmarks.md`

4. **Write knowledge files to Obsidian:**
   - Copy knowledge files (tools, articles) to `{vaultPath}/{knowledgeFolder}/`
   - Use Obsidian-enhanced format with frontmatter

5. **Git operations in Obsidian vault** (if vault is a git repo):
   ```bash
   if [ -d "$VAULT/.git" ]; then
     cd "$VAULT" && git add -A && git commit -m "Add Twitter bookmarks" && git push
   fi
   ```

### Parallel Processing with Obsidian

When using subagents for 3+ bookmarks, each subagent should:
1. Write local batch file as normal (`.state/batch-N.md`)
2. Also write Obsidian batch file (`.state/obsidian-batch-N.md`) with enhanced formatting

During merge phase:
1. Merge local batch files → `bookmarks.md`
2. Merge Obsidian batch files → `{vaultPath}/{bookmarksFolder}/bookmarks.md`
3. Copy knowledge files to both locations

### Example Obsidian Output

**In vault: `Twitter Captures/bookmarks.md`**
```markdown
---
title: Twitter Bookmarks
description: Curated bookmarks from Twitter/X
tags:
  - twitter
  - bookmarks
  - index
---

# Twitter Bookmarks

## Friday, January 3, 2026

### @tom_doerr - Whisper-Flow Real-time Transcription

> This is amazing - real-time transcription that actually works! https://t.co/abc123

- **Tweet:** [View on X](https://x.com/tom_doerr/status/123)
- **Link:** [github.com/dimastatz/whisper-flow](https://github.com/dimastatz/whisper-flow)
- **Tags:** #twitter/bookmark #tool #ai #transcription
- **Filed:** [[whisper-flow|Whisper-Flow Tool]]
- **What:** Real-time speech-to-text transcription tool with high accuracy

---
```

**In vault: `Twitter Captures/Knowledge/Tools/whisper-flow.md`**
```markdown
---
title: Whisper-Flow
date: 2026-01-03
type: tool
source: https://github.com/dimastatz/whisper-flow
via: "@tom_doerr"
stars: 1250
language: Python
tags:
  - tool
  - python
  - ai
  - transcription
  - speech-to-text
aliases:
  - whisper-flow
---

# Whisper-Flow

Real-time speech-to-text transcription using OpenAI's Whisper model with streaming support.

## Key Features

- Real-time transcription with low latency
- Multiple language support
- Easy integration with existing applications

## Links

- **GitHub:** [dimastatz/whisper-flow](https://github.com/dimastatz/whisper-flow)
- **Via:** [[@tom_doerr]] - [[2026-01-03#@tom_doerr - Whisper-Flow|Original Tweet]]

## Related

- [[Tools]]
- [[AI Projects]]
- [[Python Projects]]
```
