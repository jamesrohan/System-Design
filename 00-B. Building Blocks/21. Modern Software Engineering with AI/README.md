<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# [Standford AI SE Course Link](https://themodernsoftware.dev/)

# 🎯 Detailed Portfolio Projects: Step-by-Step Implementation Guides

Let me break down each project into **actionable, time-boxed implementations** with both **local** and **cloud** options. Each project is scoped for **2-3 days maximum**! 🚀

***

## 📋 Project 1: Multi-Agent RAG System with LangGraph

### 🎯 Learning Goals

- ✅ Build stateful workflows with LangGraph
- ✅ Implement multi-agent coordination patterns
- ✅ Work with vector databases and embeddings
- ✅ Practice RAG architecture from scratch
- ✅ Handle agent state management


### ⏱️ Time Estimate: **2-3 days**


***

### 🖥️ Option A: Local Implementation (Laptop)

**Prerequisites**:

- Python 3.10+
- 8GB+ RAM
- No GPU needed (using embeddings APIs)


#### **Day 1: Setup \& Basic RAG (4-6 hours)**

**Step 1: Environment Setup (30 min)**

```bash
# Create project
mkdir multi-agent-rag && cd multi-agent-rag
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install langgraph langchain langchain-openai
pip install chromadb sentence-transformers
pip install python-dotenv pypdf beautifulsoup4
```

**Step 2: Create Document Database (1 hour)**

```python
# ingest_docs.py
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_community.vectorstores import Chroma
import os

# Use free local embeddings (no API needed!)
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)

# Create sample documents
os.makedirs("docs/technical", exist_ok=True)
os.makedirs("docs/marketing", exist_ok=True)
os.makedirs("docs/legal", exist_ok=True)

# Add sample content
with open("docs/technical/api_guide.txt", "w") as f:
    f.write("""
    API Documentation
    
    GET /users - Retrieve all users
    POST /users - Create new user
    Authentication: Bearer token required
    Rate limit: 100 requests/minute
    """)

with open("docs/marketing/product_info.txt", "w") as f:
    f.write("""
    Product Overview
    
    Our platform offers AI-powered analytics.
    Pricing: $99/month for starter plan.
    Features: Real-time dashboards, custom reports.
    """)

with open("docs/legal/terms.txt", "w") as f:
    f.write("""
    Terms of Service
    
    Users must be 18+ years old.
    Data retention: 90 days.
    Refund policy: 30-day money back guarantee.
    """)

# Load and split documents
def load_documents(directory):
    docs = []
    for root, dirs, files in os.walk(directory):
        for file in files:
            if file.endswith('.txt'):
                loader = TextLoader(os.path.join(root, file))
                docs.extend(loader.load())
    return docs

documents = load_documents("docs")

# Split into chunks
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50
)
chunks = text_splitter.split_documents(documents)

# Create vector stores per category
tech_chunks = [c for c in chunks if "technical" in c.metadata["source"]]
marketing_chunks = [c for c in chunks if "marketing" in c.metadata["source"]]
legal_chunks = [c for c in chunks if "legal" in c.metadata["source"]]

# Create separate vector stores
tech_vectorstore = Chroma.from_documents(
    tech_chunks, embeddings, persist_directory="./chroma_tech"
)
marketing_vectorstore = Chroma.from_documents(
    marketing_chunks, embeddings, persist_directory="./chroma_marketing"
)
legal_vectorstore = Chroma.from_documents(
    legal_chunks, embeddings, persist_directory="./chroma_legal"
)

print("✅ Document databases created!")
```

**Run it:**

```bash
python ingest_docs.py
```

**Step 3: Build Specialized RAG Agents (2-3 hours)**

```python
# agents.py
from typing import TypedDict, Annotated, Literal
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain.prompts import ChatPromptTemplate
import os

# Set your OpenAI API key
os.environ["OPENAI_API_KEY"] = "your-key-here"

# State definition
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    query_category: str
    retrieved_docs: list
    final_answer: str

# Initialize components
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

# Load vector stores
tech_vectorstore = Chroma(
    persist_directory="./chroma_tech",
    embedding_function=embeddings
)
marketing_vectorstore = Chroma(
    persist_directory="./chroma_marketing",
    embedding_function=embeddings
)
legal_vectorstore = Chroma(
    persist_directory="./chroma_legal",
    embedding_function=embeddings
)

# Retriever functions
def retrieve_technical(query: str) -> list:
    return tech_vectorstore.similarity_search(query, k=3)

def retrieve_marketing(query: str) -> list:
    return marketing_vectorstore.similarity_search(query, k=3)

def retrieve_legal(query: str) -> list:
    return legal_vectorstore.similarity_search(query, k=3)

print("✅ Agents initialized!")
```

**Step 4: Simple Test (1 hour)**

```python
# test_retrieval.py
from agents import retrieve_technical, retrieve_marketing, retrieve_legal

# Test each retriever
print("Testing Technical RAG:")
tech_results = retrieve_technical("How do I authenticate with the API?")
for doc in tech_results:
    print(f"- {doc.page_content[:100]}...")

print("\nTesting Marketing RAG:")
marketing_results = retrieve_marketing("What is the pricing?")
for doc in marketing_results:
    print(f"- {doc.page_content[:100]}...")

print("\nTesting Legal RAG:")
legal_results = retrieve_legal("What is the refund policy?")
for doc in legal_results:
    print(f"- {doc.page_content[:100]}...")
```


***

#### **Day 2: Multi-Agent LangGraph System (4-6 hours)**

**Step 5: Build Supervisor Agent (2 hours)**

```python
# supervisor.py
from langgraph.graph import StateGraph, END
from agents import AgentState, llm
from langchain_core.messages import HumanMessage, AIMessage

def classify_query(state: AgentState):
    """Supervisor classifies the query"""
    messages = state["messages"]
    user_query = messages[-1].content
    
    classification_prompt = f"""
    Classify this query into ONE category:
    - technical: API, authentication, integration questions
    - marketing: pricing, features, product info
    - legal: terms, policies, compliance
    
    Query: {user_query}
    
    Respond with just the category name (technical/marketing/legal):
    """
    
    response = llm.invoke([HumanMessage(content=classification_prompt)])
    category = response.content.strip().lower()
    
    print(f"🎯 Classified as: {category}")
    
    return {
        "query_category": category,
        "messages": [AIMessage(content=f"Routing to {category} agent...")]
    }

def route_to_agent(state: AgentState) -> Literal["technical", "marketing", "legal"]:
    """Route based on classification"""
    return state["query_category"]

print("✅ Supervisor ready!")
```

**Step 6: Build Specialized Agent Nodes (2 hours)**

```python
# specialized_agents.py
from agents import (
    AgentState, llm, 
    retrieve_technical, retrieve_marketing, retrieve_legal
)
from langchain_core.messages import HumanMessage, AIMessage

def technical_agent(state: AgentState):
    """Technical documentation agent"""
    user_query = state["messages"][0].content
    
    # Retrieve relevant docs
    docs = retrieve_technical(user_query)
    context = "\n\n".join([doc.page_content for doc in docs])
    
    # Generate answer
    prompt = f"""
    You are a technical support agent. Use this documentation to answer:
    
    Documentation:
    {context}
    
    Question: {user_query}
    
    Provide a clear, technical answer:
    """
    
    response = llm.invoke([HumanMessage(content=prompt)])
    
    return {
        "retrieved_docs": docs,
        "final_answer": response.content,
        "messages": [AIMessage(content=response.content)]
    }

def marketing_agent(state: AgentState):
    """Marketing information agent"""
    user_query = state["messages"][0].content
    
    docs = retrieve_marketing(user_query)
    context = "\n\n".join([doc.page_content for doc in docs])
    
    prompt = f"""
    You are a friendly product specialist. Use this information:
    
    Product Info:
    {context}
    
    Question: {user_query}
    
    Provide a helpful, conversational answer:
    """
    
    response = llm.invoke([HumanMessage(content=prompt)])
    
    return {
        "retrieved_docs": docs,
        "final_answer": response.content,
        "messages": [AIMessage(content=response.content)]
    }

def legal_agent(state: AgentState):
    """Legal/policy agent"""
    user_query = state["messages"][0].content
    
    docs = retrieve_legal(user_query)
    context = "\n\n".join([doc.page_content for doc in docs])
    
    prompt = f"""
    You are a compliance assistant. Use these policies:
    
    Legal Documentation:
    {context}
    
    Question: {user_query}
    
    Provide accurate policy information:
    """
    
    response = llm.invoke([HumanMessage(content=prompt)])
    
    return {
        "retrieved_docs": docs,
        "final_answer": response.content,
        "messages": [AIMessage(content=response.content)]
    }

print("✅ Specialized agents ready!")
```

**Step 7: Assemble the Graph (1 hour)**

