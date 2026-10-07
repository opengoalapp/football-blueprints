# Possession flow

A wind-map style animation of how a team moves the ball. Every pass and carry a team made over a
season becomes a flow field: routes they use a lot turn into thick, bright currents, rarely used
areas stay calm, and particles move faster where the ball moved faster. Two pitches side by side
let you compare any two teams.

## The prompt

[`idea.md`](idea.md). Paste it into your coding agent as-is, or adapt it.

## Building it

- Start your agent in an empty folder; it builds there.
- You need [uv](https://docs.astral.sh/uv/) (or any Python 3) and a browser.
- The first run downloads about **410 MB** of StatsBomb open data (FA WSL 2023/24), cached locally.
- Using Claude Sonnet, a one-shot build takes **11–14 minutes** and costs **$1.10–$1.59** at
  pay-as-you-go API prices, generating about **55–76k tokens**.

## Example output

Chelsea (left) against Arsenal (right), Women's Super League 2023/24. Each build from the same prompt comes out a little
different; more examples are in [`gallery/`](gallery/).

![Example output](gallery/run-1.gif)

## Data

Event data from [StatsBomb open data](https://github.com/statsbomb/open-data), used under their
open data terms. Please credit StatsBomb if you publish anything built from it.
