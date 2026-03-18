# snirs-skills

A collection of custom AI agent skills for code assistants.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **view-md** | `/view-md <file.md>` | Preview any markdown file as a styled HTML page with dark/light toggle, syntax highlighting, and copy-to-clipboard |
| **split** | `/split <video>` | Split a video file into equal parts with zero quality loss (ffmpeg stream copy) |
| **code-reviewer** | `/code-reviewer` | Run an expert code review — categorizes issues as Critical, Warning, or Suggestion |
| **ship** | `/ship` | Full shipping workflow: commit, review, fix, test, QA with Playwright, merge |

## Installation

Clone this repo into your agent's skills directory:

```bash
# Back up existing skills (if any)
mv ~/.claude/skills ~/.claude/skills.bak

# Clone
git clone https://github.com/balgaly/snirs-skills.git ~/.claude/skills

# Or if you already have the directory, init and pull
cd ~/.claude/skills
git remote add origin https://github.com/balgaly/snirs-skills.git
git pull origin main
```

Skills are automatically discovered — no additional configuration needed.

## Usage

In any Claude Code session, use the slash command:

```
/view-md ./README.md
/split ~/videos/meeting.mp4
/code-reviewer
/ship
```

## Structure

```
~/.claude/skills/
  view-md/
    SKILL.md          # Skill definition
    template.html     # HTML template with dark/light mode
  split/
    SKILL.md
  code-reviewer/
    SKILL.md
  ship/
    SKILL.md
```

Each skill is a self-contained folder with a `SKILL.md` that defines the skill's behavior, triggers, and instructions.

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feat/my-new-skill`
3. Add your skill folder with a `SKILL.md`
4. Open a PR against `main`

## License

MIT