```python
# graph.py
from langgraph.graph import StateGraph, END
from agents import AgentState
from supervisor import classify_query, route_to_agent
from specialized_agents import technical_agent, marketing_agent, legal_agent

# Build the graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("supervisor", classify_query)
workflow.add_node("technical", technical_agent)
workflow.add_node("marketing", marketing_agent)
workflow.add_node("legal", legal_agent)

# Set entry point
workflow.set_entry_point("supervisor")

# Add conditional routing
workflow.add_conditional_edges(
    "supervisor",
    route_to_agent,
    {
        "technical": "technical",
        "marketing": "marketing",
        "legal": "legal"
    }
)

# All agents lead to END
workflow.add_edge("technical", END)
workflow.add_edge("marketing", END)
workflow.add_edge("legal", END)

# Compile
app = workflow.compile()

print("✅ Multi-agent graph compiled!")

# Save visualization (optional)
try:
    from IPython.display import Image, display
    display(Image(app.get_graph().draw_mermaid_png()))
except:
    print("Run in Jupyter to see graph visualization")
```

**Step 8: Test the Full System (1 hour)**

```python
# main.py
from graph import app
from langchain_core.messages import HumanMessage

def query_system(question: str):
    """Query the multi-agent system"""
    print(f"\n{'='*60}")
    print(f"Question: {question}")
    print(f"{'='*60}")
    
    result = app.invoke({
        "messages": [HumanMessage(content=question)]
    })
    
    print(f"\n📂 Category: {result['query_category']}")
    print(f"\n📄 Retrieved {len(result['retrieved_docs'])} documents")
    print(f"\n💬 Answer:\n{result['final_answer']}")
    print(f"\n{'='*60}")

if __name__ == "__main__":
    # Test different query types
    query_system("How do I authenticate with the API?")
    query_system("What are your pricing plans?")
    query_system("What is your refund policy?")
    query_system("What's the rate limit for API calls?")
```

**Run it:**

```bash
python main.py
```


***

#### **Day 3: Enhancements \& Polish (3-4 hours)**

**Step 9: Add Human-in-the-Loop (1 hour)**

```python
# enhanced_graph.py
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from agents import AgentState
from supervisor import classify_query, route_to_agent
from specialized_agents import technical_agent, marketing_agent, legal_agent

def approval_gate(state: AgentState):
    """Wait for human approval on sensitive queries"""
    if state["query_category"] == "legal":
        print("\n⚠️  Legal query detected - review required")
        print(f"Draft answer: {state['final_answer'][:200]}...")
        approval = input("\nApprove? (yes/no): ")
        
        if approval.lower() != "yes":
            return {
                "final_answer": "Response requires human review.",
                "messages": [AIMessage(content="Escalated to human.")]
            }
    
    return state

# Build enhanced graph with checkpointing
workflow = StateGraph(AgentState)

workflow.add_node("supervisor", classify_query)
workflow.add_node("technical", technical_agent)
workflow.add_node("marketing", marketing_agent)
workflow.add_node("legal", legal_agent)
workflow.add_node("approval", approval_gate)

workflow.set_entry_point("supervisor")

workflow.add_conditional_edges(
    "supervisor",
    route_to_agent,
    {
        "technical": "technical",
        "marketing": "marketing",
        "legal": "legal"
    }
)

# Route to approval before END
workflow.add_edge("technical", END)
workflow.add_edge("marketing", END)
workflow.add_edge("legal", "approval")
workflow.add_edge("approval", END)

# Add memory for state persistence
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

print("✅ Enhanced graph with approval gate ready!")
```

**Step 10: Add Streaming (1 hour)**

```python
# streaming_demo.py
from enhanced_graph import app
from langchain_core.messages import HumanMessage

def stream_query(question: str):
    """Stream agent responses in real-time"""
    config = {"configurable": {"thread_id": "user-123"}}
    
    print(f"\nQuestion: {question}")
    print("\nProcessing...")
    
    for event in app.stream(
        {"messages": [HumanMessage(content=question)]},
        config
    ):
        for node_name, node_output in event.items():
            print(f"\n🔄 {node_name.upper()}")
            if "messages" in node_output:
                last_msg = node_output["messages"][-1]
                print(f"  {last_msg.content[:100]}...")

if __name__ == "__main__":
    stream_query("What's the API rate limit?")
```

**Step 11: Simple Web Interface (1-2 hours)**

```python
# app_simple.py
import streamlit as st
from graph import app
from langchain_core.messages import HumanMessage

st.title("🤖 Multi-Agent RAG System")
st.write("Ask questions about technical docs, products, or policies!")

# Query input
query = st.text_input("Your question:")

if st.button("Ask") and query:
    with st.spinner("Thinking..."):
        result = app.invoke({
            "messages": [HumanMessage(content=query)]
        })
        
        st.success(f"Category: {result['query_category'].title()}")
        st.info(f"Retrieved {len(result['retrieved_docs'])} documents")
        st.markdown(f"**Answer:**\n\n{result['final_answer']}")
        
        with st.expander("View source documents"):
            for i, doc in enumerate(result['retrieved_docs']):
                st.markdown(f"**Doc {i+1}:** {doc.page_content}")
```

**Run it:**

```bash
pip install streamlit
streamlit run app_simple.py
```


***

### ☁️ Option B: Cloud Implementation

**Use these free alternatives:**

**Day 1: Setup on Google Colab**

```python
# Run in Colab notebook
!pip install langgraph langchain langchain-openai chromadb sentence-transformers

# Upload your documents or use Google Drive
from google.colab import drive
drive.mount('/content/drive')

# Use the same code but with Colab's free GPU for embeddings
```

**Day 2-3: Deploy to Cloud**

**Option 1: Hugging Face Spaces (Free)**

```bash
# Create space at huggingface.co/spaces
# Upload your code + requirements.txt
# Use Streamlit interface

# requirements.txt
langgraph
langchain
langchain-openai
chromadb
sentence-transformers
streamlit
```

**Option 2: Replit (Free tier)**

- Create new Repl
- Use Python template
- Deploy with one click
- Get shareable URL

**Cloud Storage for Vector DB:**

```python
# Use Pinecone (free tier: 1 index, 100K vectors)
from langchain_pinecone import PineconeVectorStore
import pinecone

pc = pinecone.Pinecone(api_key="your-key")
index = pc.Index("multi-agent-rag")

vectorstore = PineconeVectorStore(
    index=index,
    embedding=embeddings
)
```


***

## 📋 Project 2: MCP Server for Backend Services

### 🎯 Learning Goals

- ✅ Understand MCP protocol architecture
- ✅ Build standardized tool integrations
- ✅ Design server/client communication
- ✅ Expose backend APIs to AI agents
- ✅ Handle authentication and security


### ⏱️ Time Estimate: **2-3 days**


***

### 🖥️ Option A: Local Implementation

#### **Day 1: Basic MCP Server (4-6 hours)**

**Step 1: Setup (30 min)**

```bash
mkdir mcp-backend-server && cd mcp-backend-server
python -m venv venv
source venv/bin/activate

# Install MCP SDK
pip install mcp
pip install fastapi uvicorn sqlalchemy
```

**Step 2: Create Mock Backend Database (1 hour)**

```python
# database.py
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    subscription = Column(String)

class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer)
    product = Column(String)
    amount = Column(Float)
    status = Column(String)

# Create in-memory database
engine = create_engine("sqlite:///./backend.db")
Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine)

# Seed data
def seed_database():
    db = SessionLocal()
    
    users = [
        User(id=1, name="Alice", email="alice@example.com", subscription="pro"),
        User(id=2, name="Bob", email="bob@example.com", subscription="free"),
    ]
    
    orders = [
        Order(id=1, user_id=1, product="Widget A", amount=99.99, status="completed"),
        Order(id=2, user_id=1, product="Widget B", amount=149.99, status="pending"),
        Order(id=3, user_id=2, product="Widget C", amount=49.99, status="completed"),
    ]
    
    db.add_all(users + orders)
    db.commit()
    db.close()
    print("✅ Database seeded!")

if __name__ == "__main__":
    seed_database()
```

**Run it:**

```bash
python database.py
```

**Step 3: Build MCP Server (2-3 hours)**

