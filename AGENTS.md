# AGENTS.md - Agentic Risk & Capability Framework

Essential information for AI agents working in this repository.

## Build Commands

```bash
# Documentation (MkDocs)
mkdocs serve              # Local development with hot reload
mkdocs build              # Build static site to site/

# Risk Register Data Generation (run from repo root)
python scripts/build_risk_register.py

# Streamlit Application
cd app && pip install -r requirements.txt && streamlit run app.py

# Virtual Environment (if needed)
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
.venv\Scripts\activate      # Windows
```

**No test framework configured.** Test manually via Streamlit interface.

## Project Structure

```
.
├── docs/                    # MkDocs documentation source
│   ├── arc_framework/       # Core framework documentation
│   ├── implementation/      # Implementation guides
│   └── assets/             # Generated risk_register_data.json
├── app/                     # Streamlit risk assessment application
│   ├── app.py              # Main entry point
│   ├── models/             # Pydantic schemas
│   └── utils/              # Data loading, LLM utilities
├── scripts/                # Build automation
│   └── build_risk_register.py
├── data/                   # Source YAML definitions (baseline)
├── arc-risk-register/      # Updated YAML definitions (primary source)
└── mkdocs.yml             # MkDocs configuration
```

## Code Style Guidelines

### Python

**Import Order (enforced pattern)**:
```python
# 1. Standard library
from typing import Dict, List, Any
import os

# 2. Third-party
import yaml
import streamlit as st
from pydantic import BaseModel, Field, validator

# 3. Local (absolute within package)
from models.schemas import SessionKeys, RiskAssessment
from utils.data_loader import load_data
```

**Type Hints (required)**:
```python
def get_controls_for_risk(risk_id: str) -> Dict[str, Any]:
    """Return controls mapped to a risk ID."""
```

**Naming**:
- Classes: `PascalCase` (e.g., `RiskAssessment`, `SessionKeys`)
- Functions/variables: `snake_case`
- Constants: `UPPER_SNAKE_CASE` (class attributes for session keys)

**Documentation**:
- Module-level docstrings explain purpose
- Function docstrings with Args/Returns sections
- Field descriptions in Pydantic models

**Error Handling**:
```python
try:
    with open(file_path, 'r') as f:
        return yaml.safe_load(f)
except FileNotFoundError:
    st.error(f"File not found: {file_path}")
    return {}  # Graceful default
except yaml.YAMLError as e:
    st.error(f"YAML parsing error: {e}")
    return {}
```

### Key Patterns

**Pydantic Models**:
```python
class ScoreAssessment(BaseModel):
    score: int = Field(ge=1, le=5, description="Score 1-5")
    reasoning: str = Field(description="Explanation")
    
    @validator('score', pre=True)
    def convert_score(cls, v):
        # Normalize data before validation
        return int(v) if isinstance(v, float) else v
```

**Streamlit Session Keys (critical for state management)**:
```python
# Use centralized SessionKeys class - NEVER hardcode strings
st.session_state[SessionKeys.APPLICATION_INFO] = data
st.session_state[SessionKeys.PAGE] = "results"

# Available keys:
# PAGE, EDIT_MODE, APPLICATION_INFO, CAPABILITY_ANALYSIS
# RISK_ASSESSMENTS, HIGH_PRIORITY_RISKS, LIKELIHOOD_THRESHOLD
```

**Data Loading with Caching**:
```python
@st.cache_data
def load_data() -> Tuple[Dict, Dict, Dict, Dict, Dict]:
    """Load and cache YAML data files."""
    # Implementation returns tuple of dicts
```

### YAML Data Files

Located in `/arc-risk-register/`:
- `capabilities.yaml` - Capability taxonomy
- `risks.yaml` - Risk definitions
- `controls.yaml` - Control definitions
- `components.yaml` - System components
- `design.yaml` - Design elements

**Workflow**: Edit YAML → Run `build_risk_register.py` → Refresh docs

## Dependencies

**Documentation**: MkDocs, Material theme, PyYAML
**App**: Streamlit, LiteLLM, Pydantic v2, python-docx
**No linting tools configured** — follow observed patterns in existing code.

**Pydantic validator pattern** — Always use `pre=True` for data normalization:
```python
@validator('field', pre=True)
def normalize(cls, v):
    if isinstance(v, str):
        return int(float(v))  # Handle string/float inputs from LLM
    return v
```

**Streamlit caching** — Data loading functions must use decorator:
```python
@st.cache_data
def load_data() -> Tuple[Dict, ...]:
    """Load YAML data with caching."""
```

## Environment Variables

```bash
# Required for Streamlit app
OPENAI_API_KEY=sk-...
```

Use `python-dotenv` to load from `.env` file.

## Common Tasks

**Adding a new risk**:
1. Edit `arc-risk-register/risks.yaml`
2. Link to capabilities/components/design elements
3. Reference relevant controls
4. Run `python scripts/build_risk_register.py`
5. Verify at `/arc_framework/risk-register.html`

**Modifying the Streamlit app**:
- Entry point: `app/app.py`
- Data models: `app/models/schemas.py`
- Utilities: `app/utils/`
- Always update SessionKeys for new state

## References

- MkDocs Material: https://squidfunk.github.io/mkdocs-material/
- Streamlit Docs: https://docs.streamlit.io/
- Pydantic v2: https://docs.pydantic.dev/latest/
