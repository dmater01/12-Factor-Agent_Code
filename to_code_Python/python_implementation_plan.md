# 12-Factor AI Agent Architecture: Python Implementation Plan

## Phase 1: Foundation Setup (Week 1-2)

### 1.1 Environment & Dependencies Setup

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Core dependencies
pip install \
  openai \
  pydantic \
  python-dotenv \
  uuid \
  asyncio \
  jinja2 \
  pyyaml \
  typing-extensions

# Development dependencies
pip install \
  pytest \
  pytest-asyncio \
  black \
  flake8 \
  mypy \
  jupyter
```

### 1.2 Project Structure
```
project-root/
├── src/
│   ├── agents/           # Small, focused agents
│   ├── context/          # Context engineering
│   ├── tools/           # Tool definitions
│   ├── control/         # Control flow management
│   ├── state/           # State management
│   └── types/           # Type definitions
├── prompts/             # Version-controlled prompts
├── tests/               # Testing framework
├── examples/            # Implementation examples
├── requirements.txt     # Dependencies
└── pyproject.toml       # Python project config
```

### 1.3 Core Types Definition

```python
# src/types/agent_types.py
from pydantic import BaseModel, Field
from typing import Dict, List, Any, Optional, Union, Literal
from datetime import datetime
from enum import Enum
import uuid

class ThreadStatus(str, Enum):
    RUNNING = "running"
    PAUSED = "paused"
    COMPLETED = "completed"
    ERROR = "error"

class AgentEvent(BaseModel):
    type: str
    data: Dict[str, Any]
    timestamp: datetime = Field(default_factory=datetime.now)
    id: str = Field(default_factory=lambda: f"event-{uuid.uuid4().hex[:8]}")

class AgentThread(BaseModel):
    id: str = Field(default_factory=lambda: f"thread-{uuid.uuid4().hex[:8]}")
    events: List[AgentEvent] = Field(default_factory=list)
    status: ThreadStatus = ThreadStatus.RUNNING
    metadata: Dict[str, Any] = Field(default_factory=dict)

class ToolCall(BaseModel):
    tool: str
    parameters: Dict[str, Any]
    id: str = Field(default_factory=lambda: f"tool-{uuid.uuid4().hex[:8]}")

class ToolResult(BaseModel):
    success: bool
    data: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

class AgentDecision(BaseModel):
    intent: str
    tool: Optional[str] = None
    parameters: Optional[Dict[str, Any]] = None
    reasoning: Optional[str] = None
    confidence: Optional[float] = None
```

## Phase 2: Core Factor Implementation (Week 2-4)

### 2.1 Factor 2: Own Your Prompts

```python
# src/context/prompt_manager.py
import os
import re
from pathlib import Path
from typing import Dict, Any
from jinja2 import Environment, FileSystemLoader

class PromptManager:
    def __init__(self, prompts_dir: str = "prompts"):
        self.prompts_dir = Path(prompts_dir)
        self.prompts_cache: Dict[str, str] = {}
        self.jinja_env = Environment(
            loader=FileSystemLoader(self.prompts_dir),
            trim_blocks=True,
            lstrip_blocks=True
        )
    
    def load_prompt(self, name: str) -> str:
        """Load prompt from file with caching"""
        if name in self.prompts_cache:
            return self.prompts_cache[name]
        
        prompt_path = self.prompts_dir / f"{name}.txt"
        if not prompt_path.exists():
            raise FileNotFoundError(f"Prompt file not found: {prompt_path}")
        
        with open(prompt_path, 'r', encoding='utf-8') as f:
            prompt = f.read()
        
        self.prompts_cache[name] = prompt
        return prompt
    
    def render_prompt(self, template: str, variables: Dict[str, Any]) -> str:
        """Render prompt template with variables using Jinja2"""
        template_obj = self.jinja_env.from_string(template)
        return template_obj.render(**variables)
    
    def render_prompt_file(self, name: str, variables: Dict[str, Any]) -> str:
        """Load and render prompt file"""
        template = self.load_prompt(name)
        return self.render_prompt(template, variables)
```

Create your first prompt template:
```txt
# prompts/customer_review_triage.txt
You are an expert customer service analyst. Your job is to analyze customer reviews and categorize them for appropriate action.

Current date: {{ current_date }}

Here's what happened so far:
{{ thread_events }}

Analyze the following review and respond with a JSON object:

<user_review>
{{ review_text }}
</user_review>

Respond only with valid JSON in this format:
{
  "category": "urgent_action_required" | "feedback" | "praise" | "complaint",
  "sentiment": "positive" | "negative" | "neutral",
  "urgency": 1-5,
  "extracted_entities": {
    "product_name": "string",
    "order_id": "string or null",
    "issue_type": "string"
  },
  "suggested_next_action": "create_high_priority_ticket" | "route_to_support" | "thank_customer" | "no_action",
  "reasoning": "Brief explanation of the decision"
}
```

### 2.2 Factor 3: Own Your Context Window

```python
# src/context/context_builder.py
from typing import List, Dict, Any
from datetime import datetime
import json
import yaml
from ..types.agent_types import AgentEvent

class ContextBuilder:
    def __init__(self):
        self.events: List[AgentEvent] = []
    
    def add_event(self, event: AgentEvent) -> None:
        """Add event to context"""
        self.events.append(event)
    
    def build_xml_context(self) -> str:
        """Build structured XML context for optimal LLM processing"""
        context_parts = []
        
        for event in self.events:
            context_parts.append(f"<{event.type}>")
            
            if isinstance(event.data, dict):
                # Format as YAML for readability
                context_parts.append(yaml.dump(event.data, default_flow_style=False))
            else:
                context_parts.append(str(event.data))
            
            context_parts.append(f"</{event.type}>")
            context_parts.append("")  # Empty line for separation
        
        return "\n".join(context_parts).strip()
    
    def build_structured_context(self) -> Dict[str, Any]:
        """Build structured context for programmatic use"""
        context = {
            "events": [event.dict() for event in self.events],
            "summary": self._generate_summary(),
            "recent_errors": self._extract_recent_errors(),
            "current_state": self._infer_current_state()
        }
        return context
    
    def compact_errors(self, max_errors: int = 3) -> None:
        """Keep only recent errors to manage context size"""
        error_events = [e for e in self.events if e.type in ['error', 'tool_error']]
        if len(error_events) > max_errors:
            # Remove older error events
            old_errors = error_events[:-max_errors]
            self.events = [e for e in self.events if e not in old_errors]
    
    def _generate_summary(self) -> str:
        """Generate a brief summary of what has happened"""
        if not self.events:
            return "No events recorded"
        
        event_types = [e.type for e in self.events]
        unique_types = list(set(event_types))
        
        return f"Events: {', '.join(unique_types)} (Total: {len(self.events)})"
    
    def _extract_recent_errors(self) -> List[Dict[str, Any]]:
        """Extract recent errors for context"""
        error_events = [e for e in self.events if e.type in ['error', 'tool_error']]
        return [e.dict() for e in error_events[-3:]]  # Last 3 errors
    
    def _infer_current_state(self) -> str:
        """Infer current state from events"""
        if not self.events:
            return "initialized"
        
        last_event = self.events[-1]
        if last_event.type == 'agent_decision':
            return "awaiting_execution"
        elif last_event.type == 'tool_result':
            return "processing_result"
        elif last_event.type == 'error':
            return "error_recovery"
        else:
            return "active"
```

### 2.3 Factor 4: Tools Are Just Structured Outputs

```python
# src/tools/tool_registry.py
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Callable, Optional
from pydantic import BaseModel, ValidationError
import asyncio
from ..types.agent_types import ToolCall, ToolResult

