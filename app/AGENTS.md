# AGENTS.md - Streamlit Application

Essential information for AI agents working in the `app/` directory.

## Overview

Streamlit-based risk assessment wizard for the Agentic Risk & Capability Framework. Four-page flow: application description → capability identification → risk assessment → controls recommendation.

## Structure

```
app/
├── app.py              # Main entry (853 lines, page routing via session state)
├── models/
│   └── schemas.py      # Pydantic models + SessionKeys constants
├── utils/
│   ├── data_loader.py  # YAML loading with @st.cache_data
│   ├── llm_utils.py    # LiteLLM integration (4 analysis functions)
│   ├── session_utils.py # Session state helpers
│   └── export_utils.py  # DOCX report generation
└── .streamlit/
    └── config.toml     # Theme config (blue primary, sans-serif)
```

## Key Patterns

### SessionKeys (CRITICAL)

**Never** hardcode session state keys. Use centralized constants:

```python
from models.schemas import SessionKeys

st.session_state[SessionKeys.PAGE] = "results"
data = st.session_state[SessionKeys.APPLICATION_INFO]
```

**Available keys**: PAGE, EDIT_MODE, APPLICATION_INFO, CAPABILITY_ANALYSIS, RISK_ASSESSMENTS, HIGH_PRIORITY_RISKS, LIKELIHOOD_THRESHOLD, IMPACT_THRESHOLD, FORM_DATA_CLASSIFICATION, PURPOSE_TEXT, REPO_URL, etc.

### Pydantic Models

Use `pre=True` validators for data normalization:

```python
class ScoreAssessment(BaseModel):
    score: int = Field(ge=1, le=5)
    reasoning: str
    
    @validator('score', pre=True)
    def convert_score(cls, v):
        # Handle string/float inputs from LLM
        return int(v) if isinstance(v, (float, str)) else v
```

### LLM Integration Pattern

```python
from litellm import completion
from pydantic import BaseModel

class CapabilityAnalysis(BaseModel):
    applicable_capabilities: List[str]
    reasoning: str

response = completion(
    model="gpt-4o",
    messages=[...],
    response_format={"type": "json_object"}  # Required for structured output
)
analysis = CapabilityAnalysis(**json.loads(response.choices[0].message.content))
```

### Data Loading

```python
@st.cache_data
def load_data() -> Tuple[Dict, Dict, Dict, Dict, Dict]:
    """Load and cache YAML data. Returns tuple of 5 dicts."""
    # Loads from ../data/ relative to app/
    return capabilities, risks, controls, components, design
```

## Conventions

**Imports** (absolute within package):
```python
# 1. Standard library
import json
from typing import Dict, List

# 2. Third-party
import streamlit as st
from pydantic import BaseModel
from litellm import completion

# 3. Local (absolute)
from models.schemas import SessionKeys, RiskAssessment
from utils.data_loader import load_data
from utils.llm_utils import analyze_capabilities
```

**Error Handling**:
```python
try:
    result = some_operation()
except FileNotFoundError:
    st.error(f"File not found: {path}")
    return {}  # Graceful default
except yaml.YAMLError as e:
    st.error(f"YAML error: {e}")
    return {}
```

## Commands

```bash
# Development
cd app && pip install -r requirements.txt && streamlit run app.py

# Deployment (Airbase)
airbase link          # Initialize
cd .. && airbase container deploy  # From parent dir with .env

# Requirements (app/ vs root)
app/requirements.txt  # Streamlit, LiteLLM, Pydantic, python-docx
../requirements.txt   # MkDocs, PyYAML (docs)
```

## Dependencies

- **streamlit**: UI framework
- **litellm**: Multi-provider LLM API
- **pydantic>=2.0**: Data validation
- **python-docx**: Report generation
- **PyYAML**: Data loading
- **python-dotenv**: Environment loading

## Environment

Required in parent directory `.env`:
```bash
OPENAI_API_KEY=sk-...
# Optional: ANTHROPIC_API_KEY, etc.
```

## Common Tasks

**Adding a new page**: 
1. Add routing logic in `app.py` (display_* functions)
2. Update SessionKeys if new state needed
3. Update `llm_utils.py` if LLM analysis required

**Modifying models**:
1. Edit `models/schemas.py`
2. Add validator with `pre=True` for data normalization
3. Ensure None handling with defaults

**Adding LLM analysis**:
1. Define Pydantic response model in `models/schemas.py`
2. Add function in `llm_utils.py` using `completion()` with `response_format`
3. Handle JSON parsing errors gracefully
