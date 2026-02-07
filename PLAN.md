# Plan: Skill-Based Meeting Review Architecture

## Why This Approach

### Problems with the current monolith

- **Rigid** — changing a rubric weight means editing Ruby code, re-testing the whole script
- **Double LLM tax** — Ruby calls Gemini to analyze transcripts, but you're already in Claude Code which IS an LLM. Why pay for two?
- **All-or-nothing** — can't just fetch meetings without grading, or re-grade without re-fetching
- **One analysis style** — the rubrics are hardcoded. Can't do a quick scan vs. a deep dive without editing code
- **Opaque** — the grading happens inside Gemini calls; you can't see Claude's reasoning or steer it mid-analysis

### Benefits of the skill-based approach

- **Composable** — `/firefly` and `/review` are independent. Fetch once, review many times. Swap analysis styles freely
- **No external LLM** — Claude does the analysis directly. Better model, zero API cost, and you can converse with it about the results
- **Iterate in plain English** — change your analysis style by editing `analysis.md`, not code. Tweak a weight, add a criterion, change the tone — just edit markdown
- **Multiple analysis modes** — `quick-analysis.md` for a fast morning scan, `deepdive-analysis.md` for board prep or post-mortems. Same data, different lenses
- **Transparent** — Claude shows its reasoning inline. You can push back ("re-score this, you missed the context about X") in conversation
- **Standalone + conversational** — the Ruby fetcher still works from cron/CLI, but now you also have interactive skills in Claude Code

## Key Architectural Shift

- **Before**: Ruby script → Fireflies API → Gemini API → markdown report
- **After**: `/firefly` skill (fetch+cluster+save) → `/review` skill (reads analysis style + data → generates report)
- Claude replaces Gemini as the analyzer — no external LLM needed

## Files to Create

### 1. `analysis.md` — Default Analysis Style

Single source of truth for the user's standard analysis preferences:

- **Meeting types** — definitions, clustering keywords, internal participants
- **Rubrics per type** — criteria, evaluation questions, weights (extracted from current `RUBRICS` hash)
- **Personal preferences** — tone, strictness, focus areas, scoring philosophy
- **Report format** — template for how the output report should look

This is the file the user iterates on to refine their default analysis style.

### 2. `quick-analysis.md` — Fast Morning Scan

A lightweight analysis variant for quick daily triage:

- **Fewer criteria** — only 2-3 highest-weighted criteria per meeting type (skip the long tail)
- **Binary scoring** — pass/fail or 3-tier (good/okay/needs work) instead of 1-10 scale
- **No transcript deep-dives** — score from metadata + summary, not full transcript
- **Compact format** — one-liner per meeting, emoji status, just the action items
- **Use case** — morning standup prep, quick "did anything go wrong yesterday?" check

Example output style:
```
## 2025-01-15 Quick Scan (6 meetings)
- [x] Sprint Planning — decisions clear, actions assigned
- [!] 1:1 with Alex — no blockers discussed, follow up
- [x] Pitch: Acme Corp — strong, schedule follow-up
- [ ] Team Sync — no agenda, ran 15min over
```

### 3. `deepdive-analysis.md` — Comprehensive Deep Dive

A thorough analysis variant for high-stakes review:

- **Full rubric** — all criteria scored 1-10 with evidence quotes from transcript
- **Pattern analysis** — cross-meeting trends (e.g., "you consistently score low on time efficiency across internal meetings")
- **Behavioral observations** — speaking time ratios, question types, interruption patterns
- **Comparative scoring** — how this meeting compares to your averages for this type
- **Coaching recommendations** — specific, behavioral suggestions with example phrases
- **Use case** — weekly retrospective, preparing for board reviews, self-improvement deep dives

### 4. `.claude/commands/firefly.md` — `/firefly` Skill

Claude Code slash command that:
1. Accepts optional date arg (defaults to today)
2. Runs `lib/firefly_fetch.rb <date>` to pull meetings from Fireflies API
3. Reads the chosen analysis style for clustering rules (keywords, internal participants)
4. Clusters each meeting by type with confidence
5. Saves structured JSON to `data/<date>-meetings.json`

### 5. `.claude/commands/review.md` — `/review` Skill

Claude Code slash command that:
1. Accepts optional analysis style arg (defaults to `analysis.md`)
2. Reads latest meeting data from `data/`
3. Reads the specified analysis style for rubrics, preferences, and report format
4. Claude directly grades each meeting transcript against the rubric
5. Generates improvement recommendations for low-scoring areas
6. Writes report to `reports/<date>-meeting-review.md`

Usage: `/review` (default), `/review quick-analysis.md`, `/review deepdive-analysis.md`

### 6. `lib/firefly_fetch.rb` — Standalone Fireflies Client

Extracted from current script. Just the API wrapper:
- `list_recent(limit)` — fetch recent meetings
- `get_transcript(id)` — fetch transcript for a meeting
- CLI interface: `ruby lib/firefly_fetch.rb [date] [limit]`
- Outputs JSON to stdout (skill captures it) or saves to `data/`

## Files to Modify

### 7. `meeting_review_daily.rb` — Keep as Legacy/Automation

Keep for backward compatibility (cron/launchd), but simplify to call the modular pieces. Or mark as deprecated in favor of skill-based workflow.

## Directory Structure (After)

```
.claude/
  commands/
    firefly.md              # /firefly slash command
    review.md               # /review slash command
analysis.md                 # Default analysis style
quick-analysis.md           # Fast morning scan variant
deepdive-analysis.md        # Comprehensive deep dive variant
lib/
  firefly_fetch.rb          # Standalone Fireflies API client
data/                       # Structured meeting data (JSON) - gitignored
reports/                    # Generated reports - gitignored
meeting_review_daily.rb     # Legacy script (kept for automation)
```

## Implementation Steps

1. Create `analysis.md` — extract rubrics, keywords, and preferences from the Ruby script, add personal preferences and report format sections
2. Create `quick-analysis.md` — lightweight variant with fewer criteria, binary scoring, compact format
3. Create `deepdive-analysis.md` — comprehensive variant with full rubrics, pattern analysis, coaching
4. Create `lib/firefly_fetch.rb` — extract `FirefliesAPI` class and add CLI interface
5. Create `data/` directory with `.gitkeep`
6. Create `.claude/commands/firefly.md` — skill prompt for fetching+clustering
7. Create `.claude/commands/review.md` — skill prompt for analysis+report generation (with analysis style arg)
8. Update `.gitignore` to include `data/*.json`

## Verification

- Run `/firefly` in Claude Code → should fetch meetings, cluster them, save to `data/`
- Run `/review` → uses default `analysis.md`, generates full graded report
- Run `/review quick-analysis.md` → generates compact morning scan
- Run `/review deepdive-analysis.md` → generates comprehensive deep dive with patterns
- Modify any analysis file's rubric weights → re-run `/review` → report reflects changes
- The user iterates on analysis files in plain English to dial in their style
