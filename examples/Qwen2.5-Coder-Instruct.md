# Use Qwen2.5-Coder-7B-Instruct By transformers
The most significant but also the simplest usage of Qwen2.5-Coder-7B-Instruct is using the `transformers` library. In this document, we show how to chat with Qwen2.5-Coder-7B-Instruct in either streaming mode or not.

## Basic Usage
You can just write several lines of code with `transformers` to chat with Qwen2.5-Coder-7B-Instruct. Essentially, we build the tokenizer and the model with `from_pretrained` method, and we use generate method to perform chatting with the help of chat template provided by the tokenizer. Below is an example of how to chat with Qwen2.5-Coder-7B-Instruct:

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

device = "cuda" # the device to load the model onto

# Now you do not need to add "trust_remote_code=True"
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-Coder-7B-Instruct")
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-Coder-7B-Instruct", device_map="auto").eval()

# tokenize the input into tokens

# Instead of using model.chat(), we directly use model.generate()
# But you need to use tokenizer.apply_chat_template() to format your inputs as shown below
prompt = "write a quick sort algorithm."
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": prompt}
]
text = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True
)
model_inputs = tokenizer([text], return_tensors="pt").to(device)

# Directly use generate() and tokenizer.decode() to get the output.
# Use `max_new_tokens` to control the maximum output length.
generated_ids = model.generate(
    model_inputs.input_ids,
    max_new_tokens=2048
)
generated_ids = [
    output_ids[len(input_ids):] for input_ids, output_ids in zip(model_inputs.input_ids, generated_ids)
]

response = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
```

The `apply_chat_template()` function is used to convert the messages into a format that the model can understand. 
The `add_generation_prompt` argument is used to add a generation prompt, which refers to `<|im_start|>assistant\n` to the input. Notably, we apply ChatML template for chat models following our previous practice. 
The `max_new_tokens` argument is used to set the maximum length of the response. The `tokenizer.batch_decode()` function is used to decode the response. In terms of the input, the above messages is an example to show how to format your dialog history and system prompt.


## Streaming Mode

With the help of `TextStreamer`, you can modify your chatting with CodeQwen to streaming mode. Below we show you an example of how to use it:


```python
# Repeat the code above before model.generate()
# Starting here, we add streamer for text generation.
from transformers import TextStreamer
streamer = TextStreamer(tokenizer, skip_prompt=True, skip_special_tokens=True)

# This will print the output in the streaming mode.
generated_ids = model.generate(
    model_inputs.input_ids,
    max_new_tokens=2048,
    streamer=streamer,
)
```

Besides using `TextStreamer`, we can also use `TextIteratorStreamer` which stores print-ready text in a queue, to be used by a downstream application as an iterator:

```python
# Repeat the code above before model.generate()
# Starting here, we add streamer for text generation.
from transformers import TextIteratorStreamer
streamer = TextIteratorStreamer(tokenizer, skip_prompt=True, skip_special_tokens=True)

from threading import Thread
generation_kwargs = dict(inputs=model_inputs.input_ids, streamer=streamer, max_new_tokens=2048)
thread = Thread(target=model.generate, kwargs=generation_kwargs)

thread.start()
generated_text = ""
for new_text in streamer:
    generated_text += new_text
    print(new_text, end="")
```
