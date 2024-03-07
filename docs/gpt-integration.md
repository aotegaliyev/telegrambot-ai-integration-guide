---
title: ChatGPT integration with Application
---

## **Integrating ChatGPT**

At last, we've reached the **pièce de résistance**, the cherry on the cake of our project – integrating ChatGPT! This is where things get really exciting, as we empower our bot with the brainpower of one of the most sophisticated language models out there.

### 1 **Integration structure and response model schemas**
To weave this magic, we'll roll up our sleeves and introduce a dedicated `gpt` module. Within this, we'll craft specific files – `client.py` for interacting with **ChatGPT** and `schemas.py` for structuring our data beautifully. This integration isn't just about adding smarts to our bot; it's about unlocking a whole new realm of possibilities, enabling it to converse, assist, and dazzle users like never before. Let's dive in and bring our bot to life with the power of ChatGPT!

Firstly lets configure our schema to parse result

```python
# schemas.py
from pydantic import BaseModel


class Algorithm(BaseModel):
    order: int
    text: str


class Task(BaseModel):
    algorithms: list[Algorithm]
```

### 2 **Define ENV variables**

We define OpenAI client with **API_KEY** that you can get from [open ai api-keys page](https://platform.openai.com/account/api-keys). We should update our `.env` file and `settings -> Сonfig`.

```python
# settings.py
class Config:
    ...
    OPENAI_API_KEY =  os.getenv('OPENAI_API_KEY')
```

### 3 **ChatGPT Client**

Next, we encapsulate it using the `instructor` package, utilizing a response model from schemas. To effectively integrate with **ChatGPT**, we must craft a precise prompt to elicit the correct response. Within `messages -> content`, we provide specific instructions tailored to our use case.

```python
# client.py
import logging

import instructor
from openai import OpenAI

from settings import Config
from . import schemas

logging.basicConfig(level=logging.DEBUG)

client = instructor.patch(
    OpenAI(api_key=Config.OPENAI_API_KEY), mode=instructor.Mode.MD_JSON
)


async def divide_task_to_algo(task: str) -> schemas.TaskDetail | None:
    try:
        return client.chat.completions.create(
            model='gpt-4',
            response_model=schemas.TaskDetail,
            max_retries=3,
            messages=[
                {
                    'role': 'system',
                    'content': (
                        'You can convert your task to algorithms '
                        'with its order and text.'
                    ),
                },
                {
                    'role': 'user',
                    'content': (
                        f'I have a task: {task}.'
                        'Please help me to convert it to algorithms.'
                    )
                },
            ],
            max_tokens=3000,
        )
    except Exception as e:
        logging.error(e)
        return None
```

And then we should convert this response to readable text from telegram bot, todo so we should define class property that converts response data to readable text

```python
# schemas.py
class TaskDetail(BaseModel):
    algorithms: list[Algorithm]

    @property
    def pretty_text(self) -> str:
        prefix = 'Here is your algorithm:\n'
        algorithms = '\n'.join(f'{a.order}) {a.text}'
                               for a in self.algorithms)

        return prefix + algorithms
```

### 4 **🙋🏾 Implementing `handle_task_to_algo`**

Once we've nailed down these steps, it's time to treat all incoming Telegram bot messages as tasks that need breaking down into algorithms. The next crucial phase is to meld our bot with **ChatGPT**, which we achieve by embedding the necessary logic inside the `handle_task_to_algo` function. This step is key to refining our bot's brain, giving it the **ChatGPT**-powered boost it needs to interact intelligently and effectively.

```python
# handlers.py
from telegram import Update
from telegram.ext import ContextTypes
from telegram.constants import ChatAction

from gpt import client as openai_client
from gpt.schemas import TaskDetail

# Other handlers
...

async def handle_task_to_algo(
    update: Update, context: ContextTypes.DEFAULT_TYPE,
) -> None:
    await context.bot.send_chat_action(
        chat_id=update.effective_message.chat_id,
        action=ChatAction.TYPING,
    )  # Here we show typing animation in telegram chat
    await update.message.reply_text('🧠 Processing...')

    task = update.message.text
    task_detail: TaskDetail = await openai_client.divide_task_to_algo(task)

    if not task_detail:
        await update.message.reply_text(
            'I could not convert your task to algorithms. 😔'
        )
        return None

    await update.message.reply_text(task_detail.pretty_text)
```

## **Run bot**

To start the bot just type in your project root:

```
uvicorn app:app --reload
```

## **Project structure**

Final project structure should look like:

```
project_root/
│
├── gpt/                # Directory for GPT-related modules
│   ├── __init__.py     # Initializes the gpt package
│   ├── client.py       # Contains the logic to interact with the ChatGPT API
│   └── schemas.py      # Defines pydantic models for ChatGPT data structures
│
├── taskalgobot/        # Directory for Telegram bot-specific modules
│   ├── __init__.py     # Initializes the taskalgobot package
│   ├── handlers.py     # Contains handler functions for different bot commands
│   └── keyboards.py    # Defines custom keyboards for the bot interface
│
├── app.py              # Main application entry point
├── requirements.txt    # Lists all necessary Python dependencies
├── .env                # Stores environment variables for local development
└── .env.example        # Provides a template for setting up .env files
```


And there we have it, folks! We've journeyed through the code jungle and emerged with a bot that's not just smart but ChatGPT-smart. Now, who's ready for the grand finale? Let's roll out the red carpet and unveil the fruits of our labor. Get your popcorn ready because the results of this tutorial are about to dazzle and amuse. Drumroll, please! 🥁🎉