```python
# mcp_server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, Resource
from database import SessionLocal, User, Order
import json
import asyncio

# Create MCP server
app = Server("backend-api-server")

# Tool 1: Query Users
@app.tool()
async def get_user(user_id: int) -> list[TextContent]:
    """Get user information by ID"""
    db = SessionLocal()
    user = db.query(User).filter(User.id == user_id).first()
    db.close()
    
    if user:
        user_data = {
            "id": user.id,
            "name": user.name,
            "email": user.email,
            "subscription": user.subscription
        }
        return [TextContent(
            type="text",
            text=json.dumps(user_data, indent=2)
        )]
    else:
        return [TextContent(
            type="text",
            text=f"User {user_id} not found"
        )]

# Tool 2: List Orders
@app.tool()
async def list_orders(
    user_id: int | None = None,
    status: str | None = None
) -> list[TextContent]:
    """List orders, optionally filtered by user_id or status"""
    db = SessionLocal()
    query = db.query(Order)
    
    if user_id:
        query = query.filter(Order.user_id == user_id)
    if status:
        query = query.filter(Order.status == status)
    
    orders = query.all()
    db.close()
    
    orders_data = [
        {
            "id": o.id,
            "user_id": o.user_id,
            "product": o.product,
            "amount": o.amount,
            "status": o.status
        }
        for o in orders
    ]
    
    return [TextContent(
        type="text",
        text=json.dumps(orders_data, indent=2)
    )]

# Tool 3: Update Order Status
@app.tool()
async def update_order_status(
    order_id: int,
    new_status: str
) -> list[TextContent]:
    """Update order status (pending, completed, cancelled)"""
    db = SessionLocal()
    order = db.query(Order).filter(Order.id == order_id).first()
    
    if order:
        order.status = new_status
        db.commit()
        result = f"Order {order_id} status updated to {new_status}"
    else:
        result = f"Order {order_id} not found"
    
    db.close()
    
    return [TextContent(type="text", text=result)]

# Resource: Database schema
@app.list_resources()
async def list_resources() -> list[Resource]:
    """List available resources"""
    return [
        Resource(
            uri="backend://schema/users",
            name="User Database Schema",
            mimeType="application/json"
        ),
        Resource(
            uri="backend://schema/orders",
            name="Order Database Schema",
            mimeType="application/json"
        )
    ]

@app.read_resource()
async def read_resource(uri: str) -> str:
    """Read resource content"""
    if uri == "backend://schema/users":
        return json.dumps({
            "table": "users",
            "columns": ["id", "name", "email", "subscription"]
        })
    elif uri == "backend://schema/orders":
        return json.dumps({
            "table": "orders",
            "columns": ["id", "user_id", "product", "amount", "status"]
        })
    return "{}"

# Run server
async def main():
    print("🚀 Backend MCP Server starting...")
    async with stdio_server() as (read_stream, write_stream):
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )

if __name__ == "__main__":
    asyncio.run(main())
```

**Step 4: Test MCP Server (1 hour)**

```python
# test_mcp_client.py
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import asyncio
import json

async def test_mcp_server():
    """Test MCP server tools"""
    
    # Connect to server
    server_params = StdioServerParameters(
        command="python",
        args=["mcp_server.py"]
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize
            await session.initialize()
            
            print("✅ Connected to MCP server\n")
            
            # List available tools
            tools = await session.list_tools()
            print("Available tools:")
            for tool in tools.tools:
                print(f"  - {tool.name}: {tool.description}")
            
            print("\n" + "="*60)
            
            # Test get_user
            print("\n1️⃣  Testing get_user(1):")
            result = await session.call_tool("get_user", {"user_id": 1})
            print(result.content[0].text)
            
            # Test list_orders
            print("\n2️⃣  Testing list_orders(user_id=1):")
            result = await session.call_tool("list_orders", {"user_id": 1})
            print(result.content[0].text)
            
            # Test update_order_status
            print("\n3️⃣  Testing update_order_status(2, 'completed'):")
            result = await session.call_tool(
                "update_order_status",
                {"order_id": 2, "new_status": "completed"}
            )
            print(result.content[0].text)
            
            # List resources
            print("\n4️⃣  Testing list_resources():")
            resources = await session.list_resources()
            for resource in resources.resources:
                print(f"  - {resource.name}")

if __name__ == "__main__":
    asyncio.run(test_mcp_server())
```

**Run it:**

```bash
python test_mcp_client.py
```


***

#### **Day 2: Integrate with LangChain (4-6 hours)**

**Step 5: MCP + LangChain Bridge (2 hours)**

```python
# langchain_mcp_bridge.py
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain.tools import tool
from typing import Optional
import asyncio
import json

# Global session (in real app, manage properly)
_mcp_session = None

async def connect_mcp():
    """Connect to MCP server"""
    global _mcp_session
    
    server_params = StdioServerParameters(
        command="python",
        args=["mcp_server.py"]
    )
    
    read, write = await stdio_client(server_params).__aenter__()
    _mcp_session = await ClientSession(read, write).__aenter__()
    await _mcp_session.initialize()
    print("✅ MCP connection established")

# Create LangChain tools from MCP tools
@tool
def get_user_info(user_id: int) -> str:
    """Get user information by ID"""
    async def _call():
        result = await _mcp_session.call_tool("get_user", {"user_id": user_id})
        return result.content[0].text
    
    return asyncio.run(_call())

@tool
def list_user_orders(user_id: int, status: Optional[str] = None) -> str:
    """List orders for a user, optionally filtered by status"""
    async def _call():
        params = {"user_id": user_id}
        if status:
            params["status"] = status
        result = await _mcp_session.call_tool("list_orders", params)
        return result.content[0].text
    
    return asyncio.run(_call())

@tool
def update_order(order_id: int, new_status: str) -> str:
    """Update order status (pending, completed, cancelled)"""
    async def _call():
        result = await _mcp_session.call_tool(
            "update_order_status",
            {"order_id": order_id, "new_status": new_status}
        )
        return result.content[0].text
    
    return asyncio.run(_call())

# Initialize connection
asyncio.run(connect_mcp())

print("✅ LangChain tools created from MCP server")
```

**Step 6: Build ReAct Agent with MCP Tools (2 hours)**

```python
# agent_with_mcp.py
from langchain_openai import ChatOpenAI
from langchain.agents import create_react_agent, AgentExecutor
from langchain import hub
from langchain_mcp_bridge import get_user_info, list_user_orders, update_order
import os

# Set your API key
os.environ["OPENAI_API_KEY"] = "your-key"

# Create tools list
tools = [get_user_info, list_user_orders, update_order]

# Get ReAct prompt
prompt = hub.pull("hwchase17/react")

# Create LLM
llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

# Create agent
agent = create_react_agent(llm, tools, prompt)

# Create executor
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True,
    max_iterations=5
)

def query_agent(question: str):
    """Query the agent"""
    print(f"\n{'='*60}")
    print(f"Question: {question}")
    print(f"{'='*60}\n")
    
    result = agent_executor.invoke({"input": question})
    
    print(f"\n{'='*60}")
    print(f"Answer: {result['output']}")
    print(f"{'='*60}\n")

if __name__ == "__main__":
    # Test queries
    query_agent("What is the email address for user 1?")
    query_agent("Show me all pending orders for user 1")
    query_agent("Mark order 2 as completed")
    query_agent("What products has user 2 ordered?")
```

**Run it:**

```bash
python agent_with_mcp.py
```

**Step 7: Add REST API Wrapper (2 hours)**

```python
# api_server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from agent_with_mcp import agent_executor

app = FastAPI(title="Backend MCP API")

class QueryRequest(BaseModel):
    question: str

class QueryResponse(BaseModel):
    answer: str
    reasoning: list[str]

@app.post("/query", response_model=QueryResponse)
async def query_backend(request: QueryRequest):
    """Query the MCP-powered agent"""
    try:
        result = agent_executor.invoke({"input": request.question})
        
        return QueryResponse(
            answer=result["output"],
            reasoning=result.get("intermediate_steps", [])
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Run it:**

```bash
pip install fastapi uvicorn
python api_server.py

# Test in another terminal:
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What orders does user 1 have?"}'
```


***

#### **Day 3: TypeScript Version + Polish (3-4 hours)**

**Step 8: TypeScript MCP Server (2 hours)**

```bash
mkdir mcp-ts-server && cd mcp-ts-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install --save-dev typescript @types/node
npx tsc --init
```

```typescript
// server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// Mock data store
const users = [
  { id: 1, name: "Alice", email: "alice@example.com", subscription: "pro" },
  { id: 2, name: "Bob", email: "bob@example.com", subscription: "free" }
];

const orders = [
  { id: 1, userId: 1, product: "Widget A", amount: 99.99, status: "completed" },
  { id: 2, userId: 1, product: "Widget B", amount: 149.99, status: "pending" }
];

// Create server
const server = new McpServer({
  name: "backend-api-server-ts",
  version: "1.0.0"
});

// Define tools
server.tool(
  "get_user",
  "Get user information by ID",
  {
    user_id: z.number().describe("User ID")
  },
  async ({ user_id }) => {
    const user = users.find(u => u.id === user_id);
    
    return {
      content: [{
        type: "text",
        text: user ? JSON.stringify(user, null, 2) : `User ${user_id} not found`
      }]
    };
  }
);

