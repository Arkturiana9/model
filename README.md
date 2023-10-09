---
datasets:
- IlyaGusev/ru_turbo_alpaca
- IlyaGusev/ru_turbo_saiga
- IlyaGusev/ru_sharegpt_cleaned
- IlyaGusev/oasst1_ru_main_branch
- IlyaGusev/ru_turbo_alpaca_evol_instruct
- lksy/ru_instruct_gpt4
language:
- ru
pipeline_tag: conversational
license: cc-by-4.0
---

# Saiga2 7B, Russian LLaMA2-based chatbot

Based on [Mistral OpenOrca](https://huggingface.co/Open-Orca/Mistral-7B-OpenOrca).

This is an adapter-only version.

Llama.cpp version: TBA

Colab: TBA

Training code: [link](https://github.com/IlyaGusev/rulm/tree/master/self_instruct).

```python
from peft import PeftModel, PeftConfig
from transformers import AutoModelForCausalLM, AutoTokenizer, GenerationConfig

MODEL_NAME = "IlyaGusev/saiga_mistral_7b"
DEFAULT_MESSAGE_TEMPLATE = "<s>{role}\n{content}</s>"
DEFAULT_RESPONSE_TEMPLATE = "<s>bot\n"
DEFAULT_SYSTEM_PROMPT = "Ты — Сайга, русскоязычный автоматический ассистент. Ты разговариваешь с людьми и помогаешь им."

class Conversation:
    def __init__(
        self,
        message_template=DEFAULT_MESSAGE_TEMPLATE,
        system_prompt=DEFAULT_SYSTEM_PROMPT,
        start_token_id=1,
    ):
        self.message_template = message_template
        self.start_token_id = start_token_id
        self.messages = [{
            "role": "system",
            "content": system_prompt
        }]

    def get_start_token_id(self):
        return self.start_token_id

    def get_bot_token_id(self):
        return self.bot_token_id

    def add_user_message(self, message):
        self.messages.append({
            "role": "user",
            "content": message
        })

    def add_bot_message(self, message):
        self.messages.append({
            "role": "bot",
            "content": message
        })

    def get_prompt(self, tokenizer):
        final_text = ""
        for message in self.messages:
            message_text = self.message_template.format(**message)
            final_text += message_text
        final_text += tokenizer.decode([self.start_token_id, self.bot_token_id])
        return final_text.strip()


def generate(model, tokenizer, prompt, generation_config):
    data = tokenizer(prompt, return_tensors="pt")
    data = {k: v.to(model.device) for k, v in data.items()}
    output_ids = model.generate(
        **data,
        generation_config=generation_config
    )[0]
    output_ids = output_ids[len(data["input_ids"][0]):]
    output = tokenizer.decode(output_ids, skip_special_tokens=True)
    return output.strip()

config = PeftConfig.from_pretrained(MODEL_NAME)
model = AutoModelForCausalLM.from_pretrained(
    config.base_model_name_or_path,
    load_in_8bit=True,
    torch_dtype=torch.float16,
    device_map="auto"
)
model = PeftModel.from_pretrained(
    model,
    MODEL_NAME,
    torch_dtype=torch.float16
)
model.eval()

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME, use_fast=False)
generation_config = GenerationConfig.from_pretrained(MODEL_NAME)
print(generation_config)

inputs = ["Почему трава зеленая?", "Сочини длинный рассказ, обязательно упоминая следующие объекты. Дано: Таня, мяч"]
for inp in inputs:
    conversation = Conversation()
    conversation.add_user_message(inp)
    prompt = conversation.get_prompt(tokenizer)

    output = generate(model, tokenizer, prompt, generation_config)
    print(inp)
    print(output)
    print()
    print("==============================")
    print()
```

Examples:
```
User: Почему трава зеленая? 
Saiga: ```

```
User: Сочини длинный рассказ, обязательно упоминая следующие объекты. Дано: Таня, мяч
Saiga:
```

- dataset code revision d0d123dd221e10bb2a3383bcb1c6e4efe1b4a28a
- wandb [link](https://wandb.ai/ilyagusev/rulm_self_instruct/runs/ip1qmm9p)
- 5 datasets: ru_turbo_saiga, ru_sharegpt_cleaned, oasst1_ru_main_branch, gpt_roleplay_realm, ru_instruct_gpt4
- Datasets merging script: [create_short_chat_set.py](https://github.com/IlyaGusev/rulm/blob/d0d123dd221e10bb2a3383bcb1c6e4efe1b4a28a/self_instruct/src/data_processing/create_short_chat_set.py)
- saiga_mistral_7b vs saiga2_13b: 243-31-141