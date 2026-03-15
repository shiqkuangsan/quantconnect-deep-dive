# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a documentation-only repository containing a 9-part deep dive series on QuantConnect and its LEAN engine. The target audience is frontend/full-stack developers with basic quantitative finance knowledge.

There is no source code, build system, or test suite — the repo consists entirely of Markdown articles.

## Repository Structure

- `quantconnect-deep-dive/README.md` — Series index with reading order, article summaries, and key concept cross-references
- `quantconnect-deep-dive/01-09*.md` — Nine articles ordered sequentially; later articles reference concepts from earlier ones

## Content Architecture

The series follows a deliberate progression:

1. **Platform context** (01) — What QuantConnect is, positioning, pricing
2. **Engine internals** (02–06) — LEAN architecture, data pipeline, algorithm framework, order execution, backtest/live unification
3. **Ecosystem & operations** (07–08) — Open-source repos, plugin patterns, local deployment
4. **Evaluation** (09) — Competitive comparison and decision framework

## Writing Conventions

- Written in Simplified Chinese; technical terms kept in English (e.g., TimeSlice, FillModel, IBrokerage)
- Each article is self-contained but assumes prior articles have been read
- Articles use tables, code blocks (C#/Python), and architecture diagrams in text form
