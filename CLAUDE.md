# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Noira is a structured educational knowledge base for Swedish grades 4-6 (ages 10-12), mapped to the national curriculum LGR22. The project contains JSON-formatted "knowledge blocks" (kunskapsblock) covering SO subjects (Social Sciences): Historia, Geografi, Religionskunskap, and Samhällskunskap.

There is no build system, test suite, or runtime — this is a static content repository of JSON + Markdown files.

## Repository Structure

- `noira-kunskapsbas/so/` — Knowledge blocks organized by subject (historia/, religion/, geografi/, samhallskunskap/)
- `noira-kunskapsbas/plan/` — Master curriculum plan mapping blocks to LGR22
- `noira-kunskapsbas/kvalitet/` — Quality assessment reports (granskning) for each reviewed block

## Knowledge Block Schema

Each JSON block (e.g. `HI-46-B1.json`) follows a consistent structure:

- **metadata** — Block ID, theme, title, version, recommended level, prerequisites
- **nyckelbegrepp** — 8 key concepts, each with 3-tier definitions (nivå 1/2/3) + context
- **kronologi** — Timeline events with significance levels
- **karnfakta** — 5 core fact sections, each with 3 complexity levels
- **samband_och_resonemang** — Cause-effect (5), comparisons (3), continuity/change (3)
- **historiska_kallor** — 5 source types with critical thinking questions
- **historiebruk** — Examples of how history is used in society
- **tvaramnes_kopplingar** — Cross-subject connections
- **fragor_for_resonemang** — 3x3 question matrix (3 cognitive levels x 3 grade levels)

## Three-Level Progression System

All content uses a consistent three-level model:
- **Nivå 1** (åk 4): Foundational knowledge, simple reasoning
- **Nivå 2** (åk 5): Deeper understanding, developed reasoning with multiple perspectives
- **Nivå 3** (åk 6): Synthesis, critical analysis, meta-cognitive reflection

Each level maps to Swedish grading criteria: E (elementary), C (competent), A (advanced).

## Block ID Naming Convention

Format: `{SUBJECT}-{GRADES}-{THEME}{NUMBER}`
- `HI-46-B1` = Historia, grades 4-6, Theme B, Block 1
- `RE-46-G5` = Religionskunskap, grades 4-6, Theme G, Block 5
- `GE-46-D1` = Geografi, grades 4-6, Theme D, Block 1

## Quality Assurance

Each block undergoes review documented in `kvalitet/granskning-{BLOCK_ID}.md` with an 18-point checklist covering: JSON validity, field completeness, concept count (8), chronology, core facts (5 sections), level independence, samband completeness, question matrix (3x3), source critique, historiebruk, cross-subject connections, factual accuracy, and no plagiarism.

## Content Language

All knowledge base content is written in Swedish. File names, field names in JSON, and documentation are in Swedish.