class Tool(ABC):
    def __init__(self, name: str, description: str, schema: type[BaseModel]):
        self.name = name
        self.description = description
        self.schema = schema
    
    @abstractmethod
    async def execute(self, params: Dict[str, Any]) -> ToolResult:
        """Execute the tool with given parameters"""
        pass
    
    def validate_params(self, params: Dict[str, Any]) -> BaseModel:
        """Validate parameters against schema"""
        try:
            return self.schema(**params)
        except ValidationError as e:
            raise ValueError(f"Invalid parameters for {self.name}: {e}")

class ToolRegistry:
    def __init__(self):
        self.tools: Dict[str, Tool] = {}
    
    def register(self, tool: Tool) -> None:
        """Register a tool"""
        self.tools[tool.name] = tool
    
    async def execute(self, tool_call: ToolCall) -> ToolResult:
        """Execute a tool call"""
        tool = self.tools.get(tool_call.tool)
        if not tool:
            return ToolResult(
                success=False,
                error=f"Tool {tool_call.tool} not found"
            )
        
        try:
            validated_params = tool.validate_params(tool_call.parameters)
            return await tool.execute(validated_params.dict())
        except Exception as e:
            return ToolResult(
                success=False,
                error=f"Tool execution failed: {str(e)}"
            )
    
    def get_tool_definitions(self) -> List[Dict[str, Any]]:
        """Get tool definitions for LLM context"""
        return [
            {
                "name": tool.name,
                "description": tool.description,
                "schema": tool.schema.schema()
            }
            for tool in self.tools.values()
        ]
```

Example tool implementations:
```python
# src/tools/customer_support_tools.py
from pydantic import BaseModel
from typing import Optional
from .tool_registry import Tool, ToolResult

class CreateTicketParams(BaseModel):
    title: str
    description: str
    customer_id: Optional[str] = None
    product: Optional[str] = None
    urgency: int = 3

class CreateHighPriorityTicket(Tool):
    def __init__(self):
        super().__init__(
            name="create_high_priority_ticket",
            description="Creates a high priority support ticket",
            schema=CreateTicketParams
        )
    
    async def execute(self, params: dict) -> ToolResult:
        # Simulate ticket creation
        ticket_id = f"TICKET-{int(datetime.now().timestamp())}"
        
        print(f"🎫 Created high priority ticket: {ticket_id}")
        print(f"   Title: {params['title']}")
        print(f"   Description: {params['description']}")
        
        return ToolResult(
            success=True,
            data={
                "ticket_id": ticket_id,
                "status": "created",
                "priority": "high"
            }
        )

class RequestHumanInputParams(BaseModel):
    question: str
    context: str
    urgency: str = "medium"
    format: str = "free_text"
    choices: Optional[List[str]] = None

class RequestHumanInput(Tool):
    def __init__(self):
        super().__init__(
            name="request_human_input",
            description="Request input or approval from a human",
            schema=RequestHumanInputParams
        )
    
    async def execute(self, params: dict) -> ToolResult:
        print(f"🤖 HUMAN INPUT REQUESTED:")
        print(f"   Question: {params['question']}")
        print(f"   Context: {params['context']}")
        print(f"   Urgency: {params['urgency']}")
        
        # In production, this would integrate with Slack, email, etc.
        # For demo, simulate approval
        return ToolResult(
            success=True,
            data={
                "approved": True,
                "response": "Approved by human operator",
                "timestamp": datetime.now().isoformat()
            }
        )
```

## Phase 3: Control Flow & State Management (Week 4-6)

### 3.1 Factor 5: Unify Execution State and Business State

```python
# src/state/thread_manager.py
import json
from typing import Dict, Optional
from datetime import datetime
from ..types.agent_types import AgentThread, AgentEvent, ThreadStatus

class ThreadManager:
    def __init__(self):
        self.threads: Dict[str, AgentThread] = {}
    
    def create_thread(self, initial_event: AgentEvent) -> AgentThread:
        """Create a new agent thread"""
        thread = AgentThread(events=[initial_event])
        self.threads[thread.id] = thread
        return thread
    
    def add_event(self, thread_id: str, event: AgentEvent) -> None:
        """Add event to thread"""
        thread = self.threads.get(thread_id)
        if thread:
            thread.events.append(event)
    
    def pause_thread(self, thread_id: str) -> None:
        """Pause thread execution"""
        thread = self.threads.get(thread_id)
        if thread:
            thread.status = ThreadStatus.PAUSED
    
    def resume_thread(self, thread_id: str) -> None:
        """Resume thread execution"""
        thread = self.threads.get(thread_id)
        if thread:
            thread.status = ThreadStatus.RUNNING
    
    def complete_thread(self, thread_id: str) -> None:
        """Mark thread as completed"""
        thread = self.threads.get(thread_id)
        if thread:
            thread.status = ThreadStatus.COMPLETED
    
    def get_thread(self, thread_id: str) -> Optional[AgentThread]:
        """Get thread by ID"""
        return self.threads.get(thread_id)
    
    def serialize_thread(self, thread_id: str) -> str:
        """Serialize thread for persistence"""
        thread = self.threads.get(thread_id)
        if not thread:
            return ""
        return thread.json()
    
    def deserialize_thread(self, data: str) -> AgentThread:
        """Deserialize thread from storage"""
        thread_data = json.loads(data)
        thread = AgentThread(**thread_data)
        self.threads[thread.id] = thread
        return thread
```

### 3.2 Factor 8: Own Your Control Flow

```python
# src/control/agent_orchestrator.py
import json
from typing import Optional, List
from openai import AsyncOpenAI
from ..context.prompt_manager import PromptManager
from ..context.context_builder import ContextBuilder
from ..tools.tool_registry import ToolRegistry
from ..state.thread_manager import ThreadManager
from ..types.agent_types import AgentEvent, AgentDecision, ToolCall

class LLMClient:
    def __init__(self, api_key: str):
        self.client = AsyncOpenAI(api_key=api_key)
    
    async def get_completion(
        self, 
        prompt: str, 
        model: str = "gpt-4",
        temperature: float = 0.1
    ) -> str:
        """Get completion from LLM"""
        try:
            response = await self.client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                temperature=temperature
            )
            return response.choices[0].message.content
        except Exception as e:
            raise Exception(f"LLM request failed: {str(e)}")

