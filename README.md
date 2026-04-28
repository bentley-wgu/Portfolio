# Portfolio Segmentation Calculator

Interactive web calculator for classifying B2B partner accounts into four portfolios (AP, GP, MP, WP) and projecting FY27 targets. Built for WGU's vertical partner teams.

## Live site

The calculator is hosted via GitHub Pages from this repository. Once Pages is enabled, the URL will be available in **Settings > Pages**.

## What's in here

- `index.html` — the calculator. Self-contained, no build step, no external dependencies. Open it directly in a browser or serve it from any static host.
- `Portfolio Classification Logic.md` — plain-language documentation of the classification rules, formulas, and worked examples. Drop-in Excel and Python implementations included.

## Updating the data

The calculator embeds account data inline in `index.html`. To refresh, regenerate `index.html` from the source spreadsheet and replace the file in this repo. Pages redeploys automatically within about a minute of the commit.

## Customization quick reference

The calculator reads four pieces of data per account: Partner Type, FY25 System Starts, FY26 System Starts, and Velocity. Five partner types are supported: Corporate, Healthcare, K12 Partner, Transfer Partner, Military. Each partner type has its own Volume Trigger, Velocity Trigger, and four per-portfolio growth rates that the user can adjust live.

## Brand

Styled to the WGU Design System (WGU Blue #002855, Jost typography substituting for Futura PT, soft blue-tinted shadows). The WGU Corporation Logo is embedded in the header.