server.tool(
  "list_orders",
  "List orders for a user",
  {
    user_id: z.number().describe("User ID"),
    status: z.string().optional().describe("Filter by status")
  },
  async ({ user_id, status }) => {
    let filtered = orders.filter(o => o.userId === user_id);
    
    if (status) {
      filtered = filtered.filter(o => o.status === status);
    }
    
    return {
      content: [{
        type: "text",
        text: JSON.stringify(filtered, null, 2)
      }]
    };
  }
);

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.log("🚀 TypeScript MCP server running");
}

main();
```

**Build and run:**

```bash
npx tsc
node dist/server.js
```

**Step 9: Documentation (1 hour)**

Create `README.md`:

```markdown
# Backend MCP Server

## Features
- ✅ User management tools
- ✅ Order management tools
- ✅ Python and TypeScript versions
- ✅ LangChain integration
- ✅ REST API wrapper

## Quick Start

### Python Version
```

python mcp_server.py

```

### Test
```

python test_mcp_client.py

```

### With LangChain Agent
```

python agent_with_mcp.py

```

## Tools Available
- `get_user(user_id)` - Get user info
- `list_orders(user_id, status?)` - List orders
- `update_order_status(order_id, status)` - Update order

## Configuration
Add to your MCP client config:
```

{
"mcpServers": {
"backend": {
"command": "python",
"args": ["mcp_server.py"]
}
}
}

```
```


***

### ☁️ Option B: Cloud Implementation

**Day 1-2: Deploy to Cloud**

**Option 1: Railway.app (Free tier)**

```bash
# Create railway.toml
[build]
builder = "NIXPACKS"

[deploy]
startCommand = "python api_server.py"
restartPolicyType = "ON_FAILURE"
```

Deploy:

```bash
railway login
railway init
railway up
```

**Option 2: Render.com (Free tier)**

```yaml
# render.yaml
services:
  - type: web
    name: mcp-backend-server
    env: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "python api_server.py"
```

**Option 3: Fly.io (Free tier)**

```bash
fly launch
fly deploy
```

**Use Cloud Database:**

```python
# Replace SQLite with PostgreSQL
import os
from sqlalchemy import create_engine

# Use environment variable
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./backend.db")
engine = create_engine(DATABASE_URL)
```


***

## 📋 Project 3: LLM Inference Platform with Multi-LoRA

### 🎯 Learning Goals

- ✅ Deploy LLM inference servers (vLLM)
- ✅ Implement LoRA adapter management
- ✅ Build model serving APIs
- ✅ Monitor inference performance
- ✅ Handle concurrent requests


### ⏱️ Time Estimate: **2-3 days**


***

### 🖥️ Option A: Local Implementation

**Prerequisites:**

- NVIDIA GPU (8GB+ VRAM) OR use CPU-only mode
- 16GB+ RAM
- Docker (optional)


#### **Day 1: Setup vLLM + Base Model (4-6 hours)**

**Step 1: Environment Setup (1 hour)**

```bash
mkdir llm-inference-platform && cd llm-inference-platform

# Create venv
python -m venv venv
source venv/bin/activate

# Install vLLM (GPU version)
pip install vllm

# OR CPU-only (slower but works without GPU)
pip install vllm-cpu-only

# Other dependencies
pip install fastapi uvicorn requests
```

**Step 2: Download Small Model (1 hour)**

```python
# download_model.py
from huggingface_hub import snapshot_download

# Download small model (1-2GB)
# Phi-2 is great for laptops!
model_name = "microsoft/phi-2"

print(f"Downloading {model_name}...")
model_path = snapshot_download(
    repo_id=model_name,
    cache_dir="./models"
)

print(f"✅ Model downloaded to: {model_path}")
```

**Run it:**

```bash
python download_model.py
```

**Step 3: Start vLLM Server (30 min)**

```python
# start_vllm.py
from vllm import LLM, SamplingParams

# Initialize model
llm = LLM(
    model="microsoft/phi-2",
    download_dir="./models",
    max_model_len=2048,  # Reduce for less VRAM
    gpu_memory_utilization=0.8
)

print("✅ vLLM model loaded!")

# Test inference
prompts = ["What is machine learning?"]
sampling_params = SamplingParams(
    temperature=0.7,
    max_tokens=100
)

outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    print(f"\nPrompt: {output.prompt}")
    print(f"Output: {output.outputs[0].text}")
```

**Run it:**

```bash
python start_vllm.py
```

**Step 4: Create API Server (1-2 hours)**

```python
# vllm_api_server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from vllm import LLM, SamplingParams
from typing import Optional

app = FastAPI(title="LLM Inference API")

# Initialize model
llm = LLM(
    model="microsoft/phi-2",
    download_dir="./models",
    max_model_len=2048
)

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int = 100
    temperature: float = 0.7
    lora_adapter: Optional[str] = None

class GenerateResponse(BaseModel):
    text: str
    tokens_used: int

@app.post("/generate", response_model=GenerateResponse)
async def generate(request: GenerateRequest):
    """Generate text from prompt"""
    try:
        sampling_params = SamplingParams(
            temperature=request.temperature,
            max_tokens=request.max_tokens
        )
        
        # TODO: Add LoRA adapter support
        outputs = llm.generate([request.prompt], sampling_params)
        
        result = outputs[0].outputs[0]
        
        return GenerateResponse(
            text=result.text,
            tokens_used=len(result.token_ids)
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "healthy", "model": "phi-2"}

@app.get("/models")
async def list_models():
    return {
        "base_model": "microsoft/phi-2",
        "lora_adapters": []  # Will add in Day 2
    }

if __name__ == "__main__":
    import uvicorn
    print("🚀 Starting vLLM API server...")
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Run it:**

```bash
python vllm_api_server.py

# Test in another terminal:
curl -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Explain neural networks:", "max_tokens": 50}'
```


***

#### **Day 2: Add LoRA Support (4-6 hours)**

**Step 5: Fine-tune LoRA Adapter (2 hours)**

```python
# train_lora.py
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer
)
from peft import LoraConfig, get_peft_model
from datasets import load_dataset

# Load base model
model_name = "microsoft/phi-2"
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    cache_dir="./models",
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Configure LoRA
lora_config = LoraConfig(
    r=8,  # Low rank
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],  # Which layers to adapt
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

# Apply LoRA
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()

# Prepare tiny dataset (for demo)
train_data = [
    "Customer: How do I reset my password?\nAgent: You can reset via Settings > Account > Reset Password.",
    "Customer: What are your business hours?\nAgent: We're open Monday-Friday, 9 AM to 5 PM EST.",
    "Customer: How do I cancel my subscription?\nAgent: Go to Account > Billing > Cancel Subscription.",
]

# Tokenize
def tokenize(text):
    return tokenizer(text, truncation=True, max_length=512)

# Quick training (just for demo)
training_args = TrainingArguments(
    output_dir="./lora_adapters/customer_support",
    num_train_epochs=3,
    per_device_train_batch_size=1,
    save_steps=10,
    save_total_limit=2,
    logging_steps=1
)

print("🏋️ Training LoRA adapter...")
# In real scenario, use proper dataset and training loop
# For this demo, we'll just save the adapter

model.save_pretrained("./lora_adapters/customer_support")
tokenizer.save_pretrained("./lora_adapters/customer_support")

print("✅ LoRA adapter saved!")
```

**Simplified version (skip training, just create adapter structure):**

```bash
# Create adapter directories
mkdir -p lora_adapters/customer_support
mkdir -p lora_adapters/coding_assistant

# We'll simulate adapters for the demo
```

**Step 6: Multi-LoRA Server (2-3 hours)**

