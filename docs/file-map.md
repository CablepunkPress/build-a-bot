# Bountiful File Map

Every file across all four repos, annotated.
Last updated: September 2026


## bountiful

bountiful/
├── build.py              # First-run setup: name, venv, pip, infrastructure
├── run.py                # Shim → basic_ui.launch.launch()
├── add_tools.py          # Shim → basic_bot.setup.tools.run()
├── add_secrets.py        # Shim → basic_bot.setup.secrets.run()
├── dashboard.json        # Agent identity: id, name
├── persona.md            # Agent personality, user-authored
├── config.toml           # Agent overrides: provider, port, model
├── pyproject.toml        # Pins basic-bot and basic-ui versions
├── ARCHITECTURE.md       # Ecosystem architecture (stale)
├── README.md             # End-user quickstart
├── LICENSE
├── docs/
├── tools/
│   └── README.md         # Explains box tool structure


## basic-bot
...