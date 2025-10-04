---
layout: post
comments: false
title:  "Simple QnA using LangGraph (1/15)"
excerpt: "Getting started with LangGraph by building a simple QnA bot"
date:   2025-10-03 10:00:00
---

Ask one question, get one answer.

**Prerequisites:**
1. Setup `uv` and Python 3.11 (you can install Python 3.11 using `uv`)
2. Create an account on Groq
3. Run `uv add black langchain langchain-groq langgraph python-dotenv`
4. Run `.venv\Scripts\activate` (Windows)

```python
import os
import asyncio
from dotenv import load_dotenv

from langchain_groq import ChatGroq
from langchain_core.messages import SystemMessage, HumanMessage
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.graph.state import CompiledStateGraph

load_dotenv()

class Style:
    RED = '\033[91m'
    GREEN = '\033[92m'
    YELLOW = '\033[93m'
    BLUE = '\033[35m'
    CYAN = '\033[96m'
    RESET = '\033[0m'


MODEL = "llama-3.3-70b-versatile"


class AgentState(MessagesState):
    name: str


chat_agent = ChatGroq(model=MODEL)


def chat_node(state: AgentState) -> AgentState:
    SYSTEM_MESSAGE = SystemMessage(content="You are a helpful assistant.")

    response = chat_agent.invoke([SYSTEM_MESSAGE, *state["messages"]])

    return {
        "messages": [response.content]
    }


def build_graph() -> CompiledStateGraph:
    graph_builder = StateGraph(AgentState)

    graph_builder.add_node("chat", chat_node)
    
    graph_builder.add_edge(START, "chat")
    graph_builder.add_edge("chat", END) # END node is not mandatory

    return graph_builder.compile()


async def invoke_graph():
    state = await graph.ainvoke({'messages': HumanMessage(content="Hello, how are you?")})
    print(Style.CYAN + "Bot response: ", state['messages'][-1].content + Style.RESET)


if __name__ == "__main__":
    if "GROQ_API_KEY" not in os.environ:
        print(Style.RED + "GROQ_API_KEY is not set." + Style.RESET)
        print(Style.RED + "Create .env file at project root with GROQ_API_KEY=your_api_key" + Style.RESET)
    else:
        print(Style.YELLOW + f"GROQ_API_KEY is set: {os.environ['GROQ_API_KEY'][:10]}..." + Style.RESET)
        
        graph = build_graph()
        asyncio.run(invoke_graph())

```
