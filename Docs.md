
# OPIK Integration and Evaluation Guide for `comp_chem`

This guide details integrating OPIK into the `comp_chem` application (based on `opik-ace`) and setting up online evaluation for **Hallucination**. Other metrics (Answer Relevance, Correctness, Moderation Rule) follow a similar evaluation setup.

## 1. Integration Modifications

### 1.1. `agents/comp-chem-agent/agent.py`

#### Imports
Add to `agent.py`:
```python
import os
import opik
from opik.integrations.adk import OpikTracer
from google.adk.runners import Runner
```

#### OPIK Environment Variables
After `load_tool_configs()`, add:
```python
OPIK_API_KEY = os.getenv("OPIK_API_KEY", "local")
OPIK_WORKSPACE = os.getenv("OPIK_WORKSPACE", "default")
OPIK_PROJECT_NAME = os.getenv("OPIK_PROJECT_NAME", "comp-chem-agent-demo")
OPIK_BASE_URL = os.getenv("OPIK_BASE_URL", "https://opik-frontend-1.own4.aganitha.ai:8443/api")
```

#### OPIK Configuration and Tracer
Add with error handling:
```python
try:
    os.environ["OPIK_URL_OVERRIDE"] = OPIK_BASE_URL
    os.environ["OPIK_WORKSPACE"] = OPIK_WORKSPACE
    os.environ["OPIK_API_KEY"] = OPIK_API_KEY
    os.environ["OPIK_PROJECT_NAME"] = OPIK_PROJECT_NAME
    opik.configure(api_key=OPIK_API_KEY, url=OPIK_BASE_URL, workspace=OPIK_WORKSPACE)
except Exception:
    pass

try:
    opik_tracer = OpikTracer(project_name=OPIK_PROJECT_NAME)
    from opik.api_objects import opik_client
    client = opik_client.get_client_cached()
    projects = client.get_projects()
except Exception:
    pass
```

#### `AfterModelCallback` Modifications
Comment out logging:
```python
class AfterModelCallback(BaseModel):
    def __call__(self, callback_context: CallbackContext, llm_response: LlmResponse) -> LlmResponse:
        if hasattr(llm_response, 'content') and hasattr(llm_response.content, 'parts'):
            for part in llm_response.content.parts:
                if hasattr(part, 'thought'):
                    part.thought = None
                if hasattr(part, 'thought_signature'):
                    part.thought_signature = None
        return llm_response
```

#### OPIK Callbacks
Add conditional callbacks:
```python
opik_callbacks = {}
if opik_tracer:
    def debug_after_agent_callback(callback_context, *args, **kwargs):
        try:
            result = opik_tracer.after_agent_callback(callback_context, *args, **kwargs)
            opik_tracer.flush()
            return result
        except Exception:
            return opik_tracer.after_agent_callback(callback_context, *args, **kwargs)
    
    opik_callbacks = {
        "before_agent_callback": opik_tracer.before_agent_callback,
        "after_agent_callback": debug_after_agent_callback,
        "before_model_callback": opik_tracer.before_model_callback,
        "after_model_callback": opik_tracer.after_model_callback,
        "before_tool_callback": opik_tracer.before_tool_callback,
        "after_tool_callback": opik_tracer.after_tool_callback,
    }
```

#### Agent Instantiation
Update `root_agent`:
```python
root_agent = Agent(
    name="my_custom_agent",
    model="gemini-2.5-flash-preview-04-17",
    description="Agent with dynamically loaded tools.",
    instruction=SYSTEM_PROMPT,
    tools=tools,
    **opik_callbacks
)
```

#### Main Block
Update with OPIK tracing:
```python
if __name__ == "__main__":
    runner = Runner(agent=root_agent)
    if opik_tracer:
        try:
            import time
            opik.trace(
                name="connection-test",
                project_name=OPIK_PROJECT_NAME,
                input={"test": "connectivity", "timestamp": time.time()},
                output={"status": "success", "message": "OPIK connection working"}
            )
            opik_tracer.flush()
        except Exception:
            pass
    
    query = "Write a haiku about AI engineering."
    print(f"Running agent for query: {query}")
    response = runner.run(query)
    print("\nAgent response:")
    print(response)
    
    if opik_tracer:
        try:
            opik_tracer.flush()
        except Exception:
            pass
```

