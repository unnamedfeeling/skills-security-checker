# Skills Security Checker 🛡️

A composite GitHub Action and security CLI that audits agent skills and prompts against malicious actions, prompt injections, and unsafe commands using an OpenAI-compatible endpoint.

## Motivation

We are increasingly depending on natural language prompting techniques to write code. This means that increasing amount tools, prompts, agents are added to our codebases. This is an attack vector for malicious actors - planting backdoors, stealing data and secrets and more can be performed through prompts. Thus security of prompts is getting increasingly more important today. The solution here provides a basic guard against such acts of aggression. You can use your own (known good) - LLM models, validation prompts and thresholds. Please read below for more information.

---

## Features

- **OpenAI-Compatible**: Seamlessly works with OpenAI, OpenRouter, Ollama, LiteLLM, vLLM, or any custom inference endpoint.
- **Bring your own rules**: There is a sane defaults, but you own the validation process on your side
- **Native GitHub Annotations & Issues**: Emits inline `::error` annotations on PR diffs, generates Markdown step summaries, and optionally opens tracked GitHub Security Issues for findings with automatic deduplication.
- **Enterprise Findings Export**: Extensible exporter architecture supporting **OASIS SARIF v2.1.0** (GitHub Code Scanning) and **AWS Security Finding Format (ASFF)** (AWS Security Hub).
- **Modern tooling**: Python 3.10+, uv, ruff and agentic development supported

Baseline (included):
- **Prompt Injection Defense**: Detects jailbreak attempts, hidden developer overrides, and behavioral deviations in `SKILL.md` definitions.
- **Dangerous Command Detection**: Catches destructive shell operations (`rm -rf`, pipe-to-bash, unauthorized `sudo`).
- **Data Exfiltration Prevention**: Flags unauthorized access to secrets (`~/.ssh`, `~/.aws`, `.env`) and suspicious outbound network requests.

---

## Quickstart (GitHub Action)

Good pattern - default workflows running over all of your repositories during PR validation or on schedule.

Add this action to your `.github/workflows/security.yaml`:

```yaml
name: Audit Agent Skills

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  audit-skills:
    name: Skill Security Audit
    runs-on: ubuntu-latest
    permissions:
      contents: read
      issues: write # Required if create-issues is enabled
      security-events: write # Required if upload-sarif is enabled
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Run Skills Security Checker
        uses: unnamedfeeling/skills-security-checker@v1 # Or your-username/skills-security-checker@v1
        with:
          target-directories: |
            skills
            prompts
            .copilot
          api-key: ${{ secrets.OPENAI_API_KEY }}
          fail-on-severity: 'HIGH'
          sarif-file: 'results.sarif'
          aws-findings-file: 'aws-findings.json'
          # Optional: Automatically open tracked GitHub Security Issues
          create-issues: 'true'
          github-token: ${{ secrets.GITHUB_TOKEN }}
          issue-min-severity: 'HIGH'

      - name: Upload SARIF report to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'results.sarif'
```

---

## Enterprise Customization & Policy Overrides

Organizations can customize security policies, rule definitions, and severity classifications externally while retaining the built-in defaults for unmentioned fields.

Overrides can be provided via individual action inputs, environment variables, or a unified `.skills-security.json` in the repository root.

Please note: you can use custom configuration file per repository or according to your organization rules in runtime.

### Example: `.skills-security.json`

```json
{
  "prompt_path": "prompts/custom_audit.md",
  "finding_levels": {
    "order": {
      "BLOCKER": 6,
      "CRITICAL": 5,
      "HIGH": 4,
      "MEDIUM": 3,
      "LOW": 2,
      "INFO": 1
    },
    "sarif_levels": {
      "CRITICAL": "error",
      "HIGH": "error",
      "MEDIUM": "warning"
    },
    "aws_levels": {
      "CRITICAL": "CRITICAL",
      "HIGH": "HIGH",
      "MEDIUM": "MEDIUM",
      "LOW": "LOW",
      "INFO": "INFORMATIONAL"
    },
    "annotation_levels": {
      "CRITICAL": "error",
      "HIGH": "error",
      "MEDIUM": "warning",
      "LOW": "warning"
    }
  },
  "aws_types_mapping": {
    "prompt_injection": [
      "Software and Configuration Checks/Vulnerabilities/Prompt Injection",
      "TTPs/Initial Access"
    ]
  },
  "sarif_rules": {
    "prompt_injection": {
      "id": "ACME-SEC-001",
      "name": "AcmePromptSecurityPolicy",
      "shortDescription": { "text": "Violation of Acme AI Agent Prompt Policy" },
      "help": { "text": "Refer to https://security.internal.acme.corp/policies/ai-skills" }
    }
  }
}
```

### Action Configuration Inputs

| Input | Description | Default |
|---|---|---|
| `prompt-path` | Path to custom audit system prompt markdown file | Built-in `system_audit.md` |
| `finding-levels` | JSON string or path to JSON file overriding severity ranking, SARIF levels, AWS levels, and annotation levels | Built-in mappings |
| `aws-types-mapping` | JSON string or path to JSON file overriding AWS Security Hub finding types | Built-in ASFF taxonomy |
| `sarif-rules` | JSON string or path to JSON file overriding SARIF rule descriptors | Built-in OASIS rules |
| `config-file` | Path to unified JSON configuration file | Auto-detects `.skills-security.json` |

---

## Local Development & Testing

### Prerequisites
- Python 3.10+
- [`uv`](https://docs.astral.sh/uv/)

### Installation & Run
```bash
# Clone the repository
git clone https://github.com/unnamedfeeling/skills-security-checker.git
cd skills-security-checker

# Sync dependencies
uv sync

# Copy example environment configuration
cp .env.example .env

# Run security scan
uv run python -m src.app
```

### Verification & Testing
```bash
# Run unit tests
uv run pytest -v

# Run linter & formatter
uv run ruff check .
uv run ruff format --check .

# Run static type checker
uv run mypy src
```

---

## Disclaimer

> [!CAUTION]
> **Provided "AS IS"**: This software is provided on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
>
> While Skills Security Checker uses heuristic static scanning and LLM-assisted semantic analysis to identify common attack vectors and malicious actions, no automated tool can guarantee 100% detection of all vulnerabilities, zero-days, or novel evasion techniques. Always employ defense-in-depth, least-privilege sandboxing, and manual human reviews for high-risk agent operations.

---

## License

This project is licensed under the [Apache License, Version 2.0](LICENSE).

---

## Support & Feedback
Found a false positive or encounter an issue? Please open an issue on [GitHub Issues](https://github.com/unnamedfeeling/skills-security-checker/issues).
