
### Part 1: Build a Basic Chatbot

**`add_messages`** educer function in LangGraph. It's used to manage and update the list of messages in the state.

**Reducer Function**: In LangGraph, a reducer function like add_messages helps manage state changes. It takes the current state and an action as input, then returns a new state based on those inputs.

Usage in State Management: When you invoke a graph in LangGraph, **`add_messages`** is used to **append new messages to the existing list of messages in the state**.

**`add_message`** append, update, and remove messages in the existing list of messages in the state.

### Part 2: Enhancing the Chatbot with Tools

#### Integrating tools for search results

Install the Required Packages to Add Tool (tavily-python and langchain_community) for Real-Time Search (e.g., Karachi Weather)

```bash
%%capture --no-stderr
%pip install -U tavily-python langchain_community
```

Setting the API key in the environment variable TAVILY_API_KEY

```bash
os.environ["TAVILY_API_KEY"] = userdata.get("TAVILY_API_KEY")
```

Next, define the tool

**`TavilySearchResults(max_results=2)`** initializes a search tool that retrieves and returns up to 2 search results.

Initializes a search tool that retrieves and returns up to 2 search results.
```bash
tool = TavilySearchResults(max_results=2)
tools = [tool]
```

**`bind_tools()`** Integrates the specified tools with the language model, enabling the model to invoke these tools during its execution

```bash
llm_with_tools = llm.bind_tools(tools)
```
This line integrates the tools list with the language model llm, resulting in a new instance (llm_with_tools) that can use these tools during its operations.

**`add_conditional_edges`** Define conditional edges for the chatbot node based on tool usage
```bash
graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition
)
```


**rm note**

**`ToolNode`** is a built-in function of LangGraph. We can create our own custom function that provides a list of tools such as a search engine, Excel file operations, weather search, or database search in SQL. To integrate these tools into LangGraph, pass them into a custom `ToolNode`. We’ll build our own `BasicToolNode` and replace LangGraph’s prebuilt `ToolNode` and `tools_condition` with it, using

ToolNode(tools=[your_tool]).


**ToolNode** is essentially a node within a workflow or graph that is designated to execute a specific tool or function when called.

**ToolNode** is a LangChain Runnable that takes graph state (with a list of messages) as input and outputs state update with the result of tool calls.

**ToolNode** is just a class that helps with this execution. You pass it your list of tools, and internally it stores them as name-function pairs.


**function call** provide external data to llm
add anything in function call

We can add data from our database or search for things on the web available in part 2

Note: The LLM never directly calls the tool. Instead, it generates a query and specifies which tool is required to run

Relationship between the LLM and LangGraph, and why the tool needs to be passed twice.

**LLM and LangGraph**

LangGraph has nodes.

The LLM doesn't know which tool to call by itself. LangGraph manages the tools.

To enable the LLM to call the tool, we bind the tool to the LLM.

The LLM then generates the query and specifies which tool is required to run.

LangGraph handles the actual tool invocation.

That's why we pass the tool twice:

(Part 2: Enhancing the Chatbot with Tools)

**Initializes a search tool that retrieves and returns up to 2 search results**
```bash
tool = TavilySearchResults(max_results=2)
tools = [tool]
```

**Modification: tell the LLM which tools it can call**
```bash
llm_with_tools = llm.bind_tools(tools)
```

**`add_conditional_edges()`** is a function used to add conditional connections between nodes in a graph, based on specific conditions. It helps create dynamic workflows where the next step depends on certain criteria being met.

**`graph_builder.add_conditional_edges("chatbot", tools_condition)`**

Purpose: Adds an edge from the `chatbot` node to another node, only if the `tools_condition` is satisfied.

### Part 3: Adding Memory to the Chatbot

**Memory checkpointing**  
provides state to the chatbot, allowing it to maintain the history based on `thread_id`.


Add memory saver for checkpointing:
```bash
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
```

Compile the graph with memory checkpointing:
```bash
graph = graph_builder.compile(checkpointer=memory)
```
Checkpointing the state as the graph works through each node.

**`graph = graph_builder.compile(checkpointer=MemorySaver())`** compiles the graph with memory checkpointing, allowing it to save and restore the state of the graph's nodes during its execution.

To interact with the bot, first, pick a thread to use as the key for this conversation:
```bash
config = {"configurable": {"thread_id": "1"}}
```

The **chatbot can remember the state of the conversation within a given thread** but won't retain context if the thread ID is changed.

goes into a checkpoint? To inspect a graph's state for a given config at any time, call get_state(config).

```bash
# Fetch the current state of the graph for the given configuration
snapshot = graph.get_state(config)

# Inspect the snapshot
snapshot

```

Retrieve the next node to execute (empty if the graph ended this turn)

```bash
snapshot.next
```
(since the graph ended this turn, `next` is empty. If you fetch a state from within a graph invocation, next tells which node will execute next)

### Part 4: Human-in-the-loop

**Freezer and un unfreez node**

LangGraph's interrupt_before functionality to always break the tool node.

**`graph = graph_builder.compile(checkpointer=memory, interrupt_before=["tools"])`**

**`interrupt_before=["tools"]`** This is new! it pose the node and can perform read and write opration

Note: can also interrupt __after__ tools, if desired.
**`interrupt_after=["tools"]`** it pose the node and can perform read and write opration


**`get_state()`**  Retrieve the current state of a particular object

**`.next`** Find out what the next node to execute

**`.config`** Show the current configuration of the state

**`.value`**  Show all values

**inspect the graph state to confirm it worked**

* `snapshot = graph.get_state(config)`  # Retrieve the current state of the graph
* `snapshot.config`  # Show the current configuration of the state
* `snapshot.next`  # Retrieve the next node that will execute in the graph. If the graph ended this turn, next will be empty


```bash
snapshot = graph.get_state(config)  # Retrieve the current state or status of the graph
snapshot.next  # Find out what the next node to execute is
snapshot.config  # Show the current configuration of the state
snapshot.values  # Show values
snapshot.values['messages']  # Show messages

existing_message = snapshot.values["messages"][-1]  # Find out the last message
existing_message.pretty_print()  # Print message

existing_message.tool_calls  # Get tool data
```

`None` will append nothing new to the current state, letting it resume as if it had never been interrupted

```bash
events = graph.stream(None, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```
### Part 5: Manually Updating the State

**Note**
query: LangGraph,  
now update the query before go to the tool

Save the state new_messages[-1]
than update state `graph.get_state(config).values["messages"][-2:]`

```bash
new_messages = [
 ToolMessage(content=answer, tool_call_id=existing_message.tool_calls[0]["id"]),  AIMessage(content=answer),
]

new_messages[-1].pretty_print()
graph.update_state( config, {"messages": new_messages},)
```
### Part 6: Customizing State



**add_messages** : append new messages to the existing list of messages in the state in the graph.
Define the State class with a list of messages
```bash
class State(TypedDict):
    messages: Annotated[list, add_messages]
    ask_human: bool
```

(Part 6: Customizing State)

**`ask_human: bool`** flag indicates whether only a human can give instructions, true means only a human can give instructions

Bind the llm to a tool definition, a pydantic model, or a json schema
```bash
llm_with_tools = llm.bind_tools(tools + [RequestAssistance])
```
**`[RequestAssistance]`**  human use RequestAssistance Schema

Now llm triger both Tool and RequestAssistance Schema

**Notice:** the LLM has invoked the "`RequestAssistance`" tool we provided it, and the interrupt has been set. Let's inspect the graph state to confirm.


### Part 7: Time Travel













