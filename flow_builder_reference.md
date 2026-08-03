# watsonx Orchestrate ADK — Flow Builder Reference

Notes distilled from the official examples:
https://github.com/IBM/ibm-watsonx-orchestrate-adk/tree/main/examples/flow_builder

## What Flow Builder is

A Python DSL in `ibm_watsonx_orchestrate.flow_builder` for building multi-step
orchestration flows out of nodes (tools, scripts, forms, agents) wired together
with edges. A flow is itself packaged/imported as a tool that an agent can call.

Core imports:
```python
from ibm_watsonx_orchestrate.flow_builder.flows import (
    Flow, flow, START, END, Branch, ScriptNode
)
from ibm_watsonx_orchestrate.flow_builder.types import ForeachPolicy
```

## Anatomy of a flow

Define pydantic schemas, then decorate a builder function with `@flow`:

```python
from pydantic import BaseModel, Field

class Name(BaseModel):
    first_name: str
    last_name: str

class Message(BaseModel):
    msg: str

@flow(
    name="hello_message_flow",
    display_name="Hello Message",     # optional
    description="...",                # optional
    input_schema=Name,
    output_schema=Message,
    private_schema=PrivateState,      # optional scratch state
)
def build_hello_message_flow(aflow: Flow) -> Flow:
    ...
    return aflow
```

## Node types

- **Tool node** — `aflow.tool(python_callable)` or `aflow.tool("openApiOperationId")`
  for tools imported from an OpenAPI spec. Optional `output_schema=...`.
- **Script node** — `aflow.script(name=..., display_name=..., script="""...python...""")`.
  Inside the script you read/write `flow.input`, `flow.private`, `flow.output`,
  and `flow.<node_name>.output`. `time` is available (e.g. `time.sleep`).
- **Form node** — `aflow.form(...)` / `subflow.form(...)` for user input (see Forms).

## Edges / sequencing

```python
aflow.edge(START, node_a)
aflow.edge(node_a, node_b)
# chaining:
aflow.edge(START, a).edge(a, b).edge(b, END)
# shortcut:
aflow.sequence(START, node_a, node_b, END)
```

## Branching (if/else) — `aflow.conditions()`

```python
branch: Branch = aflow.conditions()
branch.condition(expression="flow.input.kind.strip().lower() == 'dog'", to_node=dog_node) \
      .condition(expression="flow.input.kind.strip().lower() == 'cat'", to_node=cat_node) \
      .condition(to_node=dog_node, default=True)   # fallback
aflow.edge(START, branch)
aflow.edge(dog_node, END)
aflow.edge(cat_node, END)
```

## Parallel execution

- **Unconditional** — all branches run:
  ```python
  p = aflow.parallel(evaluator=None, name="phase2", display_name="Development")
  s1 = p.script(name="squad1", script="...")
  p.sequence(START, s1, END)   # wire each branch inside the parallel subflow
  aflow.edge(prev_node, p)     # join point continues after all complete
  ```
- **Conditional parallel** — subset of branches run based on conditions:
  ```python
  pc = aflow.parallel_conditions(name="phase1")
  d = pc.script(name="design_work", script="...")
  pc.condition(expression="flow.private.design_needed is True", to_node=d) \
    .condition(default=True, to_node=skip_node)
  pc.sequence(d, END)
  ```

## Loops — `aflow.foreach(...)`

```python
foreach_flow: Flow = aflow.foreach(item_schema=CustomerRecord) \
    .policy(kind=ForeachPolicy.SEQUENTIAL)   # or parallel policy
node = foreach_flow.tool(send_invitation_email)
foreach_flow.sequence(START, node, END)
aflow.edge(get_list_node, foreach_flow)
aflow.edge(foreach_flow, END)
```

## Data mapping

Automatic mapping happens at runtime by default. To map explicitly:

```python
# per-node input mapping
node.map_input(input_variable="first_name", expression="flow.input.first_name")
node.map_input(input_variable="last_name",
               expression="flow.input.last_name",
               default_value="default_last_name")
# reference a prior node's whole output
next_node.map_input(input_variable="name", expression="flow.combine_names.output")
# map flow output
aflow.map_output(output_variable="msg", expression="flow.get_hello_message.output")
```

Expression namespace: `flow.input.*`, `flow.private.*`, `flow.output.*`,
`flow.<node_name>.output`.

## Forms (interactive user input)

```python
form = user_flow.form(name="ApplicationForm", display_name="Application",
                      cancel_button_label="Cancel")
form.text_input_field(name="lastName", label="Last name", required=True,
                      placeholder_text="...", regex="^[a-zA-Z0-9\\s]+$")
form.number_input_field(name="age", label="Age", required=True)
form.boolean_input_field(name="married", label="Married", single_checkbox=True)
form.multi_choice_input_field(name="fruits", label="List of Fruits",
                              show_as_dropdown=True, minItems=1, maxItems=2)
form.file_upload_field(name="credentials", label="Upload credentials",
                       allow_multiple_files=True)
form.user_input_field(name="approvers", label="Select Approvers", multiple_users=True)
# single-choice dropdowns bind options via DataMap/Assignment from flow data
```

## Typical project layout (per example)

```
<example>/
  tools/         # flow + tool python files, requirements.txt, *.openapi.yml
  agents/        # agent yaml/definitions that use the flow
  generated/     # auto-generated artifacts
  main.py        # run flow directly
  import-all.sh  # import tools + agents into orchestrate
  README.md
```

## Running / testing

- Import + chat: run `import-all.sh`, then `orchestrate chat start`.
- Direct: `python3 main.py` (set `PYTHONPATH` appropriately).

## Example index (patterns to copy from)

- `hello_message_flow` — basic sequence, auto data mapping, flow-inside-agent.
- `hello_message_flow_datamap` — explicit `map_input` / `map_output`.
- `hello_message_script_flow` — script nodes.
- `get_pet_facts_if_else` — `conditions()` branching + OpenAPI tools.
- `get_pet_facts_error_branching` — error/failure branching.
- `parallel_flow` — `parallel` + `parallel_conditions`, phased workflow, private state.
- `foreach_email` — `foreach` with `ForeachPolicy`.
- `user_activity_with_forms` / `dynamic_forms` — form field types, user assignment.
- `document_extractor*`, `text_extraction`, `document_classifier` — document AI flows.
- `flow_callback`, `agent_scheduler`, `schedule_helpdesk_alert` — callbacks/scheduling.
- `collaborator_agents`, `triage_workflow_agent_swarm` — multi-agent orchestration.
- `live_agent_transfer_flow`, `masking_test_flow`, `flow_mcp_tester` — misc integrations.
