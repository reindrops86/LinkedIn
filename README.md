# CrewAI FMCG Research Script

This project uses a lightweight CrewAI workflow to research recent AI developments relevant to fast-moving consumer goods (FMCG) and turn them into a short executive summary.

## Overview

The script:
- searches the web using Tavily
- identifies the top 3 recent AI advancements relevant to FMCG
- creates a leadership-ready summary in bullet format
- saves the final output to an output file for portfolio or demo use

## Features

- Web research via Tavily API
- Two-agent CrewAI workflow:
  - Research agent
  - Executive writer agent
- Automatic output saving to `output/fmcg_ai_summary.txt`
- Environment-based configuration with `.env`

## Prerequisites

- Python 3.10+
- Tavily API key
- OpenAI API key or Azure OpenAI API key

## Setup

1. Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