### 1.2. `docker-compose.yml`
Update `comp_chem` service:
- **Container Name**: `opik-test-comp-chem-dr-server`
- **Dockerfile**: `apps/opik-comp-chem/Dockerfile`
- **Environment**:
  ```yaml
  environment:
    - OPIK_API_KEY=${OPIK_API_KEY:-local}
    - OPIK_WORKSPACE=${OPIK_WORKSPACE:-default}
    - OPIK_BASE_URL=${OPIK_BASE_URL:-https://opik-frontend-1.own4.aganitha.ai:8443/api}
    - OPIK_PROJECT_NAME=${OPIK_PROJECT_NAME:-comp-chem-agent-demo}
  ```
Update `redis` and `worker` services for container names and Dockerfile paths.

### 1.3. Install OPIK
Add `opik = "*"` to `pyproject.toml` under `[tool.poetry.dependencies]`. Update `Dockerfile` to install dependencies (`poetry install --no-root --no-dev`).

### 1.4. Build and Run
```bash
docker-compose build
docker-compose up
```

## 2. Workflow Steps

### 2.1. Configure and Chat with Agent
After configuring the agent, use the dev-UI to chat with the agent and test its responses.
eg:- https://opik-test-ace-dr-server-yash.own4.aganitha.ai:8443/dev-ui/?app=ace-agent

### 2.2. Access Opik Dashboard
Go to the Opik dashboard at https://opik-frontend-1.own4.aganitha.ai:8443/.

### 2.3. Locate Your Agent
You will see your agent (e.g., `ace-agent-demo`) listed in the dashboard.
<img width="1674" height="966" alt="Screenshot 2025-07-11 at 3 10 12 PM" src="https://github.com/user-attachments/assets/429ff8c3-d56c-4d17-88bd-ff4818e7efeb" />


### 2.4. Navigate to Evaluation Section
Select the "Traces," "LLM calls," "Threads," "Metrics," or "Online evaluation" tabs to view details, then click on "Online evaluation."
<img width="1674" height="966" alt="Screenshot 2025-07-11 at 2 34 40 PM" src="https://github.com/user-attachments/assets/6c7e49a6-aea1-488b-ad26-77f6e4d647c7" />

### 2.5. Create a New Rule
Click the "Create new rule" button to set up a new evaluation rule.
![Screenshot 2025-07-11 at 2 34 40 PM](https://github.com/user-attachments/assets/c8c4162b-cda7-406c-b485-f7c4ef5b8d86) 
OR 

<img width="1674" height="966" alt="Screenshot 2025-07-11 at 2 56 56 PM" src="https://github.com/user-attachments/assets/92c8e1b3-28fb-4d71-af01-808317926fce" />


## 3. Hallucination Evaluation Setup
Evaluates if the agent’s response contains fabricated information.

- **Name**: "HallucinationEval"
- **Sampling Rate**: 0.5–1 (e.g., 0.5 for 50% of traces)
- **LLM Model**: GPT-4o
- **Prompt**:
  ```
  Check if {{output}} introduces information not present in {{context}}. Score 0 for no hallucinations, 1 for hallucinations.
  ```
- **Variables**:
  - `{{input}}`: User query (e.g., "Calculate molecular weight of H2O")
  - `{{output}}`: Agent response (e.g., "18.015 g/mol")
  - `{{context}}`: Prompt template or reference data (e.g., "Provide accurate chemical calculations")
- **Score Definition**: Number (0 = no hallucination, 1 = hallucination)
<img width="1674" height="966" alt="Screenshot 2025-07-11 at 2 41 50 PM" src="https://github.com/user-attachments/assets/50b058e3-807e-470e-a439-671698dc8cbd" />

## 4. Configuring Hallucination Rule
When creating the "Hallucination" rule, set:
- **Prompt**: Use the provided hallucination prompt.
- **Variables**: Map `{{input}}`, `{{output}}`, and `{{context}}` to trace fields.
- **Score Definition**: Select "Number" for hallucination score (0 or 1).
<img width="401" height="966" alt="Screenshot 2025-07-11 at 3 15 34 PM" src="https://github.com/user-attachments/assets/9f63efac-5a15-4e7a-a273-c3c448daf3b6" />

## 5. Viewing Evaluation Results
Access evaluation results under the "Online evaluation" tab, where you can see metrics like Answer Relevance (avg. 0.783), Correctness (avg. 6.25), Hallucination (avg. 0.6), and Moderation (avg. 0) for each trace.
<img width="1674" height="966" alt="Screenshot 2025-07-11 at 3 12 03 PM" src="https://github.com/user-attachments/assets/427e3649-8a53-4680-8a5b-eeca79766566" />



## 6. Notes
- Other metrics (Answer Relevance, Correctness, Moderation Rule) use similar setup with adjusted prompts and scoring.
- Map variables in Opik’s UI if needed.
- Adjust sampling rate based on resources.
- Ensure OPIK instance is accessible from Docker containers.
