# mql-developer

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for professional MQL4/MQL5 development on MetaTrader 4 and MetaTrader 5 platforms.

Covers the full ecosystem: Expert Advisors, custom indicators, scripts, libraries, UI panels, trading operations, external API communication, backtesting, and code protection.

## What It Provides

- **MQL4 & MQL5 language reference** — data types, functions, predefined variables, preprocessor, error handling
- **OOP & Standard Library (MQL5)** — classes, interfaces, templates, CTrade, CPositionInfo, CCanvas
- **EA architecture patterns** — single-file, modular (Signal + Trade + Risk + Filter), state machine, multi-timeframe, multi-symbol
- **Trading operations** — order/position management with retry logic, risk-based position sizing, drawdown control, trailing stops
- **Indicators & UI** — custom indicators (buffers, draw styles, OnCalculate), graphical objects, CAppDialog panels, scripts
- **External communication** — WebRequest REST API, JSON handling, Node.js server integration, sockets, inter-program communication
- **Backtesting** — Strategy Tester modes, walk-forward analysis, Monte Carlo simulation, optimization
- **Security & licensing** — account-based licensing, server-side validation, anti-decompilation, MQL5 Cloud Protector

## Skill Structure

```
mql-developer/
├── SKILL.md                            # Entry point with navigation table
└── references/
    ├── mql4-reference.md               # MQL4 language reference
    ├── mql5-reference.md               # MQL5 OOP, CTrade, Standard Library
    ├── architecture-patterns.md        # EA architectures and design patterns
    ├── trading-operations.md           # Orders, risk management, trailing stops
    ├── indicators-and-ui.md            # Indicators, panels, scripts, chart objects
    ├── external-communication.md       # WebRequest, JSON, Node.js, sockets
    ├── backtesting.md                  # Strategy Tester, optimization, Monte Carlo
    └── security-licensing.md           # Code protection and licensing
```

Uses progressive disclosure: `SKILL.md` loads first (~140 lines), then only the relevant reference file is loaded based on the task.

## Installation

Copy the skill to your Claude Code skills directory:

```bash
# Option 1: Clone and copy
git clone https://github.com/YOUR_USERNAME/mql-developer.git
cp -r mql-developer ~/.claude/skills/

# Option 2: Direct copy (if already cloned)
cp -r mql-developer ~/.claude/skills/
```

The skill activates automatically when Claude detects MQL-related tasks.

## Usage Examples

```
You: "Create an EA with MA crossover strategy for EURUSD"
You: "Add risk management with 2% per trade and max 10% daily drawdown"
You: "Build a custom RSI indicator with color zones"
You: "Set up WebRequest to send trade notifications to my Node.js server"
You: "Add account-based licensing to my EA"
```

## Design Principles

Built following the [skill-creator](https://github.com/anthropics/courses/tree/master/claude-code/09-skills) best practices:

- **SKILL.md under 500 lines** — concise entry point with navigation
- **Progressive disclosure** — reference files loaded only when needed
- **No duplication** — language references contain API signatures, specialized files contain production patterns
- **Table of contents** — all reference files include TOC for quick navigation
- **Grep patterns** — SKILL.md includes section headers for targeted search in large files

## License

MIT
