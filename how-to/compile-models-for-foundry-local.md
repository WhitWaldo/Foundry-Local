# Run Hugging Face models on Foundry Local

Foundry Local lets you run ONNX models on your local device with high performance. While the model catalog includes pre-compiled models, you can also use any ONNX-formatted model.

In this guide, you'll learn to:

- **Convert and optimize** a Hugging Face model into the ONNX format using Olive
- **Run** the optimized model using Foundry Local

## Prerequisites

- Python 3.10 or later

## Install Olive

[Olive](https://github.com/microsoft/olive) is a toolkit for optimizing models to ONNX format.

### Bash

```bash
pip install olive-ai[auto-opt]
```

### PowerShell

```powershell
pip install olive-ai[auto-opt]
```

**💡 TIP**: Install Olive in a virtual environment using [venv](https://docs.python.org/3/library/venv.html) or [conda](https://www.anaconda.com/docs/getting-started/miniconda/main).

## Sign in to Hugging Face

We'll optimize Llama-3.2-1B-Instruct, which requires Hugging Face authentication:

### Bash

```bash
huggingface-cli login
```

### PowerShell

```powershell
huggingface-cli login
```

**Note**: You'll need to [create a Hugging Face token](https://huggingface.co/docs/hub/security-tokens) and [directly request access](https://huggingface.co/meta-llama/Llama-3.2-1B-Instruct) to the model.

## Compile the model

### Step 1: Run the Olive `auto-opt` command

Run the Olive `auto-opt` command to download, convert to ONNX, quantize, and optimize the model:

### Bash

```bash
olive auto-opt \
    --model_name_or_path meta-llama/Llama-3.2-1B-Instruct \
    --trust_remote_code \
    --output_path models/llama \
    --device cpu \
    --provider CPUExecutionProvider \
    --use_ort_genai \
    --precision int4 \
    --log_level 1
```

### PowerShell

```powershell
olive auto-opt `
    --model_name_or_path meta-llama/Llama-3.2-1B-Instruct `
    --trust_remote_code `
    --output_path models/llama `
    --device cpu `
    --provider CPUExecutionProvider `
    --use_ort_genai `
    --precision int4 `
    --log_level 1
```

**Note**: Compilation takes ~60 seconds plus model download time.

The command uses the following parameters:

| Parameter            | Description                                                                |
| -------------------- | -------------------------------------------------------------------------- |
| `model_name_or_path` | Model source: Hugging Face ID, local path, or Azure AI Model registry ID   |
| `output_path`        | Where to save the optimized model                                          |
| `device`             | Target hardware: `cpu`, `gpu`, or `npu`                                    |
| `provider`           | Execution provider (e.g., `CPUExecutionProvider`, `CUDAExecutionProvider`) |
| `precision`          | Model precision: `fp16`, `fp32`, `int4`, or `int8`                         |
| `use_ort_genai`      | Creates inference configuration files                                      |

You can substitute any model from Hugging Face or a local path - Olive handles the conversion, optimization, and quantization automatically.

### Step 2: Rename the output model

Olive places files in a generic `model` directory. Rename it to make it easier to use:

### Bash

```bash
cd models/llama
mv model llama-3.2
```

### PowerShell

```powershell
cd models/llama
Rename-Item -Path "model" -NewName "llama-3.2"
```

### Step 3: Create chat template file

A chat template is a structured format that defines how input and output messages are processed for a conversational AI model. It specifies the roles (e.g., system, user, assistant) and the structure of the conversation, ensuring that the model understands the context and generates appropriate responses.

Foundry Local requires a chat template JSON file called `inference_model.json` in order to generate the appropriate responses. The template file contains the model name and a `PromptTemplate` object - this contains a `{Content}` placeholder, which Foundry Local will inject at runtime with the user prompt.

```json
{
  "Name": "llama-3.2",
  "PromptTemplate": {
    "assistant": "{Content}",
    "prompt": "<|begin_of_text|><|start_header_id|>system<|end_header_id|>\n\nCutting Knowledge Date: December 2023\nToday Date: 26 Jul 2024\n\nYou are a helpful assistant.<|eot_id|><|start_header_id|>user<|end_header_id|>\n\n{Content}<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n"
  }
}
```

To create the chat template file, you can use the `apply_chat_template` method from the Hugging Face library:

**Note**: The following example uses the Python Hugging Face library to create a chat template. The Hugging Face library is a dependency for Olive, so if you're using the same Python virtual environment you do not need to install. If you're using a different environment, install the library with `pip install transformers`.

```python
# generate_inference_model.py
# This script generates the inference_model.json file for the Llama-3.2 model.
import json
import os
from transformers import AutoTokenizer

model_path = "models/llama/llama-3.2"
tokenizer = AutoTokenizer.from_pretrained(model_path)

chat = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "{Content}"},
]

template = tokenizer.apply_chat_template(chat, tokenize=False, add_generation_prompt=True)

json_template = {
  "Name": "llama-3.2",
  "PromptTemplate": {
    "assistant": "{Content}",
    "prompt": template
  }
}

json_file = os.path.join(model_path, "inference_model.json")
with open(json_file, "w") as f:
    json.dump(json_template, f, indent=2)
```

Run the script using:

```bash
python generate_inference_model.py
```

## Run the model

You can run your compiled model using the Foundry Local CLI, REST API, or OpenAI Python SDK. First, change the model cache directory to the models directory you created in the previous step:

### Bash

```bash
foundry cache cd models
foundry cache ls  # should show llama-3.2
```

### PowerShell

```powershell
foundry cache cd models
foundry cache ls  # should show llama-3.2
```

### Using the Foundry Local CLI

### Bash

```bash
foundry model run llama-3.2 --verbose
```

### PowerShell

```powershell
foundry model run llama-3.2 --verbose
```

### Using the REST API

### Bash

```bash
curl -X POST http://localhost:5272/v1/chat/completions \
-H "Content-Type: application/json" \
-d '{
    "model": "llama-3.2",
    "messages": [{"role": "user", "content": "What is the capital of France?"}],
    "temperature": 0.7,
    "max_tokens": 50,
    "stream": true
}'
```

### PowerShell

```powershell
Invoke-RestMethod -Uri http://localhost:5272/v1/chat/completions `
    -Method Post `
    -ContentType "application/json" `
    -Body '{
        "model": "llama-3.2",
        "messages": [{"role": "user", "content": "What is the capital of France?"}],
        "temperature": 0.7,
        "max_tokens": 50,
        "stream": true
    }'
```

### Using the OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:5272/v1",
    api_key="none",  # required but not used
)

stream = client.chat.completions.create(
    model="llama-3.2",
    messages=[{"role": "user", "content": "What is the capital of France?"}],
    temperature=0.7,
    max_tokens=50,
    stream=True,
)

for event in stream:
    print(event.choices[0].delta.content, end="", flush=True)
print("\n\n")
```

**💡 TIP**: You can use any language that supports HTTP requests. See [Integrate with Inferencing SDKs](./integrate-with-inference-sdks.md) for more options.

## Next steps

- [Learn more about Olive](https://microsoft.github.io/Olive/)
- [Integrate Foundry Local with Inferencing SDKs](./integrate-with-inference-sdks.md)
