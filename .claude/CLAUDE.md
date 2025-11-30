# CLAUDE.md - Project Instructions for AI Assistants

## Project Overview

This is "Zen and the Art of Data Maintenance" (also known as "Data Preparation for Machine Learning: A Data-Centric Approach") - a comprehensive technical book by David Aronchick.

## Repository Structure

```
/
├── Chapter_000000001/
│   └── content.md          # Final, canonical chapter content
├── Chapter_000000002/
│   └── content.md
├── ...
├── Outline.md              # Final, canonical book outline
├── README.md
├── .gitignore
└── .claude/
    └── CLAUDE.md           # This file
```

## Critical File Naming Rules

### What Gets Checked In (Public)

Only these file patterns should exist in the repository:
- `content.md` - Final chapter content
- `Outline.md` - Final book outline  
- `README.md` - Repository documentation
- Standard config files (`.gitignore`, etc.)

### What NEVER Gets Checked In

The following are **working files only** and must be excluded from git:

| Pattern | Purpose | Action |
|---------|---------|--------|
| `*_REVISED*` | In-progress revisions | Delete or rename to `content.md` when done |
| `*_revised*` | In-progress revisions | Delete or rename to `content.md` when done |
| `*_RAW*` | Raw/unedited drafts | Delete or rename when done |
| `*_raw*` | Raw/unedited drafts | Delete or rename when done |
| `*_BACKUP*` | Backup copies | Delete when no longer needed |
| `*_backup*` | Backup copies | Delete when no longer needed |
| `*_OLD*` | Deprecated versions | Delete when no longer needed |
| `*_old*` | Deprecated versions | Delete when no longer needed |
| `*_DRAFT*` | Draft versions | Delete or rename when done |
| `*_draft*` | Draft versions | Delete or rename when done |
| `*_WIP*` | Work in progress | Delete or rename when done |
| `*_wip*` | Work in progress | Delete or rename when done |

### Workflow for Edits

1. **Before making changes**: Create a timestamped backup of the current canonical file
   - Format: `content_BACKUP_YYYYMMDD.md`
   - These backups are gitignored and stay local
2. **Make edits directly to canonical files**: Work on `content.md` directly
3. **If major restructuring needed**: Create `content_REVISED.md`, review, then replace canonical
4. **Commit**: Only canonical files (`content.md`, `Outline.md`) get committed

Example workflow:
```bash
# Before making changes - create local backup
cp Chapter_000000001/content.md Chapter_000000001/content_BACKUP_20251130.md

# Make edits to canonical file
# ... edit content.md ...

# Commit canonical file only (backups are gitignored)
git add Chapter_000000001/content.md
git commit -m "Chapter 1: Updated with voice consistency improvements"
```

### Current Backup Files (Local Only)
These exist locally but are gitignored:
- `Chapter_*/content_BACKUP_*.md` - Pre-edit backups
- `Outline_BACKUP_*.md` - Outline backups

## Voice Model

The book follows the "Aronchick Voice" - see the master voice model at:
`/Users/daaronch/My Drive (aronchick@gmail.com)/Memory/Voice_Model/MASTER_VOICE_MODEL.md`

Key characteristics:
- Confrontational expertise with specific disaster stories
- Real numbers and cost calculations
- Strategic profanity as punctuation (sparingly)
- Self-deprecating humor
- Systems thinking applied to data problems
- War stories over listicles

## Content Guidelines

### Code Examples
Use sparingly. Only include code when it:
- Illuminates concepts words can't convey
- Demonstrates common mistakes
- Provides useful snippets readers will copy
- Illustrates scale or complexity

### Chapter Structure
Each chapter should have:
- Opening disaster story with real numbers
- Practical, actionable content
- "Quick Wins Box" with immediate actions
- Homework exercises
- Closing P.S. with humor

### What to Avoid
- AI-generated smooth transitions ("Let me throw some numbers at you")
- Template-style formatting (What's Happening / Why / When)
- Listicle structures ("Ten Things...")
- Wikipedia-style encyclopedic tone
- Hedging language ("might," "perhaps," "could potentially")

## Current Book Status

Part I (Foundation) - Chapters 1-4:
- Chapter 1: Data-Centric AI Revolution (philosophy)
- Chapter 2: Data Types and Structure Spectrum
- Chapter 3: File Formats: Choosing Your Poison
- Chapter 4: The Hidden Costs of Data (economics)

Total planned chapters: 29

## Questions?

If unclear about voice, structure, or content decisions, reference:
1. The master voice model (Google Drive)
2. Existing approved chapters (especially Chapter 1)
3. The detailed outline in `Outline.md`
