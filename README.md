# openai-and-azure-openai-shenanigans
code examples and notes on how to get the same functionality when using azure resources with openai sdks

Added a related word doc I had made to this repo as well.

# Create Response API capable client from azure openai old endpoint
Old endpoint example - "https://xxxxxx.openai.azure.com/openai/deployments/gpt-4o-mini/chat/completions?api-version=2025-01-01-preview". Notice it says chat/completions
But we must only enter "https://xxxxxx.openai.azure.com"

```python
if os.getenv('AZURE_OPENAI_API_KEY') and os.getenv('AZURE_OPENAI_BASE_ENDPOINT'):
    azure_openai_responses_api_capable_client = AzureOpenAI(
        api_key=os.getenv("AZURE_OPENAI_API_KEY"),
        azure_endpoint=os.getenv("AZURE_OPENAI_BASE_ENDPOINT"),
        api_version="2025-03-01-preview", # according to the error I got, this is the minimum API version to support the OpenAI Responses API
    )
```


# OpenAI Agents SDK with OpenAI Key and OpenAI Tracing, and OpenAI Agents SDK with Azure OpenAI Key and OpenAI Tracing

In the below example code, these are the environmental variables:
- **MODEL_PROVIDER** = 'OPENAI' or 'AZURE'
- **LANGUAGE_MODEL** = "gpt-5.1"
- If using OpenAI for model provider:
    - **OPENAI_API_KEY_FOR_EVERYTHING** = OpenAI key for both the model and tracing
- if using Azure for model provider:
    - **AZURE_OPENAI_API_KEY**
    - **AZURE_OPENAI_API_VERSION** e.g.:- "2025-01-01-preview"
    - **AZURE_OPENAI_ENDPOINT_RESPONSES_API** - the endpoint base url. 
        - e.g. for gpt-5.1 in Azure, we got the endpoint 'https://xxxxxxx-eastus2.cognitiveservices.azure.com/openai/responses?api-version=2025-04-01-preview', but providing 'https://xxxxxxx-eastus2.cognitiveservices.azure.com' is enough.
        - Some older LLM models show a chatcompletions url. e.g.- 'https://xxxxxx-eastus2.cognitiveservices.azure.com/openai/deployments/gpt-4o/chat/completions?api-version=2025-01-01-preview'. Here I think you should include the entire URL.
    - **OPENAI_API_KEY_FOR_TRACING** - if provided, this key from openai will be used for tracing. Otherwise tracing will be disabled.

Read the comments for better understanding.

```python
from openai import AsyncAzureOpenAI, AsyncOpenAI # for initializing the default client, with its keys and stuff
from agents import (
    Agent, RunContextWrapper, ModelSettings, function_tool,
    set_tracing_disabled, set_tracing_export_api_key, set_trace_processors, # for tracing configutation
    set_default_openai_client,
    OpenAIChatCompletionsModel, # required to view generated responses from the openai trace dashboard
)
from agents.models import get_default_model_settings # used for information during debugging


class DataFrameAgents:

    # Setup tracing and model config depending on the model provider
    # ------------------------------------------------------------------------------------- #

    if os.getenv('MODEL_PROVIDER') == 'AZURE':

        openai_client = AsyncAzureOpenAI(
            api_key=os.getenv('AZURE_OPENAI_API_KEY'),
            api_version=os.getenv('AZURE_OPENAI_API_VERSION'),
            azure_endpoint=os.getenv('AZURE_OPENAI_ENDPOINT_RESPONSES_API')
        )
        set_default_openai_client(openai_client)

        # setup tracing (this must be performed after setting the default openai client above, 
        # otherwise it will go back to trying to use the azure openai key for tracing, which fails)
        trace_key = os.getenv("OPENAI_API_KEY_FOR_TRACING")
        if not trace_key or trace_key == "":
            CustomLogging.myLogger.info("OPENAI_API_KEY_FOR_TRACING not found. Tracing of the AI Agent disabled.")
            set_trace_processors([]) # Setting this to a blank array removes whatever default trace processor that tries to send all the traces to the OpenAI trace dashboard (https://platform.openai.com/traces)
            MODEL = os.getenv("LANGUAGE_MODEL")
        else:
            CustomLogging.myLogger.info("OPENAI_API_KEY_FOR_TRACING found. Tracing of the AI Agent enabled.")
            set_tracing_export_api_key(trace_key)
            MODEL = OpenAIChatCompletionsModel(
                model=os.getenv("LANGUAGE_MODEL"),
                openai_client=openai_client,
            ) # if we don't do this and leave `MODEL = os.getenv("LANGUAGE_MODEL")`, traces will still be visible in the openai traces dashboard, but when clicking on an individual generated response item, it will fail to fetch


    elif os.getenv('MODEL_PROVIDER') == 'OPENAI':

        MODEL = os.getenv("LANGUAGE_MODEL")
        openai_client = AsyncOpenAI(
            api_key=os.getenv('OPENAI_API_KEY_FOR_EVERYTHING'),
        )
        set_default_openai_client(openai_client)
        set_tracing_export_api_key(os.getenv('OPENAI_API_KEY_FOR_EVERYTHING')) # for openai agents tracing

    
    else: raise NotImplementedError

    .
    .
    .

    dataframe_agent = Agent(
        name="Agent 1",
        model=MODEL, ...
```

