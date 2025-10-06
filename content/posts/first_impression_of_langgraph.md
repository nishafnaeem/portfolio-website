+++
title = 'My First Impression with LangGraph: Building Dynamic AI Workflows'
date = 2025-10-06T07:07:07+01:00
draft = false
disableComments = true
+++

GitHub Link: https://github.com/nishafnaeem/langggraph-rest-api

LangGraph is an agentic framework designed for building graph-based AI workflows. It lets you orchestrate complex AI systems with features like persistence, cycles, and human-in-the-loop interactions. Think of it as a way to connect different AI agents and functions in a flowchart that can make decisions and branch based on results.

I recently spent time building an AI Workflow Prototyper using LangGraph. Here's what I discovered.

## What I Built

I created a FastAPI wrapper around LangGraph that turns graph building into a REST API experience. The system I built supported two main node types:

**Function Nodes** execute Python functions and return predefined outputs. **Agent Nodes** run on prompts and take inputs from other nodes using LLM models like Claude.

The API handles the complete lifecycle - create graphs, add nodes and edges, update configurations, and execute workflows dynamically. This approach lets you draw entire AI workflows without touching code.

Here's the core api called `main.py`

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from typing import Any
from langgraph.graph import StateGraph, START, END
from langgraph.config import RunnableConfig
from _types import (
    AgentNodeConfig,
    AddNodeRequest,
    AddEdgeRequest,
    DeleteEdgeRequest,
    FunctionNodeConfig,
    GraphState,
)
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv


load_dotenv(".env")

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://127.0.0.1:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
# In-memory storage for LangGraph objects
graphs: dict[int, StateGraph] = {}
graph_id: int = 0

@app.get("/")
def main():
    return {"message": "LangGraph REST API is running"}


@app.post("/graph")
def create_graph():
    global graph_id
    graph_id += 1
    builder = StateGraph(GraphState)
    graphs[graph_id] = builder
    return {"graph_id": graph_id}


@app.get("/graph/{graph_id}")
def get_graph(graph_id: int):
    if graph_id not in graphs:
        raise HTTPException(status_code=404, detail="Graph not found")

    compiled_graph = graphs[graph_id].compile()
    graph = compiled_graph.get_graph().draw_ascii()
    return {"graph": graph}


@app.post("/graph/{graph_id}/run")
def run_graph(graph_id: int, input_data: dict[str, Any]):
    if graph_id not in graphs:
        raise HTTPException(status_code=404, detail="Graph not found")

    graph = graphs[graph_id]

    try:
        print("Running graph with input:", input_data)
        initial_state: GraphState = {"input": [input_data["text"]]}
        compiled_graph = graph.compile()
        print("compiled!")
        result = compiled_graph.invoke(initial_state)
        print("finished!")
        print(result)
        return {"result": result}
    except Exception as e:
        print(e)
        raise HTTPException(status_code=500, detail=f"Graph execution failed: {str(e)}")


@app.post("/graph/{graph_id}/node")
def add_node(
    graph_id: int,
    request: AddNodeRequest,
):
    if graph_id not in graphs:
      raise HTTPException(status_code=404, detail="Graph not found")

    graph = graphs[graph_id]

    def function_node(state: GraphState, config: RunnableConfig) -> GraphState:
        if "output" not in state:
            state["output"] = {}
        state["output"][config["metadata"]["name"]] = config["metadata"].get("output")
        return state

    def agent_node(state: GraphState, config: RunnableConfig) -> GraphState:
        input_messages = []

        # Read the outputs from GraphState for nodes that target this node with an edge,
        # then push them into input_messages as user messages
        current_node_name = config["metadata"]["name"]
        
        # Find all edges that target this node
        for edge in graph.edges:
            source_node, target_node = edge
            
            # Check if this edge targets our current node
            if target_node == current_node_name:
                # Get the output from the source node if it exists in state
                if "output" in state and source_node in state["output"]:
                    source_output = state["output"][source_node]
                    if source_output:  # Only add if there's actual content
                        input_messages.append(HumanMessage(content=str(source_output)))
        
        # If no incoming messages and there's a general input, use that
        if not input_messages and "input" in state and state["input"]:
            input_messages.append(HumanMessage(content=state["input"]))

        print("input_messages")
        print(input_messages)
        agent = create_react_agent(
            model="anthropic:claude-3-7-sonnet-latest",
            tools=[],
            prompt=config["metadata"].get("prompt"),
        )

        response = agent.invoke({"messages": input_messages})
        if len(response["messages"]) == 0:
            raise HTTPException(
                status_code=500, detail="Agent did not return any messages"
            )

        last_message = response["messages"][-1]
        if hasattr(last_message, "content"):
            last_message = last_message.content

        state["output"][config["metadata"]["name"]] = last_message
        return state

    # Add the node to the graph
    if isinstance(request.config, AgentNodeConfig):
        graph.add_node(
            request.config.name, agent_node, metadata=request.config.__dict__
        )
    else:
        graph.add_node(
            request.config.name, function_node, metadata=request.config.__dict__
        )

    return {"message": f"Node {request.config.name} added to graph {graph_id}"}