class AgentOrchestrator:
    def __init__(
        self,
        prompt_manager: PromptManager,
        tool_registry: ToolRegistry,
        thread_manager: ThreadManager,
        llm_client: LLMClient
    ):
        self.prompt_manager = prompt_manager
        self.tool_registry = tool_registry
        self.thread_manager = thread_manager
        self.llm_client = llm_client
        self.high_stakes_tools = {
            "issue_refund",
            "deploy_to_production", 
            "delete_user_data",
            "send_external_email"
        }
    
    async def run_agent(
        self,
        thread_id: str,
        agent_type: str,
        max_steps: int = 10
    ) -> None:
        """Run agent with control flow management"""
        thread = self.thread_manager.get_thread(thread_id)
        if not thread or thread.status != "running":
            raise ValueError("Thread not found or not running")
        
        steps = 0
        consecutive_errors = 0
        max_consecutive_errors = 3
        
        while steps < max_steps and thread.status == "running":
            try:
                # Build context
                context_builder = ContextBuilder()
                for event in thread.events:
                    context_builder.add_event(event)
                
                # Get next step from LLM
                next_step = await self._determine_next_step(
                    agent_type,
                    context_builder.build_xml_context()
                )
                
                # Add decision to thread
                decision_event = AgentEvent(
                    type="agent_decision",
                    data=next_step.dict()
                )
                self.thread_manager.add_event(thread_id, decision_event)
                
                # Handle different intents
                if next_step.intent == "done":
                    self.thread_manager.complete_thread(thread_id)
                    break
                
                if next_step.intent == "request_human_input":
                    await self._handle_human_request(thread_id, next_step)
                    break  # Pause execution
                
                # Execute tool call with safety controls
                if next_step.tool:
                    result = await self._execute_with_safety(thread_id, next_step)
                    
                    if not result.success:
                        consecutive_errors += 1
                        if consecutive_errors >= max_consecutive_errors:
                            await self._escalate_to_human(thread_id, result.error)
                            break
                    else:
                        consecutive_errors = 0
                
                steps += 1
                
            except Exception as e:
                consecutive_errors += 1
                error_event = AgentEvent(
                    type="error",
                    data={"error": str(e), "step": steps}
                )
                self.thread_manager.add_event(thread_id, error_event)
                
                if consecutive_errors >= max_consecutive_errors:
                    await self._escalate_to_human(thread_id, str(e))
                    break
    
    async def _determine_next_step(
        self, 
        agent_type: str, 
        context: str
    ) -> AgentDecision:
        """Get next step decision from LLM"""
        prompt = self.prompt_manager.render_prompt_file(
            agent_type,
            {
                "current_date": datetime.now().isoformat(),
                "thread_events": context,
                "tool_definitions": self.tool_registry.get_tool_definitions()
            }
        )
        
        response = await self.llm_client.get_completion(prompt)
        
        try:
            decision_data = json.loads(response)
            return AgentDecision(**decision_data)
        except json.JSONDecodeError as e:
            raise ValueError(f"Invalid JSON response from LLM: {e}")
    
    async def _execute_with_safety(
        self, 
        thread_id: str, 
        decision: AgentDecision
    ) -> ToolResult:
        """Execute tool with safety controls"""
        # Check for high-stakes operations
        if decision.tool in self.high_stakes_tools:
            return await self._request_approval_and_execute(thread_id, decision)
        
        # Execute normally
        tool_call = ToolCall(
            tool=decision.tool,
            parameters=decision.parameters or {}
        )
        
        result = await self.tool_registry.execute(tool_call)
        
        # Add result to thread (Factor 9: Compact Errors)
        result_event = AgentEvent(
            type="tool_result" if result.success else "tool_error",
            data=result.dict()
        )
        self.thread_manager.add_event(thread_id, result_event)
        
        return result
    
    async def _request_approval_and_execute(
        self, 
        thread_id: str, 
        decision: AgentDecision
    ) -> ToolResult:
        """Request human approval for high-stakes operations"""
        print(f"🚨 HIGH-STAKES OPERATION DETECTED: {decision.tool}")
        print(f"   Parameters: {decision.parameters}")
        print(f"   Reasoning: {decision.reasoning}")
        
        # In production, this would pause and wait for human approval
        # For demo, we'll simulate approval
        approval_granted = True  # Simulate human approval
        
        if approval_granted:
            tool_call = ToolCall(
                tool=decision.tool,
                parameters=decision.parameters or {}
            )
            return await self.tool_registry.execute(tool_call)
        else:
            return ToolResult(
                success=False,
                error="Human approval denied"
            )
    
    async def _handle_human_request(
        self, 
        thread_id: str, 
        decision: AgentDecision
    ) -> None:
        """Handle human input request"""
        self.thread_manager.pause_thread(thread_id)
        
        # Execute the human input request tool
        if decision.tool == "request_human_input":
            tool_call = ToolCall(
                tool=decision.tool,
                parameters=decision.parameters or {}
            )
            result = await self.tool_registry.execute(tool_call)
            
            result_event = AgentEvent(
                type="human_input_result",
                data=result.dict()
            )
            self.thread_manager.add_event(thread_id, result_event)
    
    async def _escalate_to_human(self, thread_id: str, error: str) -> None:
        """Escalate to human support"""
        escalation_event = AgentEvent(
            type="escalation",
            data={
                "reason": "Multiple consecutive errors",
                "error_details": error,
                "thread_id": thread_id
            }
        )
        self.thread_manager.add_event(thread_id, escalation_event)
        self.thread_manager.pause_thread(thread_id)
        
        print(f"🚨 ESCALATED TO HUMAN: {error}")
```

## Phase 4: Error Handling & Micro-Agents (Week 6-8)

### 4.1 Factor 9: Compact Errors into Context Window

```python
# src/control/error_handler.py
from typing import List, Dict, Any
from datetime import datetime
from ..types.agent_types import AgentEvent

class ErrorHandler:
    @staticmethod
    def format_error(error: Exception, context: Dict[str, Any] = None) -> AgentEvent:
        """Format error as structured event"""
        return AgentEvent(
            type="structured_error",
            data={
                "error_type": type(error).__name__,
                "message": str(error),
                "context": context or {},
                "timestamp": datetime.now().isoformat(),
                "recoverable": True
            }
        )
    
    @staticmethod
    def should_retry(consecutive_errors: int, max_retries: int = 3) -> bool:
        """Determine if operation should be retried"""
        return consecutive_errors < max_retries
    
    @staticmethod
    def build_error_context(errors: List[AgentEvent]) -> str:
        """Build error context for LLM"""
        recent_errors = [
            e for e in errors 
            if e.type in ["structured_error", "tool_error"]
        ][-3:]  # Last 3 errors
        
        if not recent_errors:
            return ""
        
        error_context = "<recent_errors>\n"
        for error in recent_errors:
            error_context += f"  <error>\n"
            error_context += f"    Type: {error.data.get('error_type', 'Unknown')}\n"
            error_context += f"    Message: {error.data.get('message', 'No message')}\n"
            error_context += f"    Time: {error.data.get('timestamp', 'Unknown')}\n"
            error_context += f"  </error>\n"
        error_context += "</recent_errors>"
        
        return error_context
    
    @staticmethod
    def extract_learning_from_error(error_event: AgentEvent) -> Dict[str, Any]:
        """Extract learning information from error"""
        return {
            "what_failed": error_event.data.get("message", "Unknown"),
            "error_type": error_event.data.get("error_type", "Unknown"),
            "context": error_event.data.get("context", {}),
            "suggested_alternative": "Try a different approach or request human help"
        }
```

### 4.2 Factor 10: Small, Focused Agents

```python
# src/agents/customer_review_agent.py
from typing import Dict, Any, Optional
from ..control.agent_orchestrator import AgentOrchestrator
from ..state.thread_manager import ThreadManager
from ..types.agent_types import AgentEvent