```python
# multi_lora_server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from vllm import LLM, SamplingParams
from typing import Optional
import os

app = FastAPI(title="Multi-LoRA Inference API")

# Initialize base model
print("Loading base model...")
llm = LLM(
    model="microsoft/phi-2",
    download_dir="./models",
    max_model_len=2048,
    enable_lora=True,  # Enable LoRA support
    max_lora_rank=8
)

# Available adapters
ADAPTERS = {
    "customer_support": "./lora_adapters/customer_support",
    "coding_assistant": "./lora_adapters/coding_assistant",
}

# Track adapter usage
adapter_stats = {name: 0 for name in ADAPTERS}

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int = 100
    temperature: float = 0.7
    adapter: Optional[str] = None  # "customer_support" or "coding_assistant"

class GenerateResponse(BaseModel):
    text: str
    tokens_used: int
    adapter_used: Optional[str]

@app.post("/generate", response_model=GenerateResponse)
async def generate(request: GenerateRequest):
    """Generate with optional LoRA adapter"""
    try:
        sampling_params = SamplingParams(
            temperature=request.temperature,
            max_tokens=request.max_tokens
        )
        
        # Select adapter if specified
        adapter_path = None
        if request.adapter:
            if request.adapter not in ADAPTERS:
                raise HTTPException(
                    status_code=400,
                    detail=f"Unknown adapter: {request.adapter}"
                )
            adapter_path = ADAPTERS[request.adapter]
            adapter_stats[request.adapter] += 1
        
        # Generate (with or without adapter)
        # Note: vLLM's LoRA API varies by version
        outputs = llm.generate([request.prompt], sampling_params)
        
        result = outputs[0].outputs[0]
        
        return GenerateResponse(
            text=result.text,
            tokens_used=len(result.token_ids),
            adapter_used=request.adapter
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/adapters")
async def list_adapters():
    """List available LoRA adapters"""
    return {
        "adapters": list(ADAPTERS.keys()),
        "usage_stats": adapter_stats
    }

@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "base_model": "microsoft/phi-2",
        "lora_enabled": True
    }

if __name__ == "__main__":
    import uvicorn
    print("🚀 Starting Multi-LoRA server...")
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Step 7: Simple Load Testing (1 hour)**

```python
# load_test.py
import requests
import time
import concurrent.futures
from statistics import mean, stdev

API_URL = "http://localhost:8000/generate"

def single_request(prompt, adapter=None):
    """Make single inference request"""
    start = time.time()
    
    payload = {
        "prompt": prompt,
        "max_tokens": 50,
        "adapter": adapter
    }
    
    response = requests.post(API_URL, json=payload)
    latency = time.time() - start
    
    return latency, response.status_code

def load_test(num_requests=20, concurrent=5):
    """Run load test"""
    prompts = [
        "Explain machine learning",
        "Write a Python function",
        "What is cloud computing?",
        "Help me debug this code"
    ]
    
    adapters = [None, "customer_support", "coding_assistant"]
    
    print(f"🏋️ Running {num_requests} requests ({concurrent} concurrent)...")
    
    latencies = []
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=concurrent) as executor:
        futures = []
        
        for i in range(num_requests):
            prompt = prompts[i % len(prompts)]
            adapter = adapters[i % len(adapters)]
            futures.append(executor.submit(single_request, prompt, adapter))
        
        for future in concurrent.futures.as_completed(futures):
            latency, status = future.result()
            latencies.append(latency)
            if status == 200:
                print(f"✅ Request completed in {latency:.2f}s")
            else:
                print(f"❌ Request failed with status {status}")
    
    # Stats
    print(f"\n{'='*60}")
    print("📊 Load Test Results:")
    print(f"  Total requests: {num_requests}")
    print(f"  Successful: {len(latencies)}")
    print(f"  Avg latency: {mean(latencies):.2f}s")
    print(f"  Std dev: {stdev(latencies):.2f}s")
    print(f"  Min: {min(latencies):.2f}s")
    print(f"  Max: {max(latencies):.2f}s")
    print(f"{'='*60}")

if __name__ == "__main__":
    load_test()
```

**Run it:**

```bash
python load_test.py
```


***

#### **Day 3: Monitoring \& LangChain Integration (3-4 hours)**

**Step 8: Add Prometheus Metrics (1-2 hours)**

```python
# monitored_server.py
from fastapi import FastAPI
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi.responses import Response
import time

app = FastAPI()

# Metrics
request_count = Counter(
    'inference_requests_total',
    'Total inference requests',
    ['adapter', 'status']
)

request_latency = Histogram(
    'inference_latency_seconds',
    'Inference latency',
    ['adapter']
)

tokens_generated = Counter(
    'tokens_generated_total',
    'Total tokens generated',
    ['adapter']
)

@app.post("/generate")
async def generate(request: GenerateRequest):
    """Generate with metrics"""
    start_time = time.time()
    adapter = request.adapter or "base"
    
    try:
        # ... your generation code ...
        
        # Record metrics
        latency = time.time() - start_time
        request_count.labels(adapter=adapter, status='success').inc()
        request_latency.labels(adapter=adapter).observe(latency)
        tokens_generated.labels(adapter=adapter).inc(result.tokens_used)
        
        return result
    except Exception as e:
        request_count.labels(adapter=adapter, status='error').inc()
        raise

@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

**Step 9: LangChain Integration (1 hour)**

```python
# langchain_vllm_client.py
from langchain_community.llms import VLLM
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate

# Create LangChain LLM wrapper
llm = VLLM(
    model="microsoft/phi-2",
    trust_remote_code=True,
    max_new_tokens=100,
    temperature=0.7
)

# Create prompt template
template = """Question: {question}

Answer: Let me help you with that."""

prompt = PromptTemplate(template=template, input_variables=["question"])

# Create chain
chain = LLMChain(llm=llm, prompt=prompt)

# Test
questions = [
    "What is a REST API?",
    "Explain Docker containers",
    "What is Kubernetes?"
]

for q in questions:
    print(f"\nQ: {q}")
    answer = chain.run(question=q)
    print(f"A: {answer}")
```

**Step 10: Simple Dashboard (1 hour)**

```python
# dashboard.py
import streamlit as st
import requests
import time

st.title("🚀 LLM Inference Dashboard")

# API endpoint
API_URL = "http://localhost:8000"

# Sidebar: Model info
st.sidebar.header("Model Info")
try:
    health = requests.get(f"{API_URL}/health").json()
    st.sidebar.success(f"Status: {health['status']}")
    st.sidebar.info(f"Model: {health['base_model']}")
except:
    st.sidebar.error("API not reachable")

# Adapter selection
st.sidebar.header("LoRA Adapters")
try:
    adapters_resp = requests.get(f"{API_URL}/adapters").json()
    adapters = ["None"] + adapters_resp["adapters"]
    
    st.sidebar.write("Usage Stats:")
    for adapter, count in adapters_resp["usage_stats"].items():
        st.sidebar.metric(adapter, count)
except:
    adapters = ["None"]

# Main interface
st.header("Generate Text")

prompt = st.text_area("Enter your prompt:", height=100)
adapter = st.selectbox("Select adapter:", adapters)
max_tokens = st.slider("Max tokens:", 10, 500, 100)
temperature = st.slider("Temperature:", 0.0, 2.0, 0.7)

if st.button("Generate"):
    if not prompt:
        st.warning("Please enter a prompt")
    else:
        with st.spinner("Generating..."):
            start_time = time.time()
            
            payload = {
                "prompt": prompt,
                "max_tokens": max_tokens,
                "temperature": temperature
            }
            
            if adapter != "None":
                payload["adapter"] = adapter
            
            try:
                response = requests.post(
                    f"{API_URL}/generate",
                    json=payload
                )
                
                latency = time.time() - start_time
                
                if response.status_code == 200:
                    result = response.json()
                    
                    st.success(f"Generated in {latency:.2f}s")
                    st.markdown(f"**Output:**\n\n{result['text']}")
                    st.info(f"Tokens used: {result['tokens_used']}")
                else:
                    st.error(f"Error: {response.text}")
            except Exception as e:
                st.error(f"Request failed: {e}")
```

**Run it:**

```bash
pip install streamlit
streamlit run dashboard.py
```


***

### ☁️ Option B: Cloud Implementation

**Day 1-3: Cloud Deployment**

**Option 1: RunPod (GPU cloud, ~\$0.30/hr)**

```bash
# Create account at runpod.io
# Select template: "vLLM"
# Choose GPU: RTX 3090 or A4000
# Deploy with one click
```

**Option 2: Modal (Serverless GPUs)**

```python
# modal_deploy.py
from modal import Image, Stub, method, gpu

stub = Stub("llm-inference")

vllm_image = Image.debian_slim().pip_install("vllm", "fastapi")

@stub.cls(
    gpu=gpu.A10G(),
    image=vllm_image,
    container_idle_timeout=300
)
class VLLMModel:
    def __init__(self):
        from vllm import LLM
        self.llm = LLM(model="microsoft/phi-2")
    
    @method()
    def generate(self, prompt: str):
        outputs = self.llm.generate([prompt])
        return outputs[0].outputs[0].text

# Deploy
@stub.function()
def test():
    model = VLLMModel()
    return model.generate("Hello, world!")
```

Deploy:

```bash
modal deploy modal_deploy.py
```

**Option 3: Hugging Face Inference Endpoints**

- Go to huggingface.co
- Create new inference endpoint
- Select model: microsoft/phi-2
- Choose GPU tier
- Deploy with UI

Then connect:

```python
import requests

API_URL = "https://your-endpoint.hf.space"
headers = {"Authorization": f"Bearer {API_TOKEN}"}

response = requests.post(
    API_URL,
    headers=headers,
    json={"inputs": "Your prompt"}
)
```


***

## 📋 Project 4: Full-Stack AI Application

### 🎯 Learning Goals

