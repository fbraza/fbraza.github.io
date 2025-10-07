# pydantic-ai Agent Notes

These notes explain how pydantic-ai selects and sequences tools, why the Slack tool was executed in tests, and how to control/test behavior effectively.

## Mental Model

- You register tools with a toolset. The agent presents the tool schemas (name, description, params) to the model.
- The model (LLM or `TestModel`) decides which tool(s) to call based on the user prompt and tool descriptions.
- The runtime validates tool args with `pydantic`, invokes the function, then feeds the result back to the model. The model can then decide next steps.
- Sequencing is model-driven: model → tool(s) → model → … until it produces a final answer or limits are reached.

## Tool Registration (in this repo)

```py
# app/tools/slack.py
slack_toolset = FunctionToolset()

@slack_toolset.tool(name="slack.chat.postMessage")
def post_message(ctx: RunContext[Deps], params: SlackChatPostMessageParams) -> SlackResponse:
    return ctx.deps.client.chat_postMessage(channel=params.channel, text=params.text)
```

- `Deps` provides a `WebClient` instance via dependency injection.
- The tool is registered under the name `slack.chat.postMessage` with typed params.

## Why Slack Was Called Unexpectedly

- The tests used `TestModel` from `pydantic_ai.models.test`.
- `TestModel` defaults to `call_tools='all'`, meaning it attempts to call all tools in the agent on every run, regardless of the prompt.
- When `agent.run_sync("What tools are available?")` ran without providing `deps`, the Slack tool executed and tried to access `ctx.deps.client`; since `deps=None`, it raised `AttributeError: 'NoneType' object has no attribute 'client'`.

## How To Avoid Unwanted Tool Execution

- If you only want to list/inspect available tools, instantiate the model with no tool execution:

```py
from pydantic_ai.models.test import TestModel
from pydantic_ai import Agent
from app.tools import slack

model = TestModel(call_tools=[])  # do not auto-call tools
agent = Agent(model, toolsets=[slack.slack_toolset])
_ = agent.run_sync("What tools are available?")
# Inspect: model.last_model_request_parameters.function_tools
```

- If you want to exercise the tool path in tests, provide dependencies so the tool can run against a fake or real client:

```py
from pydantic_ai import RunContext, RunUsage

fake_client = FakeSlackClient()
ctx = RunContext(deps=slack.Deps(client=fake_client), model=TestModel(), usage=RunUsage())
params = tools.SlackChatPostMessageParams(channel="sandbox", text="hello")
response = slack.post_message(ctx=ctx, params=params)
```

## Testing Strategy

- Unit tests: use a fake Slack client to assert forwarding and return shape. No network calls.
- Integration tests: optional and opt-in. Guard with an env flag and DNS check to skip cleanly offline.

Example unit test idea (simplified):

```py
class FakeSlackClient:
    def __init__(self):
        self.calls = []
    def chat_postMessage(self, channel: str, text: str):
        self.calls.append((channel, text))
        return {"ok": True, "message": {"text": text}, "ts": "123.456"}

fake = FakeSlackClient()
ctx = RunContext(deps=slack.Deps(client=fake), model=TestModel(), usage=RunUsage())
params = tools.SlackChatPostMessageParams(channel="sandbox", text="hello")
res = slack.post_message(ctx=ctx, params=params)
assert fake.calls == [("sandbox", "hello")]
assert res["ok"] is True
```

Example integration test guard:

```py
# Run only when PYTEST_SLACK_LIVE=1 and SLACK_BOT_TOKEN is set
```

## Key Takeaways

- Tool selection and sequencing are driven by the model using the tool schemas you provide.
- `TestModel(call_tools='all')` calls all tools; use `call_tools=[]` to prevent execution when you only want to inspect tool registration.
- Always inject `deps` (real or fake) when a tool expects them; otherwise calls will fail with `NoneType` errors.