@app.put("/graph/{graph_id}/node/{node_id}")
def update_node(
    graph_id: int, node_id: str, config: FunctionNodeConfig | AgentNodeConfig
):
    if graph_id not in graphs:
        raise HTTPException(status_code=404, detail="Graph not found")

    graph = graphs[graph_id]
    
    # Check if the node exists
    if node_id not in graph.nodes:
        raise HTTPException(status_code=404, detail="Node not found")

    # Preserve all edges connected to this node
    preserved_edges = []
    for edge in graph.edges:
        source, target = edge
        # Convert START/END back to string representations for consistency
        source_str = "start" if source == START else source
        target_str = "end" if target == END else target
        
        # Save edges where this node is either source or target
        if source == node_id or target == node_id:
            preserved_edges.append((source_str, target_str))

    delete_node(graph_id, node_id)
    updated_config = config.model_copy(update={"name": node_id})
    add_node(graph_id, AddNodeRequest(config=updated_config))

    # Restore all preserved edges
    for source, target in preserved_edges:
        try:
            graph.add_edge(
                START if source == "start" else source,
                END if target == "end" else target
            )
        except Exception as e:
            # Log the error but continue with other edges
            print(f"Warning: Could not restore edge {source} -> {target}: {e}")

    return {"message": f"Config for graph {graph_id} created/updated successfully."}


@app.delete("/graph/{graph_id}/node/{node_id}")
def delete_node(graph_id: int, node_id: str):
    if graph_id not in graphs:
        raise HTTPException(status_code=404, detail="Graph not found")

    graph = graphs[graph_id]
    if node_id not in graph.nodes:
        raise HTTPException(status_code=404, detail="Node not found")

    # Remove the node from the nodes dictionary
    graph.nodes.pop(node_id)
    
    # Remove any edges that reference this node
    edges_to_remove = []
    for edge in graph.edges:
        source, target = edge
        if source == node_id or target == node_id:
            edges_to_remove.append(edge)
    
    for edge in edges_to_remove:
        graph.edges.remove(edge)
    
    return {"message": f"Node {node_id} deleted from graph {graph_id}"}


@app.post("/graph/{graph_id}/edge")
def add_edge(graph_id: int, request: AddEdgeRequest):
    if graph_id not in graphs:
        raise HTTPException(status_code=404, detail="Graph not found")

    graphs[graph_id].add_edge(START if request.source == "start" else request.source, END if request.target == "end" else request.target)

    return {"message": f"Edge {request.source} to {request.target} added."}


@app.delete("/graph/{graph_id}/edge")
def delete_edge(graph_id: int, request: DeleteEdgeRequest):
    if graph_id not in graphs:
        raise HTTPException(status_code=404, detail="Graph not found")

    graph = graphs[graph_id]
    graph.edges.remove((START if request.source == "start" else request.source, END if request.target == "end" else request.target))
    
    return {"message": f"Edge {request.source} to {request.target} removed."}