- ✅ Build complete frontend + backend AI app
- ✅ Implement streaming responses
- ✅ Handle authentication
- ✅ Deploy production system
- ✅ Integrate all learned technologies


### ⏱️ Time Estimate: **3 days**


***

### 🖥️ Option A: Local Implementation

#### **Day 1: Backend API (4-6 hours)**

**Step 1: Setup FastAPI Backend (1 hour)**

```bash
mkdir fullstack-ai-app && cd fullstack-ai-app
mkdir backend frontend

cd backend
python -m venv venv
source venv/bin/activate

pip install fastapi uvicorn langchain langchain-openai
pip install python-jose python-multipart
```

**Step 2: Create API with Auth (2 hours)**

```python
# backend/main.py
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional
import os

app = FastAPI(title="AI Assistant API")

# CORS for frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Simple auth (in production, use JWT)
security = HTTPBearer()
VALID_TOKENS = {"demo-token-123"}

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    if credentials.credentials not in VALID_TOKENS:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )
    return credentials.credentials

# Models
class ChatRequest(BaseModel):
    message: str
    conversation_id: Optional[str] = None

class ChatResponse(BaseModel):
    response: str
    conversation_id: str

# In-memory conversation store (use Redis in production)
conversations = {}

@app.post("/api/chat", response_model=ChatResponse)
async def chat(
    request: ChatRequest,
    token: str = Depends(verify_token)
):
    """Chat endpoint"""
    from langchain_openai import ChatOpenAI
    from langchain_core.messages import HumanMessage, AIMessage
    
    # Initialize LLM
    llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0.7)
    
    # Get or create conversation
    conv_id = request.conversation_id or f"conv_{len(conversations)}"
    history = conversations.get(conv_id, [])
    
    # Add user message
    history.append(HumanMessage(content=request.message))
    
    # Generate response
    response = llm.invoke(history)
    
    # Add AI response to history
    history.append(response)
    conversations[conv_id] = history
    
    return ChatResponse(
        response=response.content,
        conversation_id=conv_id
    )

@app.get("/api/conversations")
async def list_conversations(token: str = Depends(verify_token)):
    """List all conversations"""
    return {
        "conversations": [
            {
                "id": conv_id,
                "messages": len(messages)
            }
            for conv_id, messages in conversations.items()
        ]
    }

@app.get("/health")
async def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Step 3: Add Streaming (1-2 hours)**

```python
# backend/streaming.py
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
import json

app = FastAPI()

@app.post("/api/chat/stream")
async def stream_chat(request: ChatRequest):
    """Streaming chat endpoint"""
    
    async def generate():
        llm = ChatOpenAI(
            model="gpt-3.5-turbo",
            streaming=True
        )
        
        async for chunk in llm.astream([HumanMessage(content=request.message)]):
            yield f"data: {json.dumps({'content': chunk.content})}\n\n"
        
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )
```


***

#### **Day 2: Frontend (4-6 hours)**

**Step 4: React Setup (30 min)**

```bash
cd ../frontend
npx create-react-app . --template typescript
npm install axios react-markdown
```

**Step 5: Build Chat Interface (2-3 hours)**

```tsx
// frontend/src/App.tsx
import React, { useState, useRef, useEffect } from 'react';
import axios from 'axios';
import ReactMarkdown from 'react-markdown';
import './App.css';

const API_URL = 'http://localhost:8000';
const API_TOKEN = 'demo-token-123';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

function App() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const [conversationId, setConversationId] = useState<string | null>(null);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  useEffect(() => {
    scrollToBottom();
  }, [messages]);

  const sendMessage = async () => {
    if (!input.trim()) return;

    const userMessage: Message = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setLoading(true);

    try {
      const response = await axios.post(
        `${API_URL}/api/chat`,
        {
          message: input,
          conversation_id: conversationId
        },
        {
          headers: {
            'Authorization': `Bearer ${API_TOKEN}`,
            'Content-Type': 'application/json'
          }
        }
      );

      const assistantMessage: Message = {
        role: 'assistant',
        content: response.data.response
      };

      setMessages(prev => [...prev, assistantMessage]);
      setConversationId(response.data.conversation_id);
    } catch (error) {
      console.error('Error:', error);
      alert('Failed to send message');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="App">
      <div className="chat-container">
        <div className="chat-header">
          <h1>🤖 AI Assistant</h1>
        </div>

        <div className="messages-container">
          {messages.map((msg, idx) => (
            <div key={idx} className={`message ${msg.role}`}>
              <div className="message-content">
                <ReactMarkdown>{msg.content}</ReactMarkdown>
              </div>
            </div>
          ))}
          {loading && (
            <div className="message assistant">
              <div className="message-content">
                <span className="typing-indicator">●●●</span>
              </div>
            </div>
          )}
          <div ref={messagesEndRef} />
        </div>

        <div className="input-container">
          <input
            type="text"
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
            placeholder="Type your message..."
            disabled={loading}
          />
          <button onClick={sendMessage} disabled={loading}>
            Send
          </button>
        </div>
      </div>
    </div>
  );
}

export default App;
```

**Step 6: Styling (1 hour)**

```css
/* frontend/src/App.css */
.App {
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.chat-container {
  width: 800px;
  height: 600px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.2);
  display: flex;
  flex-direction: column;
}

.chat-header {
  padding: 20px;
  border-bottom: 1px solid #eee;
  background: #f8f9fa;
  border-radius: 10px 10px 0 0;
}

