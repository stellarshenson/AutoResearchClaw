---
name: researchclaw
description: Run ResearchClaw's 23-stage autonomous research pipeline - install package and dependencies, configure LLM provider, execute research from topic to paper
---

# ResearchClaw - Autonomous Research Pipeline Skill

## Trigger Conditions

Activate this skill when the user:
- Asks to "research [topic]", "write a paper about [topic]", or "investigate [topic]"
- Wants to run an autonomous research pipeline
- Asks to generate a research paper from scratch
- Mentions "ResearchClaw" by name
- Asks to install or set up ResearchClaw

## Bundled Resources

This skill directory contains a pre-built wheel for offline installation:
- `researchclaw-0.5.0-py3-none-any.whl` - ResearchClaw v0.5.0 (Python 3.11+, pure Python, ~400KB)

## Instructions

### Installation (run once)

Check if `researchclaw` CLI is available. If not, install from the bundled wheel:

```bash
# Install ResearchClaw from the bundled wheel in this skill directory
pip install /home/lab/workspace/learning/external-projects/aiming-lab-AutoResearchClaw/.claude/skills/researchclaw/researchclaw-0.5.0-py3-none-any.whl

# Install acpx for ACP mode (Agent Client Protocol bridge)
npm install -g acpx
```

Alternatively, install in dev mode from source (editable, tracks code changes):
```bash
pip install -e /home/lab/workspace/learning/external-projects/aiming-lab-AutoResearchClaw
```

Verify installation:
```bash
researchclaw --help && acpx --version
```

### Configuration

1. Check for existing config in the research working directory:
   ```bash
   ls config.yaml 2>/dev/null || ls config.researchclaw.example.yaml
   ```

2. If no `config.yaml`, create one from the example:
   ```bash
   cp /home/lab/workspace/learning/external-projects/aiming-lab-AutoResearchClaw/config.researchclaw.example.yaml config.yaml
   ```

3. Configure the LLM provider. Two supported modes:

   **OpenAI-compatible mode** (default):
   ```yaml
   llm:
     provider: "openai-compatible"
     base_url: "https://api.openai.com/v1"
     api_key_env: "OPENAI_API_KEY"
     primary_model: "gpt-4o"
     fallback_models: ["gpt-4o-mini"]
   ```

   **ACP mode** (persistent agent session across all 23 stages):
   ```yaml
   llm:
     provider: "acp"
     acp:
       agent: "claude"          # CLI binary name, must be on PATH
       cwd: "."                 # working directory for the agent
       session_name: "researchclaw"
       timeout_sec: 600         # per-prompt timeout
   ```
   ACP mode requires `acpx` installed and the agent CLI on PATH. The agent manages its own model selection and API keys.

### Running the Pipeline

Research output goes to the `tmp/` subfolder (gitignored) by default.

**CLI (recommended)**:
```bash
researchclaw run \
  --config config.yaml \
  --topic "Your research topic here" \
  --output tmp/artifacts \
  --auto-approve
```

Options:
- `--topic` / `-t`: Override the research topic from config
- `--config` / `-c`: Config file path (default: `config.yaml`)
- `--output` / `-o`: Output directory (default: `artifacts/rc-YYYYMMDD-HHMMSS-HASH/`)
- `--from-stage`: Resume from a specific stage (e.g., `PAPER_OUTLINE`)
- `--auto-approve`: Auto-approve gate stages (5, 9, 20) without human input
- `--resume`: Resume from last checkpoint
- `--skip-preflight`: Skip LLM connectivity check
- `--skip-noncritical-stage`: Skip noncritical stages on failure

**Python API**:
```python
from researchclaw.pipeline.runner import execute_pipeline
from researchclaw.config import RCConfig
from researchclaw.adapters import AdapterBundle
from pathlib import Path

config = RCConfig.load("config.yaml", check_paths=False)
results = execute_pipeline(
    run_dir=Path("tmp/artifacts/my-run"),
    run_id="research-001",
    config=config,
    adapters=AdapterBundle(),
    auto_approve_gates=True,
)

for r in results:
    print(f"Stage {r.stage.name}: {r.status.value}")
```

### Output Structure

```
tmp/artifacts/<run-id>/
  stage-01/ through stage-23/    # Per-stage outputs
  checkpoint.json                # Resume point
  pipeline_summary.json          # Final metrics
  deliverables/                  # Compile-ready LaTeX, BibTeX, charts
```

### Experiment Modes

| Mode | Description | Config |
|------|-------------|--------|
| `simulated` | LLM generates synthetic results (no code execution) | `experiment.mode: simulated` |
| `sandbox` | Execute generated code locally via subprocess | `experiment.mode: sandbox` |
| `docker` | Execute in GPU-enabled Docker container | `experiment.mode: docker` |
| `ssh_remote` | Execute on remote GPU server via SSH | `experiment.mode: ssh_remote` |

### Environment Health Check

```bash
researchclaw doctor --config config.yaml
```

### Troubleshooting

- **`researchclaw` not found**: Run `pip install -e /home/lab/workspace/learning/external-projects/aiming-lab-AutoResearchClaw`
- **`acpx` not found**: Run `npm install -g acpx`
- **Config validation error**: Run `researchclaw validate --config config.yaml`
- **LLM connection failure**: Check `llm.base_url` and API key, or ACP agent availability
- **Sandbox execution failure**: Verify `experiment.sandbox.python_path` exists
- **Gate rejection**: Use `--auto-approve` or manually approve at stages 5, 9, 20

## Tools Required

- Bash (for CLI execution and installation)
- File read/write (for config and artifacts)
