# Copilot Instructions for Kubeflow SDK

**IMPORTANT**: Always refer to the `AGENTS.md` file in the root directory for comprehensive project guidelines, coding standards, and architectural decisions.

**Trust these instructions.** Only search if information is incomplete or incorrect.

## Repository Summary

The **Kubeflow SDK** is a unified Python SDK for running AI workloads at scale on Kubernetes. It provides clients for Kubeflow Trainer (distributed training) and Kubeflow Optimizer (hyperparameter tuning), with local development backends (Container, LocalProcess) and Kubernetes backend support.

- **Project type**: Python library/SDK
- **Python version**: ≥3.9 (CI tests 3.9 and 3.11)
- **Package manager**: `uv` (creates `.venv` automatically)
- **Build system**: Hatchling
- **Linting/Formatting**: `ruff` (with isort integrated)
- **Testing**: `pytest` with `coverage`
- **Pre-commit hooks**: Enforced in CI

---

## Quick Command Reference

### Setup (ALWAYS run first)
```bash
make install-dev    # Installs uv, creates .venv, syncs dependencies
```

### Verify (CI parity - run before commits)
```bash
make verify         # Checks lock file, runs ruff lint + format checks
```

### Testing
```bash
make test-python              # All unit tests + HTML coverage (~2-3 min)
make test-python report=xml   # XML coverage output for CI
uv run pytest -q kubeflow/trainer/backends/kubernetes/backend_test.py  # Single file
uv run pytest -q <path>::<test_name>  # Single test
```

### Lint/Format (auto-fix)
```bash
uv run ruff check --fix .     # Fix lint issues
uv run ruff format kubeflow   # Format code
```

### Pre-commit
```bash
uv run pre-commit install              # Install hooks (once)
uv run pre-commit run --all-files      # Run all hooks manually
```

---

## Project Layout

```
kubeflow/
├── __init__.py          # Version (__version__)
├── common/              # SHARED utilities, constants (used by Trainer AND Optimizer)
├── trainer/             # Trainer component
│   ├── api/             # TrainerClient interface
│   ├── backends/        
│   │   ├── kubernetes/  # K8s backend + tests (*_test.py)
│   │   ├── container/   # Docker/Podman backend + tests
│   │   └── localprocess/# Local subprocess backend + tests
│   ├── constants/       # Trainer-specific constants (DATASET_PATH, MODEL_PATH)
│   ├── options/         # K8s options (Labels, Annotations) + tests
│   ├── types/           # Pydantic v2 data models + tests
│   └── test/common.py   # Test fixtures (TestCase, SUCCESS, FAILED)
└── optimizer/           # Optimizer/Katib component
    ├── api/             # OptimizerClient interface
    ├── backends/kubernetes/
    ├── constants/       # Optimizer-specific constants
    └── types/

scripts/                 # gen-changelog.py
docs/proposals/          # KEP proposals
.github/workflows/       # CI pipelines
```

### Key Files
| File | Purpose |
|------|---------|
| `pyproject.toml` | Dependencies, ruff config, build settings |
| `Makefile` | Development commands (install-dev, verify, test-python) |
| `.pre-commit-config.yaml` | Pre-commit hook config |
| `kubeflow/__init__.py` | Package version |
| `kubeflow/common/` | Shared logic for both Trainer and Optimizer |
| `AGENTS.md` | Detailed agent guidelines and coding standards |

---

## CI Pipeline Details

### On Pull Request
1. **pre-commit job**: Runs all pre-commit hooks
2. **test job** (Python 3.9, 3.11):
   - `make verify` - Lock check + ruff lint + ruff format
   - `make test-python report=xml` - Unit tests with coverage
3. **PR title check**: Must follow Conventional Commits

### Conventional Commit Format
```
<type>(<scope>): <description>
```
- **Types**: `chore`, `fix`, `feat`, `revert`
- **Scopes**: `ci`, `docs`, `deps`, `examples`, `optimizer`, `scripts`, `test`, `trainer`
- Scope is optional but recommended

---

## Coding Standards (Critical)

### Type Hints Required
All functions MUST have complete type annotations:
```python
def submit_job(name: str, config: dict, *, priority: str = "normal") -> str:
```

### Docstrings (Google-style)
```python
def my_function(param: str) -> Result:
    """Short description.

    Args:
        param: Description of param.

    Returns:
        Description of return value.

    Raises:
        ValueError: When param is invalid.
    """
```

### Style Rules
- Line length: 100 characters
- Python target: 3.9
- Quotes: Double quotes
- Imports: Absolute imports, sorted by ruff/isort
- First-party imports: `kubeflow`
- Naming: `snake_case` functions/vars, `PascalCase` classes, `UPPER_SNAKE_CASE` constants

### Logging Pattern (REQUIRED)
Use standard Python logging, **NOT print statements**:

**Correct:**
```python
import logging

logger = logging.getLogger(__name__)

def submit_job(job_name: str) -> str:
    logger.info("Job %s created", job_name)
    logger.debug("Job config: %s", config)
```

 **Incorrect:**
```python
print(f"Job {job_name} created")  
logging.info("Job created")        
```

### Data Modeling with Dataclasses
Use `@dataclass` for data holders and internal types:

**Correct:**
```python
from dataclasses import dataclass

@dataclass
class TrainJob:
    name: str
    status: str
    created_at: str
```

**Incorrect:**
```python
class TrainJob:  # Missing @dataclass
    def __init__(self, name, status, created_at):
        self.name = name
        ...

# Or using raw dictionaries
job = {"name": "job-1", "status": "running"}
```

### Constants Management
**Define constants in `constants.py` modules, NOT at the top of logic files:**

**Correct:**
```python
# kubeflow/trainer/constants/constants.py
DATASET_PATH = "/mnt/datasets"
MODEL_PATH = "/mnt/models"

# In your logic file
from kubeflow.trainer.constants import DATASET_PATH
```

**Incorrect:**
```python
# At top of backend.py
DATASET_PATH = "/mnt/datasets" 
```

### Test Organization
- Tests are co-located: `*_test.py` next to source files
- Use `TestCase` dataclass from `kubeflow/trainer/test/common.py`:
```python
from kubeflow.trainer.test.common import TestCase, SUCCESS, FAILED
```
- Prefer `pytest.mark.parametrize` with `TestCase` for multiple scenarios

---

## Common Errors and Fixes

| Issue | Fix |
|-------|-----|
| `uv` not found | Run `make uv` or `make install-dev` |
| Ruff not installed | Run `make install-dev` |
| Virtualenv issues | Delete `.venv/`, run `make install-dev` |
| Tests pass locally, fail CI | Run `make verify` to match CI lint/format |
| Lock file out of sync | Run `uv lock` |

---

## Before Submitting Changes

1. **Run `make verify`** - Ensures lint/format pass
2. **Run `make test-python`** - Ensures tests pass
3. **Check public API stability** - Don't break existing signatures in `__init__.py`
4. **Write tests** - Every new feature/bugfix needs unit tests
5. **Use conventional commit** - PR title format matters

---

## Do NOT Modify (Without Explicit Request)
- CI/CD workflows (`.github/workflows/`)
- Release automation (`scripts/gen-changelog.py`, `RELEASE.md`)
- `pyproject.toml` dependency versions (unless adding new deps)
- `kubeflow/__init__.py` version (handled by release process)