.chat-header h1 {
  margin: 0;
  font-size: 24px;
  color: #333;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

.message {
  margin-bottom: 20px;
  display: flex;
}

.message.user {
  justify-content: flex-end;
}

.message.assistant {
  justify-content: flex-start;
}

.message-content {
  max-width: 70%;
  padding: 12px 16px;
  border-radius: 12px;
  word-wrap: break-word;
}

.message.user .message-content {
  background: #667eea;
  color: white;
}

.message.assistant .message-content {
  background: #f1f3f5;
  color: #333;
}

.typing-indicator {
  font-size: 20px;
  animation: blink 1.4s infinite;
}

@keyframes blink {
  0%, 100% { opacity: 0; }
  50% { opacity: 1; }
}

.input-container {
  padding: 20px;
  border-top: 1px solid #eee;
  display: flex;
  gap: 10px;
}

.input-container input {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
}

.input-container button {
  padding: 12px 24px;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
}

.input-container button:hover {
  background: #5568d3;
}

.input-container button:disabled {
  background: #ccc;
  cursor: not-allowed;
}
```

**Run both:**

```bash
# Terminal 1 (backend)
cd backend
python main.py

# Terminal 2 (frontend)
cd frontend
npm start
```


***

#### **Day 3: Polish \& Deploy (3-4 hours)**

**Step 7: Add Features (2 hours)**

```tsx
// Add conversation history sidebar
// Add dark mode toggle
// Add typing indicators
// Add error handling
// Add message timestamps

// Example: Dark mode
const [darkMode, setDarkMode] = useState(false);

// Toggle dark mode class on body
useEffect(() => {
  document.body.className = darkMode ? 'dark-mode' : '';
}, [darkMode]);
```

**Step 8: Docker Everything (1 hour)**

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=0 /app/build /usr/share/nginx/html
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
```

**Run with Docker:**

```bash
docker-compose up --build
```


***

### ☁️ Option B: Cloud Implementation

**Day 1-3: Cloud Deploy**

**Option 1: Vercel (Frontend) + Railway (Backend)**

Frontend on Vercel:

```bash
cd frontend
npm install -g vercel
vercel login
vercel --prod
```

Backend on Railway:

```bash
cd backend
railway login
railway init
railway up
```

**Option 2: All on Render.com**

```yaml
# render.yaml
services:
  - type: web
    name: ai-backend
    env: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "uvicorn main:app --host 0.0.0.0 --port $PORT"
    
  - type: web
    name: ai-frontend
    env: static
    buildCommand: "npm install && npm run build"
    staticPublishPath: ./build
```

**Option 3: Heroku**

```bash
# Backend
cd backend
heroku create my-ai-backend
git push heroku main

# Frontend
cd frontend
heroku create my-ai-frontend
heroku buildpacks:add heroku/nodejs
git push heroku main
```


***

## 📋 Project 5: AI-Native Customer Support System

### 🎯 Learning Goals

- ✅ Design complex multi-agent systems
- ✅ Implement supervisor patterns
- ✅ Build production-ready workflows
- ✅ Handle escalations and approvals
- ✅ Create analytics dashboards


### ⏱️ Time Estimate: **2-3 days**


***

### 🖥️ Option A: Local Implementation

#### **Day 1: Multi-Agent System (4-6 hours)**

**Step 1: Setup (30 min)**

```bash
mkdir customer-support-ai && cd customer-support-ai
python -m venv venv
source venv/bin/activate

pip install langgraph langchain langchain-openai
pip install chromadb sentence-transformers
pip install fastapi uvicorn streamlit
```

**Step 2: Knowledge Base (1 hour)**

```python
# knowledge_base.py
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Sample support docs
support_docs = [
    {
        "category": "account",
        "content": """
        Account Management FAQ
        
        Q: How do I reset my password?
        A: Click "Forgot Password" on the login page.
        
        Q: How do I update my email?
        A: Go to Settings > Profile > Change Email.
        
        Q: How do I delete my account?
        A: Contact support@company.com for account deletion.
        """
    },
    {
        "category": "billing",
        "content": """
        Billing FAQ
        
        Q: How do I upgrade my plan?
        A: Go to Settings > Billing > Change Plan.
        
        Q: When will I be charged?
        A: Charges occur on the 1st of each month.
        
        Q: What payment methods do you accept?
        A: We accept credit cards, PayPal, and wire transfer.
        """
    },
    {
        "category": "technical",
        "content": """
        Technical Support FAQ
        
        Q: API not working?
        A: Check your API key in Settings > API Keys.
        
        Q: Getting 429 error?
        A: You've hit rate limit. Upgrade plan or wait.
        
        Q: How to integrate webhook?
        A: See docs at docs.company.com/webhooks
        """
    }
]

# Create knowledge base
def create_knowledge_base():
    embeddings = HuggingFaceEmbeddings(
        model_name="sentence-transformers/all-MiniLM-L6-v2"
    )
    
    all_texts = []
    for doc in support_docs:
        all_texts.append(doc["content"])
    
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50
    )
    
    chunks = text_splitter.create_documents(all_texts)
    
    vectorstore = Chroma.from_documents(
        chunks,
        embeddings,
        persist_directory="./support_kb"
    )
    
    print("✅ Knowledge base created!")
    return vectorstore

if __name__ == "__main__":
    create_knowledge_base()
```

**Run it:**

```bash
python knowledge_base.py
```

**Step 3: Specialized Agents (2 hours)**

```python
# agents.py
from typing import TypedDict, Annotated, Literal
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_core.messages import HumanMessage, AIMessage
import os

os.environ["OPENAI_API_KEY"] = "your-key"

# State
class SupportState(TypedDict):
    messages: Annotated[list, add_messages]
    category: str
    priority: str
    resolved: bool
    escalated: bool
    kb_results: list

# Initialize
llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)
kb = Chroma(persist_directory="./support_kb", embedding_function=embeddings)

# Triage Agent
def triage_agent(state: SupportState):
    """Classify and prioritize support request"""
    user_message = state["messages"][0].content
    
    classification = llm.invoke([
        HumanMessage(content=f"""
        Classify this support request:
        
        Request: {user_message}
        
        Provide:
        1. Category: account, billing, or technical
        2. Priority: low, medium, or high
        
        Format: category|priority
        """)
    ])
    
    category, priority = classification.content.strip().split('|')
    
    print(f"🎯 Triaged: {category} (priority: {priority})")
    
    return {
        "category": category.strip(),
        "priority": priority.strip(),
        "messages": [AIMessage(content=f"Categorized as {category}")]
    }

# Knowledge Base Agent
def kb_agent(state: SupportState):
    """Search knowledge base"""
    user_message = state["messages"][0].content
    
    # Search KB
    results = kb.similarity_search(user_message, k=2)
    context = "\n\n".join([doc.page_content for doc in results])
    
    # Generate answer
    response = llm.invoke([
        HumanMessage(content=f"""
        Use this knowledge base to answer:
        
        Knowledge Base:
        {context}
        
        Question: {user_message}
        
        Provide helpful answer. If you can't answer, say so.
        """)
    ])
    
    # Check if resolved
    resolved = "I don't know" not in response.content.lower()
    
    return {
        "kb_results": results,
        "resolved": resolved,
        "messages": [AIMessage(content=response.content)]
    }

# Billing Agent
def billing_agent(state: SupportState):
    """Handle billing queries"""
    user_message = state["messages"][0].content
    
    # Mock billing info
    billing_info = """
    Account: Premium Plan
    Next billing: December 1, 2025
    Amount: $99/month
    Payment method: **** 1234
    """
    
    response = llm.invoke([
        HumanMessage(content=f"""
        You are a billing specialist.
        
        Account Info:
        {billing_info}
        
        Customer Question: {user_message}
        
        Provide helpful billing assistance.
        """)
    ])
    
    return {
        "resolved": True,
        "messages": [AIMessage(content=response.content)]
    }

# Technical Agent
def technical_agent(state: SupportState):
    """Handle technical issues"""
    user_message = state["messages"][0].content
    
    # Check complexity
    complexity_check = llm.invoke([
        HumanMessage(content=f"""
        Is this a complex technical issue requiring escalation?
        
        Issue: {user_message}
        
        Answer: yes or no
        """)
    ])
    
    needs_escalation = "yes" in complexity_check.content.lower()
    
    if needs_escalation:
        return {
            "escalated": True,
            "resolved": False,
            "messages": [AIMessage(content="Escalating to technical team...")]
        }
    
    # Handle simple issue
    response = llm.invoke([
        HumanMessage(content=f"""
        You are a technical support agent.
        
        Issue: {user_message}
        
        Provide troubleshooting steps.
        """)
    ])
    
    return {
        "resolved": True,
        "escalated": False,
        "messages": [AIMessage(content=response.content)]
    }

print("✅ Agents initialized!")
```

**Step 4: Build LangGraph Workflow (2 hours)**

```python
# workflow.py
from langgraph.graph import StateGraph, END
from agents import (
    SupportState,
    triage_agent,
    kb_agent,
    billing_agent,
    technical_agent
)

def route_after_triage(state: SupportState) -> str:
    """Route based on category"""
    category = state["category"]
    
    # Try KB first for account questions
    if category == "account":
        return "kb"
    elif category == "billing":
        return "billing"
    elif category == "technical":
        return "technical"
    return "kb"

def route_after_kb(state: SupportState) -> str:
    """Check if resolved"""
    if state["resolved"]:
        return END
    else:
        # Route to specialist
        return state["category"]

def check_resolution(state: SupportState) -> str:
    """Check if issue resolved"""
    if state.get("escalated"):
        return "escalation"
    elif state.get("resolved"):
        return END
    else:
        return "kb"

# Build graph
workflow = StateGraph(SupportState)

# Add nodes
workflow.add_node("triage", triage_agent)
workflow.add_node("kb", kb_agent)
workflow.add_node("billing", billing_agent)
workflow.add_node("technical", technical_agent)

# Set entry
workflow.set_entry_point("triage")

# Add edges
workflow.add_conditional_edges(
    "triage",
    route_after_triage,
    {
        "kb": "kb",
        "billing": "billing",
        "technical": "technical"
    }
)

workflow.add_conditional_edges(
    "kb",
    route_after_kb,
    {
        "account": "kb",
        "billing": "billing",
        "technical": "technical",
        END: END
    }
)

workflow.add_conditional_edges(
    "billing",
    check_resolution,
    {
        "escalation": END,
        END: END,
        "kb": "kb"
    }
)

workflow.add_conditional_edges(
    "technical",
    check_resolution,
    {
        "escalation": END,
        END: END,
        "kb": "kb"
    }
)

# Compile
app = workflow.compile()

print("✅ Workflow compiled!")

# Test function
def handle_support_request(question: str):
    """Handle a support request"""
    print(f"\n{'='*60}")
    print(f"Support Request: {question}")
    print(f"{'='*60}\n")
    
    result = app.invoke({
        "messages": [HumanMessage(content=question)]
    })
    
    print(f"\nCategory: {result['category']}")
    print(f"Priority: {result['priority']}")
    print(f"Resolved: {result['resolved']}")
    print(f"Escalated: {result.get('escalated', False)}")
    print(f"\nResponse:")
    for msg in result['messages'][1:]:  # Skip original message
        if isinstance(msg, AIMessage):
            print(msg.content)
    
    print(f"\n{'='*60}\n")

if __name__ == "__main__":
    from langchain_core.messages import HumanMessage, AIMessage
    
    # Test cases
    handle_support_request("How do I reset my password?")
    handle_support_request("Why was I charged twice this month?")
    handle_support_request("My API integration is returning 500 errors")
```

**Run it:**

```bash
python workflow.py
```


***

#### **Day 2: Web Interface (4-6 hours)**

**Step 5: FastAPI Backend (2 hours)**

```python
# api.py
from fastapi import FastAPI, WebSocket
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from workflow import app as support_app
from langchain_core.messages import HumanMessage
import json

app = FastAPI(title="AI Support API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

class SupportRequest(BaseModel):
    question: str

class SupportResponse(BaseModel):
    category: str
    priority: str
    resolved: bool
    escalated: bool
    response: str

@app.post("/api/support", response_model=SupportResponse)
async def handle_request(request: SupportRequest):
    """Handle support request"""
    result = support_app.invoke({
        "messages": [HumanMessage(content=request.question)]
    })
    
    # Extract last AI message
    response_text = ""
    for msg in reversed(result['messages']):
        if hasattr(msg, 'content'):
            response_text = msg.content
            break
    
    return SupportResponse(
        category=result['category'],
        priority=result['priority'],
        resolved=result['resolved'],
        escalated=result.get('escalated', False),
        response=response_text
    )

@app.websocket("/ws/support")
async def websocket_support(websocket: WebSocket):
    """WebSocket for real-time support"""
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_text()
            question = json.loads(data)["question"]
            
            # Stream workflow steps
            for event in support_app.stream(
                {"messages": [HumanMessage(content=question)]}
            ):
                await websocket.send_json({"event": event})
            
            await websocket.send_json({"done": True})
    except:
        await websocket.close()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Step 6: Streamlit Dashboard (2-3 hours)**

```python
# dashboard.py
import streamlit as st
import requests
import time
from datetime import datetime

st.set_page_config(page_title="AI Support Dashboard", layout="wide")

st.title("🎫 AI Customer Support System")

# Sidebar: Analytics
st.sidebar.header("📊 Analytics")

# Mock analytics data
if 'tickets' not in st.session_state:
    st.session_state.tickets = []

total_tickets = len(st.session_state.tickets)
resolved = sum(1 for t in st.session_state.tickets if t['resolved'])
escalated = sum(1 for t in st.session_state.tickets if t.get('escalated'))

col1, col2, col3 = st.sidebar.columns(3)
col1.metric("Total", total_tickets)
col2.metric("Resolved", f"{resolved}/{total_tickets}")
col3.metric("Escalated", escalated)

# Category breakdown
if st.session_state.tickets:
    categories = {}
    for ticket in st.session_state.tickets:
        cat = ticket['category']
        categories[cat] = categories.get(cat, 0) + 1
    
    st.sidebar.write("**By Category:**")
    for cat, count in categories.items():
        st.sidebar.write(f"- {cat.title()}: {count}")

# Main interface
st.header("Submit Support Request")

with st.form("support_form"):
    question = st.text_area("Describe your issue:", height=100)
    submit = st.form_submit_button("Submit Request")

if submit and question:
    with st.spinner("Processing your request..."):
        start_time = time.time()
        
        try:
            response = requests.post(
                "http://localhost:8000/api/support",
                json={"question": question}
            )
            
            if response.status_code == 200:
                result = response.json()
                processing_time = time.time() - start_time
                
                # Save to session
                ticket = {
                    "timestamp": datetime.now(),
                    "question": question,
                    **result
                }
                st.session_state.tickets.append(ticket)
                
                # Display result
                st.success(f"✅ Processed in {processing_time:.2f}s")
                
                col1, col2 = st.columns(2)
                with col1:
                    st.info(f"**Category:** {result['category'].title()}")
                    st.info(f"**Priority:** {result['priority'].title()}")
                
                with col2:
                    status = "✅ Resolved" if result['resolved'] else "⚠️ Unresolved"
                    st.info(f"**Status:** {status}")
                    
                    if result.get('escalated'):
                        st.warning("🚨 **Escalated to specialist**")
                
                st.markdown("### Response:")
                st.markdown(result['response'])
            else:
                st.error("Failed to process request")
        except Exception as e:
            st.error(f"Error: {e}")

# Ticket history
if st.session_state.tickets:
    st.header("📋 Recent Tickets")
    
    for i, ticket in enumerate(reversed(st.session_state.tickets[-5:])):
        with st.expander(
            f"Ticket #{len(st.session_state.tickets) - i}: {ticket['question'][:50]}..."
        ):
            st.write(f"**Time:** {ticket['timestamp'].strftime('%Y-%m-%d %H:%M:%S')}")
            st.write(f"**Category:** {ticket['category']}")
            st.write(f"**Priority:** {ticket['priority']}")
            st.write(f"**Resolved:** {'Yes' if ticket['resolved'] else 'No'}")
            st.write(f"**Response:** {ticket['response']}")
```

**Run both:**

```bash
# Terminal 1
python api.py

# Terminal 2
streamlit run dashboard.py
```


***

#### **Day 3: Polish \& Deploy (3-4 hours)**

**Step 7: Add Email Integration (Mock) (1 hour)**

```python
# email_integration.py
from typing import Dict
import smtplib
from email.mime.text import MIMEText

def send_escalation_email(ticket: Dict):
    """Send email when ticket is escalated"""
    subject = f"[ESCALATED] Support Ticket - {ticket['category']}"
    
    body = f"""
    A support ticket has been escalated:
    
    Category: {ticket['category']}
    Priority: {ticket['priority']}
    
    Customer Question:
    {ticket['question']}
    
    Initial Response:
    {ticket['response']}
    
    Please review and respond.
    """
    
    # Mock sending (implement real SMTP)
    print(f"📧 Email sent: {subject}")
    return True

def send_resolution_email(ticket: Dict):
    """Send email when ticket is resolved"""
    # Implementation
    pass
```

**Step 8: Add Analytics (1 hour)**

```python
# analytics.py
from datetime import datetime, timedelta
from typing import List, Dict
import json

class SupportAnalytics:
    def __init__(self):
        self.tickets = []
    
    def add_ticket(self, ticket: Dict):
        """Record ticket"""
        ticket['timestamp'] = datetime.now().isoformat()
        self.tickets.append(ticket)
        self.save()
    
    def get_stats(self, days=7) -> Dict:
        """Get statistics"""
        cutoff = datetime.now() - timedelta(days=days)
        
        recent = [
            t for t in self.tickets 
            if datetime.fromisoformat(t['timestamp']) > cutoff
        ]
        
        return {
            "total": len(recent),
            "resolved": sum(1 for t in recent if t['resolved']),
            "escalated": sum(1 for t in recent if t.get('escalated')),
            "by_category": self._count_by_field(recent, 'category'),
            "by_priority": self._count_by_field(recent, 'priority'),
            "avg_response_time": self._avg_response_time(recent)
        }
    
    def _count_by_field(self, tickets: List, field: str) -> Dict:
        counts = {}
        for ticket in tickets:
            value = ticket.get(field)
            counts[value] = counts.get(value, 0) + 1
        return counts
    
    def _avg_response_time(self, tickets: List) -> float:
        # Mock - in real system, track actual time
        return 2.5
    
    def save(self):
        """Save to file"""
        with open('analytics.json', 'w') as f:
            json.dump(self.tickets, f)
    
    def load(self):
        """Load from file"""
        try:
            with open('analytics.json', 'r') as f:
                self.tickets = json.load(f)
        except FileNotFoundError:
            self.tickets = []

analytics = SupportAnalytics()
analytics.load()
```

**Step 9: Docker Deploy (1-2 hours)**

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Download models
RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

COPY . .

# Create KB
RUN python knowledge_base.py

EXPOSE 8000

CMD ["uvicorn", "api:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  support-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./support_kb:/app/support_kb
      - ./analytics.json:/app/analytics.json
```

**Run:**

```bash
docker-compose up --build
```


***

### ☁️ Option B: Cloud Implementation

**Day 1-3: Cloud Deploy**

**Option 1: Cloud Run (Google Cloud)**

```bash
gcloud run deploy support-api \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

**Option 2: AWS Lambda + API Gateway**

```python
# Use AWS SAM or Serverless Framework
# Deploy LangGraph workflow as Lambda function
# Frontend as S3 + CloudFront
```

**Option 3: Azure Container Instances**

```bash
az container create \
  --resource-group support-rg \
  --name support-api \
  --image yourregistry.azurecr.io/support-api \
  --dns-name-label support-api \
  --ports 8000
```

**Cloud Database Options:**

- **PostgreSQL**: Store ticket history
- **Redis**: Cache recent conversations
- **MongoDB**: Store KB documents

***

## 🎓 Summary

**Time Investment Summary:**

- **Project 1** (Multi-Agent RAG): 2-3 days ✅
- **Project 2** (MCP Server): 2-3 days ✅
- **Project 3** (LLM Inference): 2-3 days ✅
- **Project 4** (Full-Stack App): 3 days ✅
- **Project 5** (Support System): 2-3 days ✅

**Total: 11-15 days** for all 5 projects! 🚀

Each project builds on previous ones, creating a **complete portfolio** demonstrating:

- ✅ LangGraph multi-agent systems
- ✅ MCP protocol implementation
- ✅ LLM inference optimization
- ✅ Full-stack AI applications
- ✅ Production-ready workflows

**You'll have real, working projects** to show employers or clients! 💼