class CustomerReviewTriageAgent:
    """
    Micro-agent focused solely on customer review analysis and triage.
    Limited to 3-5 steps max to maintain context window efficiency.
    """
    
    def __init__(
        self,
        orchestrator: AgentOrchestrator,
        thread_manager: ThreadManager
    ):
        self.orchestrator = orchestrator
        self.thread_manager = thread_manager
        self.agent_type = "customer_review_triage"
        self.max_steps = 5  # Keep it small and focused
    
    async def process_review(self, review_text: str, customer_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Process a single customer review and return structured analysis.
        This is the ONLY job of this micro-agent.
        """
        # Create focused thread for this single task
        initial_event = AgentEvent(
            type="customer_review",
            data={
                "review_text": review_text,
                "customer_id": customer_id,
                "task": "triage_and_categorize"
            }
        )
        
        thread = self.thread_manager.create_thread(initial_event)
        
        try:
            # Run the focused agent (max 5 steps for this simple task)
            await self.orchestrator.run_agent(
                thread.id, 
                self.agent_type, 
                self.max_steps
            )
            
            # Extract the result
            decisions = [
                e for e in thread.events 
                if e.type == "agent_decision" and e.data.get("intent") == "analysis_complete"
            ]
            
            if decisions:
                return decisions[-1].data.get("analysis_result", {})
            else:
                # Fallback: extract from tool results
                tool_results = [
                    e for e in thread.events 
                    if e.type == "tool_result" and e.data.get("success")
                ]
                if tool_results:
                    return tool_results[-1].data.get("data", {})
                
                return {"error": "No analysis result found"}
                
        except Exception as e:
            return {
                "error": f"Review processing failed: {str(e)}",
                "thread_id": thread.id
            }

class DeploymentAgent:
    """
    Micro-agent focused on deployment workflow management.
    Handles deployment decisions but not the actual deployment execution.
    """
    
    def __init__(
        self,
        orchestrator: AgentOrchestrator,
        thread_manager: ThreadManager
    ):
        self.orchestrator = orchestrator
        self.thread_manager = thread_manager
        self.agent_type = "deployment_workflow"
        self.max_steps = 10  # Slightly larger for deployment complexity
    
    async def manage_deployment(
        self, 
        deployment_request: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Manage deployment workflow including approvals and sequencing.
        Does NOT perform actual deployments - that's deterministic code.
        """
        initial_event = AgentEvent(
            type="deployment_request",
            data=deployment_request
        )
        
        thread = self.thread_manager.create_thread(initial_event)
        
        try:
            await self.orchestrator.run_agent(
                thread.id,
                self.agent_type,
                self.max_steps
            )
            
            # Extract deployment plan
            decisions = [
                e for e in thread.events 
                if e.type == "agent_decision"
            ]
            
            return {
                "deployment_plan": [d.data for d in decisions],
                "thread_id": thread.id,
                "status": thread.status
            }
            
        except Exception as e:
            return {
                "error": f"Deployment planning failed: {str(e)}",
                "thread_id": thread.id
            }
```

Create specialized prompt for the micro-agent:
```txt
# prompts/customer_review_triage.txt
You are a Customer Review Triage Specialist. Your ONLY job is to analyze ONE customer review and output structured analysis.

You are NOT responsible for:
- Sending emails
- Creating tickets (you can suggest it)
- Issuing refunds
- Contacting customers

You ARE responsible for:
- Categorizing the review accurately
- Extracting key information
- Determining urgency level
- Suggesting next action (but not executing it)

Current context:
{{ thread_events }}

Available tools:
- analyze_review_complete: Mark analysis as complete with results

Review to analyze:
<review>
{{ review_text }}
</review>

{% if customer_id %}
Customer ID: {{ customer_id }}
{% endif %}

Respond with JSON in this format:
{
  "intent": "analysis_complete",
  "tool": "analyze_review_complete",
  "parameters": {
    "category": "urgent_action_required" | "feedback" | "praise" | "complaint",
    "sentiment": "positive" | "negative" | "neutral",
    "urgency": 1-5,
    "extracted_entities": {
      "product_name": "string or null",
      "order_id": "string or null", 
      "issue_type": "string or null",
      "customer_emotion": "string"
    },
    "suggested_next_action": "create_high_priority_ticket" | "route_to_support" | "thank_customer" | "no_action",
    "key_phrases": ["important", "phrases", "from", "review"],
    "requires_human_attention": true | false
  },
  "reasoning": "Brief explanation of analysis decisions",
  "confidence": 0.95
}
```

## Phase 5: Advanced Features & Integration (Week 8-10)

### 5.1 Factor 11: Trigger from Anywhere

```python
# src/triggers/multi_channel_trigger.py
from abc import ABC, abstractmethod
from typing import Dict, Any, Callable, Awaitable
import asyncio
from datetime import datetime

class TriggerHandler(ABC):
    @abstractmethod
    async def handle_trigger(self, data: Dict[str, Any]) -> None:
        pass

class WebhookTrigger:
    def __init__(self, agent_orchestrator):
        self.orchestrator = agent_orchestrator
        self.handlers: Dict[str, TriggerHandler] = {}
    
    def register_handler(self, event_type: str, handler: TriggerHandler):
        self.handlers[event_type] = handler
    
    async def handle_webhook(self, event_type: str, payload: Dict[str, Any]):
        """Handle incoming webhook events"""
        handler = self.handlers.get(event_type)
        if handler:
            await handler.handle_trigger(payload)
        else:
            print(f"No handler registered for event type: {event_type}")

class EmailTrigger(TriggerHandler):
    def __init__(self, review_agent: CustomerReviewTriageAgent):
        self.review_agent = review_agent
    
    async def handle_trigger(self, data: Dict[str, Any]) -> None:
        """Handle email-based review submission"""
        email_content = data.get("content", "")
        customer_email = data.get("from", "")
        
        result = await self.review_agent.process_review(
            review_text=email_content,
            customer_id=customer_email
        )
        
        print(f"📧 Email review processed: {result}")

class SlackTrigger(TriggerHandler):
    def __init__(self, deployment_agent: DeploymentAgent):
        self.deployment_agent = deployment_agent
    
    async def handle_trigger(self, data: Dict[str, Any]) -> None:
        """Handle Slack deployment request"""
        slack_message = data.get("text", "")
        user_id = data.get("user", "")
        
        deployment_request = {
            "requested_by": user_id,
            "request_text": slack_message,
            "channel": "slack",
            "timestamp": datetime.now().isoformat()
        }
        
        result = await self.deployment_agent.manage_deployment(deployment_request)
        print(f"🚀 Slack deployment request processed: {result}")

class CronTrigger:
    def __init__(self):
        self.scheduled_jobs: Dict[str, Callable] = {}
    
    def schedule_job(self, name: str, cron_expression: str, job_func: Callable):
        """Schedule a cron job (simplified implementation)"""
        self.scheduled_jobs[name] = job_func
        print(f"📅 Scheduled job: {name} with expression: {cron_expression}")
    
    async def run_scheduled_job(self, name: str):
        """Run a scheduled job"""
        job = self.scheduled_jobs.get(name)
        if job:
            await job()
```

### 5.2 Factor 12: Stateless Reducer Pattern

```python
# src/agents/stateless_agent.py
from typing import Dict, Any, List
from ..types.agent_types import AgentThread, AgentEvent, AgentDecision

class StatelessAgentReducer:
    """
    Implements agents as pure functions that reduce state.
    No side effects, fully deterministic and testable.
    """
    
    @staticmethod
    def reduce_customer_review_state(
        current_thread: AgentThread,
        new_event: AgentEvent
    ) -> AgentDecision:
        """
        Pure function to determine next action based on current state.
        No side effects - just state transformation.
        """
        # Extract relevant state from thread events
        review_events = [e for e in current_thread.events if e.type == "customer_review"]
        error_events = [e for e in current_thread.events if e.type == "error"]
        tool_results = [e for e in current_thread.events if e.type == "tool_result"]
        
        # Analyze current state
        if not review_events:
            return AgentDecision(
                intent="error",
                reasoning="No review found to analyze"
            )
        
        review_data = review_events[0].data
        review_text = review_data.get("review_text", "")
        
        # Simple rule-based analysis (in production, this would call LLM)
        if any(word in review_text.lower() for word in ["fire", "smoke", "dangerous", "injury"]):
            urgency = 5
            category = "urgent_action_required"
            next_action = "create_high_priority_ticket"
        elif any(word in review_text.lower() for word in ["love", "great", "excellent", "amazing"]):
            urgency = 1
            category = "praise"
            next_action = "thank_customer"
        elif any(word in review_text.lower() for word in ["problem", "issue", "broken", "defect"]):
            urgency = 3
            category = "complaint"
            next_action = "route_to_support"
        else:
            urgency = 2
            category = "feedback"
            next_action = "no_action"
        
        return AgentDecision(
            intent="analysis_complete",
            tool="analyze_review_complete",
            parameters={
                "category": category,
                "urgency": urgency,
                "suggested_next_action": next_action,
                "sentiment": "positive" if category == "praise" else "negative" if urgency > 3 else "neutral"
            },
            reasoning=f"Analyzed review and determined category: {category} with urgency: {urgency}"
        )
    
    @staticmethod
    def reduce_deployment_state(
        current_thread: AgentThread,
        new_event: AgentEvent
    ) -> AgentDecision:
        """
        Pure function for deployment workflow decisions.
        """
        deployment_events = [e for e in current_thread.events if e.type == "deployment_request"]
        approval_events = [e for e in current_thread.events if e.type == "human_input_result"]
        
        if not deployment_events:
            return AgentDecision(
                intent="error",
                reasoning="No deployment request found"
            )
        
        deployment_request = deployment_events[0].data
        request_text = deployment_request.get("request_text", "").lower()
        
        # Check if this is a production deployment
        if "production" in request_text or "prod" in request_text:
            # Check if we already have approval
            if not approval_events:
                return AgentDecision(
                    intent="request_human_input",
                    tool="request_human_input",
                    parameters={
                        "question": f"Approve production deployment: {request_text}",
                        "context": "Production deployment requires explicit approval",
                        "urgency": "high"
                    },
                    reasoning="Production deployment requires human approval"
                )
            else:
                # We have approval, proceed
                return AgentDecision(
                    intent="deploy",
                    tool="deploy_to_production",
                    parameters={"request": deployment_request},
                    reasoning="Human approval received, proceeding with deployment"
                )
        else:
            # Non-production deployment
            return AgentDecision(
                intent="deploy",
                tool="deploy_to_staging",
                parameters={"request": deployment_request},
                reasoning="Staging deployment, no approval required"
            )
```

### 5.3 Testing Framework

```python
# tests/test_agents.py
import pytest
import asyncio
from unittest.mock import Mock, AsyncMock
from src.agents.customer_review_agent import CustomerReviewTriageAgent
from src.types.agent_types import AgentEvent, AgentThread
from src.agents.stateless_agent import StatelessAgentReducer

class TestCustomerReviewAgent:
    
    @pytest.fixture
    def mock_orchestrator(self):
        return Mock()
    
    @pytest.fixture
    def mock_thread_manager(self):
        manager = Mock()
        manager.create_thread.return_value = AgentThread(id="test-thread")
        return manager
    
    @pytest.fixture
    def review_agent(self, mock_orchestrator, mock_thread_manager):
        return CustomerReviewTriageAgent(mock_orchestrator, mock_thread_manager)
    
    @pytest.mark.asyncio
    async def test_urgent_review_processing(self, review_agent):
        """Test processing of urgent product safety review"""
        urgent_review = "The Super Juicer 5000 started sparking and smoking! This is extremely dangerous!"
        
        # Mock the orchestrator to return urgent analysis
        review_agent.orchestrator.run_agent = AsyncMock()
        
        # Mock thread events to simulate agent decision
        mock_thread = AgentThread(
            id="test-thread",
            events=[
                AgentEvent(
                    type="customer_review",
                    data={"review_text": urgent_review}
                ),
                AgentEvent(
                    type="agent_decision",
                    data={
                        "intent": "analysis_complete",
                        "analysis_result": {
                            "category": "urgent_action_required",
                            "urgency": 5,
                            "suggested_next_action": "create_high_priority_ticket"
                        }
                    }
                )
            ]
        )
        
        review_agent.thread_manager.create_thread.return_value = mock_thread
        review_agent.thread_manager.get_thread.return_value = mock_thread
        
        result = await review_agent.process_review(urgent_review)
        
        assert result["category"] == "urgent_action_required"
        assert result["urgency"] == 5
        assert result["suggested_next_action"] == "create_high_priority_ticket"
    
    @pytest.mark.asyncio
    async def test_positive_review_processing(self, review_agent):
        """Test processing of positive customer feedback"""
        positive_review = "Love this product! Works perfectly and great customer service!"
        
        review_agent.orchestrator.run_agent = AsyncMock()
        
        mock_thread = AgentThread(
            id="test-thread",
            events=[
                AgentEvent(
                    type="customer_review",
                    data={"review_text": positive_review}
                ),
                AgentEvent(
                    type="agent_decision", 
                    data={
                        "intent": "analysis_complete",
                        "analysis_result": {
                            "category": "praise",
                            "sentiment": "positive",
                            "suggested_next_action": "thank_customer"
                        }
                    }
                )
            ]
        )
        
        review_agent.thread_manager.create_thread.return_value = mock_thread
        
        result = await review_agent.process_review(positive_review)
        
        assert result["category"] == "praise"
        assert result["sentiment"] == "positive"
        assert result["suggested_next_action"] == "thank_customer"

class TestStatelessReducer:
    
    def test_urgent_review_reduction(self):
        """Test stateless reducer with urgent review"""
        thread = AgentThread(
            events=[
                AgentEvent(
                    type="customer_review",
                    data={"review_text": "Product caught fire! Dangerous!"}
                )
            ]
        )
        
        decision = StatelessAgentReducer.reduce_customer_review_state(
            thread, 
            AgentEvent(type="analyze", data={})
        )
        
        assert decision.parameters["category"] == "urgent_action_required"
        assert decision.parameters["urgency"] == 5
        assert decision.parameters["suggested_next_action"] == "create_high_priority_ticket"
    
    def test_positive_review_reduction(self):
        """Test stateless reducer with positive review"""
        thread = AgentThread(
            events=[
                AgentEvent(
                    type="customer_review",
                    data={"review_text": "Amazing product! Love it!"}
                )
            ]
        )
        
        decision = StatelessAgentReducer.reduce_customer_review_state(
            thread,
            AgentEvent(type="analyze", data={})
        )
        
        assert decision.parameters["category"] == "praise"
        assert decision.parameters["suggested_next_action"] == "thank_customer"
    
    def test_deployment_approval_required(self):
        """Test deployment reducer requires approval for production"""
        thread = AgentThread(
            events=[
                AgentEvent(
                    type="deployment_request",
                    data={"request_text": "Deploy v2.1 to production"}
                )
            ]
        )
        
        decision = StatelessAgentReducer.reduce_deployment_state(
            thread,
            AgentEvent(type="process", data={})
        )
        
        assert decision.intent == "request_human_input"
        assert "production" in decision.parameters["question"].lower()

# Run tests with: python -m pytest tests/ -v
```

## Phase 6: Production Deployment & Advanced Features (Week 10-12)

### 6.1 Context Pre-fetching and Optimization

```python
# src/context/context_prefetcher.py
from typing import Dict, Any, List, Optional
import asyncio
from datetime import datetime, timedelta

class ContextPrefetcher:
    def __init__(self):
        self.cache: Dict[str, Any] = {}
        self.cache_ttl: Dict[str, datetime] = {}
    
    async def prefetch_likely_context(
        self, 
        agent_type: str, 
        user_input: str
    ) -> Dict[str, Any]:
        """Prefetch likely needed context based on agent type and input"""
        prefetched_data = {}
        
        if agent_type == "customer_review_triage":
            prefetched_data.update(await self._prefetch_review_context(user_input))
        elif agent_type == "deployment_workflow":
            prefetched_data.update(await self._prefetch_deployment_context(user_input))
        
        # Always include current timestamp
        prefetched_data["current_date"] = datetime.now().isoformat()
        prefetched_data["current_timestamp"] = datetime.now().timestamp()
        
        return prefetched_data
    
    async def _prefetch_review_context(self, review_text: str) -> Dict[str, Any]:
        """Prefetch context for review analysis"""
        context = {}
        
        # Prefetch common issues (cached)
        context["common_issues"] = await self._get_cached_data(
            "common_issues",
            self._fetch_common_issues,
            ttl_hours=24
        )
        
        # Prefetch product catalog if product mentioned
        if any(product in review_text.lower() for product in ["juicer", "blender", "mixer"]):
            context["product_catalog"] = await self._get_cached_data(
                "product_catalog",
                self._fetch_product_catalog,
                ttl_hours=12
            )
        
        # Prefetch escalation procedures for urgent keywords
        if any(urgent in review_text.lower() for urgent in ["fire", "smoke", "dangerous", "injury"]):
            context["escalation_procedures"] = await self._fetch_escalation_procedures()
        
        return context
    
    async def _prefetch_deployment_context(self, request_text: str) -> Dict[str, Any]:
        """Prefetch context for deployment workflow"""
        context = {}
        
        # Always prefetch current deployment status
        context["deployment_status"] = await self._get_cached_data(
            "deployment_status",
            self._fetch_deployment_status,
            ttl_hours=1
        )
        
        # Prefetch git information if version mentioned
        if any(word in request_text.lower() for word in ["v", "version", "tag", "commit"]):
            context["git_tags"] = await self._get_cached_data(
                "git_tags",
                self._fetch_git_tags,
                ttl_hours=6
            )
        
        # Prefetch approval chain for production deployments
        if "production" in request_text.lower() or "prod" in request_text.lower():
            context["approval_chain"] = await self._fetch_approval_chain()
        
        return context
    
    async def _get_cached_data(
        self, 
        key: str, 
        fetch_func: callable, 
        ttl_hours: int = 1
    ) -> Any:
        """Get data from cache or fetch if expired"""
        now = datetime.now()
        
        # Check if data exists and is still valid
        if key in self.cache and key in self.cache_ttl:
            if now < self.cache_ttl[key]:
                return self.cache[key]
        
        # Fetch fresh data
        data = await fetch_func()
        self.cache[key] = data
        self.cache_ttl[key] = now + timedelta(hours=ttl_hours)
        
        return data
    
    async def _fetch_common_issues(self) -> List[str]:
        """Fetch common customer issues"""
        # Simulate API call
        await asyncio.sleep(0.1)
        return [
            "product_defect",
            "shipping_delay", 
            "billing_issue",
            "user_error",
            "compatibility_issue",
            "packaging_damage"
        ]
    
    async def _fetch_product_catalog(self) -> Dict[str, Any]:
        """Fetch product catalog information"""
        await asyncio.sleep(0.1)
        return {
            "super_juicer_5000": {
                "id": "SJ5000",
                "category": "kitchen_appliances",
                "warranty_months": 24,
                "known_issues": ["motor_overheating", "plastic_cracking"]
            },
            "power_blender_pro": {
                "id": "PBP200",
                "category": "kitchen_appliances", 
                "warranty_months": 36,
                "known_issues": ["blade_dulling", "base_vibration"]
            }
        }
    
    async def _fetch_escalation_procedures(self) -> Dict[str, Any]:
        """Fetch escalation procedures for urgent issues"""
        await asyncio.sleep(0.1)
        return {
            "safety_incidents": {
                "immediate_actions": ["create_urgent_ticket", "notify_safety_team"],
                "notify_within_minutes": 15,
                "escalate_to": ["safety_manager", "product_manager"]
            }
        }
    
    async def _fetch_deployment_status(self) -> Dict[str, Any]:
        """Fetch current deployment status"""
        await asyncio.sleep(0.1)
        return {
            "staging": {"status": "healthy", "version": "v2.0.1", "last_deploy": "2024-01-15T10:30:00Z"},
            "production": {"status": "healthy", "version": "v2.0.0", "last_deploy": "2024-01-14T14:20:00Z"}
        }
    
    async def _fetch_git_tags(self) -> List[Dict[str, str]]:
        """Fetch recent git tags"""
        await asyncio.sleep(0.1)
        return [
            {"name": "v2.0.1", "commit": "abc123def", "date": "2024-01-15"},
            {"name": "v2.0.0", "commit": "def456ghi", "date": "2024-01-14"},
            {"name": "v1.9.9", "commit": "ghi789jkl", "date": "2024-01-10"}
        ]
    
    async def _fetch_approval_chain(self) -> List[str]:
        """Fetch approval chain for production deployments"""
        await asyncio.sleep(0.1)
        return ["tech_lead", "engineering_manager", "cto"]
```

### 6.2 Production-Ready Example Application

```python
# examples/production_review_system.py
import asyncio
import os
from dotenv import load_dotenv

from src.context.prompt_manager import PromptManager
from src.context.context_prefetcher import ContextPrefetcher
from src.tools.tool_registry import ToolRegistry
from src.tools.customer_support_tools import CreateHighPriorityTicket, RequestHumanInput
from src.state.thread_manager import ThreadManager
from src.control.agent_orchestrator import AgentOrchestrator, LLMClient
from src.agents.customer_review_agent import CustomerReviewTriageAgent
from src.triggers.multi_channel_trigger import WebhookTrigger, EmailTrigger

# Load environment variables
load_dotenv()

class ProductionReviewSystem:
    def __init__(self):
        self.setup_components()
        self.setup_agents()
        self.setup_triggers()
    
    def setup_components(self):
        """Initialize all system components"""
        # Core components
        self.prompt_manager = PromptManager("prompts")
        self.context_prefetcher = ContextPrefetcher()
        self.tool_registry = ToolRegistry()
        self.thread_manager = ThreadManager()
        self.llm_client = LLMClient(os.getenv("OPENAI_API_KEY"))
        
        # Register tools
        self.tool_registry.register(CreateHighPriorityTicket())
        self.tool_registry.register(RequestHumanInput())
        
        # Orchestrator
        self.orchestrator = AgentOrchestrator(
            self.prompt_manager,
            self.tool_registry,
            self.thread_manager,
            self.llm_client
        )
    
    def setup_agents(self):
        """Initialize micro-agents"""
        self.review_agent = CustomerReviewTriageAgent(
            self.orchestrator,
            self.thread_manager
        )
    
    def setup_triggers(self):
        """Setup multi-channel triggers"""
        self.webhook_trigger = WebhookTrigger(self.orchestrator)
        
        # Register triggers
        email_trigger = EmailTrigger(self.review_agent)
        self.webhook_trigger.register_handler("customer_email", email_trigger)
    
    async def process_review_from_email(self, email_data: dict):
        """Process review received via email"""
        print(f"📧 Processing email review from: {email_data.get('from', 'Unknown')}")
        
        result = await self.review_agent.process_review(
            review_text=email_data.get("content", ""),
            customer_id=email_data.get("from", "")
        )
        
        print(f"✅ Review processed: {result}")
        return result
    
    async def process_review_batch(self, reviews: list):
        """Process multiple reviews concurrently"""
        print(f"🔄 Processing batch of {len(reviews)} reviews...")
        
        tasks = []
        for review in reviews:
            task = self.review_agent.process_review(
                review_text=review.get("text", ""),
                customer_id=review.get("customer_id", "")
            )
            tasks.append(task)
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        successful_results = [r for r in results if not isinstance(r, Exception)]
        failed_results = [r for r in results if isinstance(r, Exception)]
        
        print(f"✅ Successfully processed: {len(successful_results)}")
        print(f"❌ Failed to process: {len(failed_results)}")
        
        return {
            "successful": successful_results,
            "failed": failed_results,
            "total": len(reviews)
        }

async def demo_production_system():
    """Demonstrate the production system"""
    system = ProductionReviewSystem()
    
    # Test individual review processing
    print("=== Single Review Processing ===")
    urgent_review = {
        "text": "The Super Juicer 5000 started sparking and smoking! This is extremely dangerous!",
        "customer_id": "customer_123@email.com"
    }
    
    result = await system.review_agent.process_review(
        urgent_review["text"],
        urgent_review["customer_id"]
    )
    print(f"Urgent review result: {result}")
    
    print("\n=== Batch Processing ===")
    # Test batch processing
    review_batch = [
        {"text": "Love this product! Works perfectly!", "customer_id": "happy_customer@email.com"},
        {"text": "Blender is too loud and doesn't blend well", "customer_id": "customer_456@email.com"},
        {"text": "Product arrived damaged, packaging was terrible", "customer_id": "customer_789@email.com"},
        {"text": "Best purchase ever! Highly recommend!", "customer_id": "customer_abc@email.com"},
        {"text": "Motor stopped working after 2 days", "customer_id": "customer_def@email.com"}
    ]
    
    batch_results = await system.process_review_batch(review_batch)
    print(f"Batch processing complete: {batch_results}")
    
    print("\n=== Email Trigger Demo ===")
    # Test email trigger
    email_data = {
        "from": "customer@example.com",
        "subject": "Product Review",
        "content": "The product is amazing but delivery was slow"
    }
    
    await system.process_review_from_email(email_data)

if __name__ == "__main__":
    asyncio.run(demo_production_system())
```

### 6.3 Monitoring and Observability

```python
# src/monitoring/agent_monitor.py
import time
import json
from typing import Dict, Any, List
from datetime import datetime, timedelta
from collections import defaultdict, deque

class AgentMetrics:
    def __init__(self):
        self.start_time = datetime.now()
        self.request_count = 0
        self.success_count = 0
        self.error_count = 0
        self.response_times = deque(maxlen=1000)  # Keep last 1000 response times
        self.error_types = defaultdict(int)
        self.agent_performance = defaultdict(lambda: {"requests": 0, "errors": 0, "avg_response_time": 0})
    
    def record_request(self, agent_type: str, response_time: float, success: bool, error_type: str = None):
        """Record metrics for an agent request"""
        self.request_count += 1
        self.response_times.append(response_time)
        
        if success:
            self.success_count += 1
        else:
            self.error_count += 1
            if error_type:
                self.error_types[error_type] += 1
        
        # Agent-specific metrics
        agent_metrics = self.agent_performance[agent_type]
        agent_metrics["requests"] += 1
        if not success:
            agent_metrics["errors"] += 1
        
        # Update average response time
        current_avg = agent_metrics["avg_response_time"]
        request_count = agent_metrics["requests"]
        agent_metrics["avg_response_time"] = (current_avg * (request_count - 1) + response_time) / request_count
    
    def get_summary(self) -> Dict[str, Any]:
        """Get performance summary"""
        uptime = datetime.now() - self.start_time
        
        return {
            "uptime_seconds": uptime.total_seconds(),
            "total_requests": self.request_count,
            "success_rate": self.success_count / max(self.request_count, 1),
            "error_rate": self.error_count / max(self.request_count, 1),
            "avg_response_time": sum(self.response_times) / max(len(self.response_times), 1),
            "error_types": dict(self.error_types),
            "agent_performance": dict(self.agent_performance)
        }

class AgentMonitor:
    def __init__(self):
        self.metrics = AgentMetrics()
        self.active_threads = {}
        self.alerts = []
    
    def start_monitoring_thread(self, thread_id: str, agent_type: str):
        """Start monitoring an agent thread"""
        self.active_threads[thread_id] = {
            "agent_type": agent_type,
            "start_time": time.time(),
            "events": []
        }
    
    def record_thread_event(self, thread_id: str, event_type: str, data: Dict[str, Any] = None):
        """Record an event in a monitored thread"""
        if thread_id in self.active_threads:
            self.active_threads[thread_id]["events"].append({
                "type": event_type,
                "timestamp": time.time(),
                "data": data or {}
            })
    
    def finish_monitoring_thread(self, thread_id: str, success: bool, error_type: str = None):
        """Finish monitoring a thread and record metrics"""
        if thread_id not in self.active_threads:
            return
        
        thread_info = self.active_threads[thread_id]
        response_time = time.time() - thread_info["start_time"]
        
        self.metrics.record_request(
            agent_type=thread_info["agent_type"],
            response_time=response_time,
            success=success,
            error_type=error_type
        )
        
        # Check for performance issues and generate alerts
        self._check_performance_alerts(thread_info, response_time, success)
        
        # Clean up
        del self.active_threads[thread_id]
    
    def _check_performance_alerts(self, thread_info: Dict[str, Any], response_time: float, success: bool):
        """Check for performance issues and generate alerts"""
        agent_type = thread_info["agent_type"]
        
        # Alert on slow response times (>10 seconds for micro-agents)
        if response_time > 10:
            self.alerts.append({
                "type": "slow_response",
                "agent_type": agent_type,
                "response_time": response_time,
                "timestamp": datetime.now(),
                "severity": "warning"
            })
        
        # Alert on failures
        if not success:
            self.alerts.append({
                "type": "agent_failure",
                "agent_type": agent_type,
                "timestamp": datetime.now(),
                "severity": "error"
            })
        
        # Alert on too many events (context window issues)
        event_count = len(thread_info["events"])
        if event_count > 20:  # Micro-agents should stay under 20 events
            self.alerts.append({
                "type": "context_overflow",
                "agent_type": agent_type,
                "event_count": event_count,
                "timestamp": datetime.now(),
                "severity": "warning"
            })
    
    def get_health_status(self) -> Dict[str, Any]:
        """Get current system health status"""
        metrics = self.metrics.get_summary()
        recent_alerts = [a for a in self.alerts if datetime.now() - a["timestamp"] < timedelta(hours=1)]
        
        # Determine overall health
        error_rate = metrics["error_rate"]
        avg_response_time = metrics["avg_response_time"]
        
        if error_rate > 0.1 or avg_response_time > 15:
            health_status = "unhealthy"
        elif error_rate > 0.05 or avg_response_time > 10:
            health_status = "degraded"
        else:
            health_status = "healthy"
        
        return {
            "status": health_status,
            "metrics": metrics,
            "active_threads": len(self.active_threads),
            "recent_alerts": len(recent_alerts),
            "alerts": recent_alerts[-10:]  # Last 10 alerts
        }

# Global monitor instance
agent_monitor = AgentMonitor()
```

### 6.4 Configuration Management

```python
# src/config/agent_config.py
from pydantic import BaseModel, Field
from typing import Dict, Any, List, Optional
import yaml
import os

class AgentConfig(BaseModel):
    name: str
    max_steps: int = 10
    timeout_seconds: int = 300
    retry_attempts: int = 3
    requires_human_approval: bool = False
    high_stakes_tools: List[str] = Field(default_factory=list)

class SystemConfig(BaseModel):
    openai_api_key: str
    max_concurrent_agents: int = 10
    context_window_size: int = 8000
    enable_monitoring: bool = True
    enable_caching: bool = True
    log_level: str = "INFO"
    
    # Agent configurations
    agents: Dict[str, AgentConfig] = Field(default_factory=dict)
    
    # Tool configurations
    tools: Dict[str, Dict[str, Any]] = Field(default_factory=dict)

class ConfigManager:
    def __init__(self, config_path: str = "config/agents.yaml"):
        self.config_path = config_path
        self.config: Optional[SystemConfig] = None
        self.load_config()
    
    def load_config(self):
        """Load configuration from file and environment"""
        config_data = {}
        
        # Load from YAML file if exists
        if os.path.exists(self.config_path):
            with open(self.config_path, 'r') as f:
                config_data = yaml.safe_load(f)
        
        # Override with environment variables
        config_data.update({
            "openai_api_key": os.getenv("OPENAI_API_KEY", ""),
            "max_concurrent_agents": int(os.getenv("MAX_CONCURRENT_AGENTS", "10")),
            "log_level": os.getenv("LOG_LEVEL", "INFO")
        })
        
        self.config = SystemConfig(**config_data)
    
    def get_agent_config(self, agent_name: str) -> AgentConfig:
        """Get configuration for specific agent"""
        if agent_name in self.config.agents:
            return self.config.agents[agent_name]
        
        # Return default config
        return AgentConfig(name=agent_name)
    
    def update_agent_config(self, agent_name: str, config: AgentConfig):
        """Update agent configuration"""
        self.config.agents[agent_name] = config
        self.save_config()
    
    def save_config(self):
        """Save current configuration to file"""
        config_dict = self.config.dict()
        
        # Remove sensitive data before saving
        config_dict.pop("openai_api_key", None)
        
        os.makedirs(os.path.dirname(self.config_path), exist_ok=True)
        with open(self.config_path, 'w') as f:
            yaml.dump(config_dict, f, default_flow_style=False)
```

Create configuration file:
```yaml
# config/agents.yaml
max_concurrent_agents: 10
context_window_size: 8000
enable_monitoring: true
enable_caching: true
log_level: "INFO"

agents:
  customer_review_triage:
    name: "customer_review_triage"
    max_steps: 5
    timeout_seconds: 60
    retry_attempts: 2
    requires_human_approval: false
    high_stakes_tools: []
  
  deployment_workflow:
    name: "deployment_workflow" 
    max_steps: 15
    timeout_seconds: 600
    retry_attempts: 1
    requires_human_approval: true
    high_stakes_tools: ["deploy_to_production", "rollback_deployment"]

tools:
  create_high_priority_ticket:
    enabled: true
    rate_limit: 100  # per hour
    requires_auth: true
  
  request_human_input:
    enabled: true
    channels: ["slack", "email"]
    timeout_minutes: 30
```

## Learning Path & Implementation Guide

### **Week 1-2: Foundation & Setup**
```bash
# 1. Set up development environment
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 2. Create basic project structure
mkdir -p src/{agents,context,tools,control,state,types} prompts tests examples config

# 3. Implement core types and basic components
# Start with: agent_types.py, prompt_manager.py, context_builder.py
```

**Learning Goals:**
- [ ] Understand the 12-Factor principles conceptually
- [ ] Set up a clean, testable project structure
- [ ] Implement basic prompt management and context building
- [ ] Create your first structured prompt template

### **Week 3-4: Core Agent Mechanics**
```bash
# 1. Implement tool registry and basic tools
# 2. Build control flow orchestrator
# 3. Create thread management system
# 4. Test with simple examples
```

**Learning Goals:**
- [ ] Master Factor 4: Tools as structured outputs
- [ ] Understand Factor 8: Own your control flow
- [ ] Implement Factor 5: Unified state management
- [ ] Build and test your first micro-agent

### **Week 5-6: Production Features**
```bash
# 1. Add error handling and self-healing
# 2. Implement human-in-the-loop patterns
# 3. Build safety controls for high-stakes operations
# 4. Add pause/resume functionality
```

**Learning Goals:**
- [ ] Master Factor 9: Error compaction and self-healing
- [ ] Implement Factor 7: Human interaction as tool calls
- [ ] Build production-grade error handling
- [ ] Add comprehensive testing

### **Week 7-8: Micro-Agent Development**
```bash
# 1. Create multiple small, focused agents
# 2. Implement stateless reducer patterns
# 3. Build agent composition patterns
# 4. Add multi-channel triggering
```

**Learning Goals:**
- [ ] Master Factor 10: Small, focused agents
- [ ] Implement Factor 12: Stateless reducer pattern
- [ ] Build Factor 11: Multi-channel triggering
- [ ] Create a library of reusable micro-agents

### **Week 9-10: Advanced Features**
```bash
# 1. Implement context pre-fetching
# 2. Add monitoring and observability
# 3. Build configuration management
# 4. Optimize for production deployment
```

**Learning Goals:**
- [ ] Advanced context engineering techniques
- [ ] Production monitoring and alerting
- [ ] Performance optimization
- [ ] Configuration-driven agent behavior

### **Week 11-12: Real-World Integration**
```bash
# 1. Deploy to production environment
# 2. Integrate with existing systems
# 3. Add comprehensive logging and metrics
# 4. Implement continuous improvement processes
```

**Learning Goals:**
- [ ] Production deployment and DevOps
- [ ] Real-world system integration
- [ ] Performance monitoring and optimization
- [ ] Continuous improvement workflows

## Key Success Metrics & Best Practices

### **Reliability Metrics**
- **Agent Success Rate**: > 95% for well-defined tasks
- **Context Window Management**: No performance degradation up to max context
- **Error Recovery**: > 90% successful self-healing from recoverable errors
- **Human Integration**: < 2 second response time for approval requests

### **Development Best Practices**

1. **Start Small**: Begin with one micro-agent, perfect it, then expand
2. **Test Everything**: Every component should have comprehensive tests
3. **Monitor Everything**: Implement monitoring from day one
4. **Version Control Prompts**: Treat prompts as first-class code
5. **Optimize Context**: Spend time on context engineering - it's the most critical skill

### **Production Readiness Checklist**

- [ ] All prompts version-controlled and tested
- [ ] Error handling with structured recovery
- [ ] Human approval flows for high-stakes operations  
- [ ] Monitoring and alerting configured
- [ ] Configuration management implemented
- [ ] Comprehensive test suite with >80% coverage
- [ ] Performance benchmarks established
- [ ] Documentation for all agents and tools

## Common Pitfalls to Avoid

1. **Making Agents Too Large**: Keep them focused (3-10 steps max)
2. **Ignoring Context Engineering**: This is more important than the LLM choice
3. **No Human Oversight**: High-stakes operations need human approval
4. **Poor Error Handling**: Errors should teach the agent, not break it
5. **Framework Lock-in**: Own your prompts, context, and control flow
6. **No Monitoring**: You can't improve what you don't measure

## Resources for Continued Learning

- **12-Factor Agents Documentation**: Reference implementation and patterns
- **Context Engineering**: Advanced techniques for optimizing LLM inputs
- **Production AI**: Best practices for deploying AI systems at scale
- **Testing AI Systems**: Strategies for testing non-deterministic systems

Remember: The goal is not to build the most sophisticated agent, but the most **reliable** one that solves real problems in production environments. Focus on the fundamentals, test extensively, and iterate based on real-world feedback.