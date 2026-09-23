# Hybrid Deterministic-LLM Computation

**A Claude skill that makes AI-assisted maths trustworthy.** Simple calculations go to an exact math engine instead of the language model. Complex tasks still use the model's reasoning. For tasks in between, the model plans the steps and the engine checks every number. Each answer ends with a label that says how its numbers were produced.

```
You:     Verify: 3 items at £19.99 plus 20% VAT — is the total £71.97?
Claude:  Subtotal £59.97 ✔ · VAT £11.99 ✔ · Total £71.96 (not £71.97)
         ⚠ Corrected by deterministic core — 59.97 × 1.2 = 71.964 → £71.96
```

---

## Why

Language models are good at understanding problems but can get arithmetic wrong without warning. A calculator is always right about arithmetic but understands nothing. This skill uses each for what it does best:

| Task type | Who does the work | Label on the answer |
|---|---|---|
| **A. Simple math** (arithmetic, %, VAT, interest, equations, dates) | Deterministic core only | `✔ Computed deterministically` |
| **B. Complex / judgement** (strategy, interpretation, writing) | LLM, as usual | `ⓘ LLM reasoning — not machine-verified` |
| **C. Hybrid** (LLM chooses the method, numbers can be checked) | LLM plans → core checks every step | `✔ Verified by deterministic core — N/N steps checked` or `⚠ Corrected by deterministic core` |

**What "verified" means:** the core checks that the **calculations** are correct. It does not check that the model understood the question or chose the right method, so in hybrid answers the model states its assumptions. The skill says **"proved"** only when the check really is a proof, for example an equation solution confirmed by substituting it back.

## How it works

```
             your request
                  │
      ┌───────────┼─────────────┐
      ▼           ▼             ▼
  A. simple    C. hybrid     B. complex
      │           │             │
      │     LLM writes plan     │
      │     as numeric steps    │
      ▼           ▼             ▼
  ┌───────────────────┐    LLM reasoning
  │ deterministic core│         │
  │ (Python + SymPy)  │         │
  │ exact fractions,  │         │
  │ solve + substitute│         │
  └─────────┬─────────┘         │
            ▼                   ▼
     answer + ✔ / ⚠ label   answer + ⓘ label
```

The core (`scripts/core.py`) is a small command-line tool that prints JSON:

| Command | What it does | Example |
|---|---|---|
| `calc` | exact value of an expression | `calc "0.1+0.2"` → `3/10` (not 0.30000000000000004) |
| `check` | is a claimed value correct? | `check "10/3" "3.4"` → `verified: false` |
| `solve` | solve an equation, then prove each root by substitution | `solve "x^2-5x+6=0"` → 2, 3 ✔ |
| `steps` | check a whole list of calculation steps | see [`examples/invoice-steps.json`](examples/invoice-steps.json) |
| `date` | date arithmetic and day counts | `date 2026-01-01 2026-12-25` → 358 days |

Input syntax: `+ - * / ^ % ( )`, `15%`, `2x`, `1,250`, `sqrt`, `log`, `sin`, `round(x, 2)`, `pi`, `e` and more.

**Safety:** input is parsed with a whitelisted syntax tree, never `eval`, so passing it arbitrary text can't run code. Huge exponents are rejected.

## Installation

The skill is the folder [`hybrid-deterministic-llm-computation/`](hybrid-deterministic-llm-computation/). It requires Python 3.9+ and SymPy (`pip install sympy`).

### Claude Code

Copy the folder into your project, or into your personal skills folder:

```bash
# per project
cp -r hybrid-deterministic-llm-computation  your-project/.claude/skills/
# or for all projects
cp -r hybrid-deterministic-llm-computation  ~/.claude/skills/
```

Then call it with `/hybrid-deterministic-llm-computation`.

### Claude app (claude.ai / desktop)

Zip the `hybrid-deterministic-llm-computation` folder and upload it in **Settings → Capabilities → Skills**. The engine is also included in `SKILL.md` itself, so the skill still works if only that file is present.

### On request or automatic?

By default the skill runs **only when you ask**: "verify this", "compute deterministically", "use the math core", or the slash command. To run it automatically for every maths task, delete `disable-model-invocation: true` from the top of `SKILL.md` and rewrite the `description` line to say when it should trigger.

## Using the core without Claude

```bash
pip install -r requirements.txt
python hybrid-deterministic-llm-computation/scripts/core.py calc "2,340 * 17.5%"
# {"ok": true, "mode": "deterministic", "exact": "819/2", "decimal": "409.500000000000", ...}

python hybrid-deterministic-llm-computation/scripts/core.py steps examples/invoice-steps.json
```

## Tests

```bash
pip install -r requirements.txt pytest
pytest
```

## Repository layout

```
├── hybrid-deterministic-llm-computation/   ← the skill (install this folder)
│   ├── SKILL.md                            instructions for Claude + embedded core
│   └── scripts/core.py                     deterministic math engine
├── examples/invoice-steps.json             hybrid-check example (catches a wrong total)
├── tests/test_core.py                      10 tests: exactness, rounding, safety, solving
├── requirements.txt
└── LICENSE
```

## Limitations

- Checks **arithmetic and algebra**, not the choice of method or the input data.
- Needs an environment where Claude can run code. If it can't, every number is labelled `ⓘ not machine-verified`.
- No unit conversion or statistics library yet. Contributions welcome.

## License

MIT. See [LICENSE](LICENSE).
