# football-blueprints

A catalogue of football analytics and visualisation ideas, written as prompts.

Each blueprint is an **idea file**: a plain-language `idea.md` describing what to build, why, and
the few things that make it work. Instead of sharing code as a gist, you share the idea. Paste it
into an AI coding agent and it builds a working version from scratch. Take the idea, change what
you like (another league, another metric, another look) and build your own.

Every blueprint comes with a short README and a gallery of real builds from the same prompt, so
you can see the range of results before you start.

## Catalogue

| Blueprint | What it builds | Data |
|---|---|---|
| [possession-flow](possession-flow/) | Wind-map style animation of each team's possession routes, two teams side by side | StatsBomb open data, FA WSL 2023/24 |

## How to use a blueprint

1. **Install an AI coding agent** that can write files and run commands in a folder on your
   machine. The blueprints here are written to work one-shot with Claude Sonnet.
2. **Install [uv](https://docs.astral.sh/uv/).** It runs Python and installs it for you if you don't have it:
   - macOS / Linux: `curl -LsSf https://astral.sh/uv/install.sh | sh`
   - Windows (PowerShell): `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
3. **Make an empty folder** where you want the project to live, and start your agent *in that
   folder*. Agents build in the directory they're started from, so don't launch one in your home
   directory or inside another project.
4. **Paste the blueprint's `idea.md`** as your first message, adapting anything you want first.
5. **Check the blueprint's README** for data download sizes, typical build time and cost.

Builds vary from run to run. That's the point: the idea is fixed, the implementation is yours.