```

The API provides CRUD operations for graphs, nodes, edges, and configurations. This makes it perfect for building visual workflow builders where users can drag and drop AI components. 


## The Positive Experience

**Learning Curve Was Manageable**  
LangGraph doesn't overwhelm you with complexity upfront. The basic concepts of nodes, edges, and state flow are intuitive for anyone who's worked with flowcharts.

**Runtime Flexibility Impressed Me**  
You can modify graph structure without recompiling. Need to change how nodes connect? Update edges on the fly. Want to alter a prompt? Change node configuration during execution.

```python
@app.post("/graphs/{graph_id}/node/{node_id}/config/update")
def create_or_update_config(graph_id: int, node_id: str, config: FunctionNodeConfig):
    graph = graphs[graph_id]
    graph.nodes[node_id].metadata = config.__dict__
    return {"message": "Config updated successfully"}
```

**LLM Integration Felt Natural**  
The `create_react_agent` function handles Claude integration seamlessly. You provide prompts and tools, and LangGraph manages the conversation flow between nodes.

**Conditional Logic Works Well**  
LangGraph supports conditional edges where routing depends on node output. When criteria aren't known beforehand, dynamic re-routing through Command and Send APIs covers unknown scenarios.

**Built-in Validation Saves Time**  
The framework includes graph validation that catches structural issues before execution. No more runtime surprises from disconnected nodes or circular references.

## The Challenges I Encountered

**Global State Management Gets Tricky**  
The biggest pain point involves parallel node execution. When multiple nodes forward the same state keys to another node, LangGraph throws `INVALID_CONCURRENT_GRAPH_UPDATE` errors.

Consider this diamond pattern:
```
    Node A
   /      \
Node B    Node C  (parallel)
   \      /
    Node D
```

Both Node B and Node C try to write to `state["output"][node_name]` simultaneously. The solution requires reducer functions or node-specific state channels - not obvious for newcomers.

**State Structure Feels Rigid**  
The global state structure gets fixed at startup. You can't add new keys during runtime. Try passing undeclared keys from nodes, and they'll be silently ignored.

```python
# This fails silently if 'custom_key' wasn't declared in GraphState
state["custom_key"] = "some_value"  # Gets dropped
```

Error reporting doesn't clearly indicate that global state ignores unexpected keys. You'll get "key not found" errors without understanding why.

**Edge Management Could Be Better**  
Nodes store as dictionaries, edges as sets. Updating connections requires manual set manipulation:

```python
def remove_edges(graph_id: int, node_id: str):
    graph = graphs[graph_id]
    final_edges = set()
    for edge in graph.edges:
        if node_id in edge[0] or node_id in edge[1]:
            continue
        final_edges.add(edge)
    graph.edges = final_edges
```

This approach feels error-prone and lacks type safety.

## Overall Experience

LangGraph delivers power for complex AI workflows. It excels at dynamic agent orchestration and handles stateful operations well. The framework hits a sweet spot between flexibility and structure.

State management requires careful planning, especially for parallel execution. Understanding the global state patterns early prevents frustration later.

The learning curve is reasonable, but documentation could better cover advanced scenarios like parallel nodes and state conflicts.

## When to Use LangGraph

**Perfect For:**
- Building user-driven workflow builders where graphs change based on input
- Multi-step AI processes that need persistence and state management  
- Applications requiring conditional routing and dynamic decision-making
- Complex agent orchestration with multiple LLM calls

**Avoid When:**
- Simple linear AI workflows (overkill for basic chains)
- Real-time applications where state management overhead matters
- Teams unfamiliar with graph-based thinking and state management patterns
- Projects requiring frequent parallel node execution without proper state design

## Final Thoughts

LangGraph shows promise for sophisticated AI applications. It's not perfect - the state management challenges are real - but it's definitely capable of handling complex workflow requirements.

The framework works best when you understand its constraints upfront. Plan your state structure carefully, design around parallel execution limitations, and you'll find LangGraph quite powerful.

For teams building dynamic AI workflows, LangGraph deserves serious consideration. Just be prepared to invest time in understanding its state management patterns.
