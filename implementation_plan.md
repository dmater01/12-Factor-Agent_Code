# 12-Factor AI Agent Architecture: Technical Implementation Plan

## Phase 1: Foundation Setup (Week 1-2)

### 1.1 Environment & Dependencies Setup

```bash
# Core dependencies
npm init -y
npm install --save \
  openai \
  typescript \
  @types/node \
  uuid \
  zod \
  dotenv

# Development dependencies
npm install --save-dev \
  @types/jest \
  jest \
  ts-node \
  nodemon
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
│   └── types/           # TypeScript definitions
├── prompts/             # Version-controlled prompts
├── tests/               # Testing framework
└── examples/            # Implementation examples
```

### 1.3 Core Types Definition

```typescript
// src/types/agent.ts
export interface AgentEvent {
  type: string;
  data: any;
  timestamp: Date;
  id: string;
}

export interface AgentThread {
  id: string;
  events: AgentEvent[];
  status: 'running' | 'paused' | 'completed' | 'error';
  metadata: Record<string, any>;
}

export interface ToolCall {
  tool: string;
  parameters: Record<string, any>;
  id: string;
}

export interface ToolResult {
  success: boolean;
  data?: any;
  error?: string;
  metadata?: Record<string, any>;
}
```

## Phase 2: Implement Core Factors (Week 2-4)

### 2.1 Factor 2: Own Your Prompts

Create a prompt management system:

```typescript
// src/context/prompt-manager.ts
import fs from 'fs/promises';
import path from 'path';

export class PromptManager {
  private prompts: Map<string, string> = new Map();
  
  async loadPrompt(name: string): Promise<string> {
    if (this.prompts.has(name)) {
      return this.prompts.get(name)!;
    }
    
    const promptPath = path.join(__dirname, '../../prompts', `${name}.txt`);
    const prompt = await fs.readFile(promptPath, 'utf-8');
    this.prompts.set(name, prompt);
    return prompt;
  }
  
  renderPrompt(template: string, variables: Record<string, any>): string {
    return template.replace(/\{\{([^}]+)\}\}/g, (match, key) => {
      return variables[key?.trim()] || match;
    });
  }
}
```

Create your first prompt template:
```txt
# prompts/customer_review_triage.txt
You are an expert customer service analyst. Your job is to analyze customer reviews and categorize them for appropriate action.

Current date: {{current_date}}

Here's what happened so far:
{{thread_events}}

Analyze the following review and respond with a JSON object:

<user_review>
{{review_text}}
</user_review>

Respond only with valid JSON in this format:
{
  "category": "urgent_action_required" | "feedback" | "praise" | "complaint",
  "sentiment": "positive" | "negative" | "neutral",
  "urgency": 1-5,
  "extracted_entities": {
    "product_name": string,
    "order_id": string | null,
    "issue_type": string
  },
  "suggested_next_action": "create_high_priority_ticket" | "route_to_support" | "thank_customer" | "no_action"
}
```

### 2.2 Factor 3: Own Your Context Window

Implement custom context formatting:

```typescript
// src/context/context-builder.ts
export class ContextBuilder {
  private events: AgentEvent[] = [];
  
  addEvent(event: AgentEvent): void {
    this.events.push(event);
  }
  
  buildXMLContext(): string {
    let context = '';
    
    for (const event of this.events) {
      context += `<${event.type}>\n`;
      
      if (typeof event.data === 'object') {
        context += this.formatObjectAsYAML(event.data);
      } else {
        context += event.data;
      }
      
      context += `\n</${event.type}>\n\n`;
    }
    
    return context.trim();
  }
  
  private formatObjectAsYAML(obj: any): string {
    return JSON.stringify(obj, null, 2);
  }
  
  compactErrors(): void {
    // Remove resolved errors, keep only recent failures
    const recentEvents = this.events.slice(-10);
    this.events = recentEvents;
  }
}
```

### 2.3 Factor 4: Tools Are Just Structured Outputs

Create tool definition system:

```typescript
// src/tools/tool-registry.ts
import { z } from 'zod';

export interface Tool {
  name: string;
  description: string;
  schema: z.ZodSchema;
  execute: (params: any) => Promise<ToolResult>;
}

export class ToolRegistry {
  private tools: Map<string, Tool> = new Map();
  
  register(tool: Tool): void {
    this.tools.set(tool.name, tool);
  }
  
  async execute(toolCall: ToolCall): Promise<ToolResult> {
    const tool = this.tools.get(toolCall.tool);
    if (!tool) {
      return {
        success: false,
        error: `Tool ${toolCall.tool} not found`
      };
    }
    
    try {
      const validatedParams = tool.schema.parse(toolCall.parameters);
      return await tool.execute(validatedParams);
    } catch (error) {
      return {
        success: false,
        error: `Tool execution failed: ${error instanceof Error ? error.message : 'Unknown error'}`
      };
    }
  }
  
  getToolDefinitions(): Array<{name: string, description: string, schema: any}> {
    return Array.from(this.tools.values()).map(tool => ({
      name: tool.name,
      description: tool.description,
      schema: tool.schema
    }));
  }
}
```

Example tool implementation:
```typescript
// src/tools/customer-support-tools.ts
import { z } from 'zod';

export const createHighPriorityTicket: Tool = {
  name: 'create_high_priority_ticket',
  description: 'Creates a high priority support ticket',
  schema: z.object({
    title: z.string(),
    description: z.string(),
    customer_id: z.string().optional(),
    product: z.string().optional()
  }),
  execute: async (params) => {
    // Simulate ticket creation
    const ticketId = `TICKET-${Date.now()}`;
    console.log(`Created high priority ticket: ${ticketId}`, params);
    
    return {
      success: true,
      data: { ticketId, status: 'created' }
    };
  }
};
```

## Phase 3: Control Flow & State Management (Week 4-6)

### 3.1 Factor 5: Unify Execution State and Business State

```typescript
// src/state/thread-manager.ts
export class ThreadManager {
  private threads: Map<string, AgentThread> = new Map();
  
  createThread(initialEvent: AgentEvent): AgentThread {
    const thread: AgentThread = {
      id: `thread-${Date.now()}`,
      events: [initialEvent],
      status: 'running',
      metadata: {}
    };
    
    this.threads.set(thread.id, thread);
    return thread;
  }
  
  addEvent(threadId: string, event: AgentEvent): void {
    const thread = this.threads.get(threadId);
    if (thread) {
      thread.events.push(event);
    }
  }
  
  pauseThread(threadId: string): void {
    const thread = this.threads.get(threadId);
    if (thread) {
      thread.status = 'paused';
    }
  }
  
  resumeThread(threadId: string): void {
    const thread = this.threads.get(threadId);
    if (thread) {
      thread.status = 'running';
    }
  }
  
  // Serialize thread for persistence
  serializeThread(threadId: string): string {
    const thread = this.threads.get(threadId);
    return thread ? JSON.stringify(thread) : '';
  }
  
  // Deserialize thread from storage
  deserializeThread(data: string): AgentThread {
    return JSON.parse(data);
  }
}
```

### 3.2 Factor 8: Own Your Control Flow

```typescript
// src/control/agent-orchestrator.ts
export class AgentOrchestrator {
  constructor(
    private promptManager: PromptManager,
    private toolRegistry: ToolRegistry,
    private threadManager: ThreadManager,
    private llmClient: OpenAI
  ) {}
  
  async runAgent(
    threadId: string, 
    agentType: string,
    maxSteps: number = 10
  ): Promise<void> {
    const thread = this.threadManager.threads.get(threadId);
    if (!thread || thread.status !== 'running') {
      throw new Error('Thread not found or not running');
    }
    
    let steps = 0;
    let consecutiveErrors = 0;
    
    while (steps < maxSteps && thread.status === 'running') {
      try {
        // Build context
        const context = new ContextBuilder();
        thread.events.forEach(event => context.addEvent(event));
        
        // Get next step from LLM
        const nextStep = await this.determineNextStep(
          agentType, 
          context.buildXMLContext()
        );
        
        // Add the decision to thread
        this.threadManager.addEvent(threadId, {
          type: 'agent_decision',
          data: nextStep,
          timestamp: new Date(),
          id: `decision-${Date.now()}`
        });
        
        // Handle different intents
        if (nextStep.intent === 'done') {
          thread.status = 'completed';
          break;
        }
        
        if (nextStep.intent === 'request_human_input') {
          await this.requestHumanInput(threadId, nextStep);
          break; // Pause execution
        }
        
        // Execute tool call
        if (nextStep.tool) {
          const result = await this.executeWithSafety(nextStep, threadId);
          
          if (!result.success) {
            consecutiveErrors++;
            if (consecutiveErrors >= 3) {
              await this.escalateToHuman(threadId, result.error);
              break;
            }
          } else {
            consecutiveErrors = 0;
          }
        }
        
        steps++;
        
      } catch (error) {
        consecutiveErrors++;
        this.threadManager.addEvent(threadId, {
          type: 'error',
          data: { error: error instanceof Error ? error.message : 'Unknown error' },
          timestamp: new Date(),
          id: `error-${Date.now()}`
        });
        
        if (consecutiveErrors >= 3) {
          await this.escalateToHuman(threadId, error instanceof Error ? error.message : 'Unknown error');
          break;
        }
      }
    }
  }
  
  private async executeWithSafety(
    nextStep: any, 
    threadId: string
  ): Promise<ToolResult> {
    // Control flow interceptor - check for high-stakes operations
    if (this.isHighStakesTool(nextStep.tool)) {
      return await this.requestApprovalAndExecute(threadId, nextStep);
    }
    
    // Execute normally
    const result = await this.toolRegistry.execute({
      tool: nextStep.tool,
      parameters: nextStep.parameters,
      id: `tool-${Date.now()}`
    });
    
    // Add result to thread (Factor 9: Compact Errors)
    this.threadManager.addEvent(threadId, {
      type: result.success ? 'tool_result' : 'tool_error',
      data: result,
      timestamp: new Date(),
      id: `result-${Date.now()}`
    });
    
    return result;
  }
  
  private isHighStakesTool(toolName: string): boolean {
    const highStakesTools = [
      'issue_refund',
      'deploy_to_production',
      'delete_user_data',
      'send_external_email'
    ];
    return highStakesTools.includes(toolName);
  }
}
```

## Phase 4: Human-in-the-Loop & Error Handling (Week 6-8)

### 4.1 Factor 7: Contact Humans with Tool Calls

```typescript
// src/tools/human-interaction-tools.ts
export const requestHumanInput: Tool = {
  name: 'request_human_input',
  description: 'Request input or approval from a human',
  schema: z.object({
    question: z.string(),
    context: z.string(),
    urgency: z.enum(['low', 'medium', 'high']),
    format: z.enum(['free_text', 'yes_no', 'multiple_choice']),
    choices: z.array(z.string()).optional()
  }),
  execute: async (params) => {
    // In a real implementation, this would integrate with Slack, email, etc.
    console.log('🤖 HUMAN INPUT REQUESTED:', params);
    
    // For demo purposes, simulate approval
    return {
      success: true,
      data: {
        approved: true,
        response: 'Approved by human operator',
        timestamp: new Date().toISOString()
      }
    };
  }
};

export const escalateToHuman: Tool = {
  name: 'escalate_to_human',
  description: 'Escalate an issue to human support',
  schema: z.object({
    reason: z.string(),
    error_details: z.string().optional(),
    thread_id: z.string()
  }),
  execute: async (params) => {
    console.log('🚨 ESCALATED TO HUMAN:', params);
    
    return {
      success: true,
      data: { escalated: true, ticket_id: `ESC-${Date.now()}` }
    };
  }
};
```

### 4.2 Factor 9: Compact Errors into Context Window

```typescript
// src/control/error-handler.ts
export class ErrorHandler {
  static formatError(error: any): AgentEvent {
    return {
      type: 'structured_error',
      data: {
        error_type: error.name || 'UnknownError',
        message: error.message || 'An error occurred',
        timestamp: new Date().toISOString(),
        recoverable: true
      },
      timestamp: new Date(),
      id: `error-${Date.now()}`
    };
  }
  
  static shouldRetry(consecutiveErrors: number, maxRetries: number = 3): boolean {
    return consecutiveErrors < maxRetries;
  }
  
  static buildErrorContext(errors: AgentEvent[]): string {
    const recentErrors = errors.filter(e => e.type === 'structured_error').slice(-3);
    
    if (recentErrors.length === 0) return '';
    
    return `<recent_errors>
${recentErrors.map(error => `
  <error>
    Type: ${error.data.error_type}
    Message: ${error.data.message}
    Time: ${error.data.timestamp}
  </error>
`).join('')}
</recent_errors>`;
  }
}
```

## Phase 5: Small Focused Agents (Week 8-10)

### 5.1 Factor 10: Create Micro-Agents

```typescript
// src/agents/customer-review-agent.ts
export class CustomerReviewTriageAgent {
  constructor(
    private orchestrator: AgentOrchestrator,
    private threadManager: ThreadManager
  ) {}
  
  async processReview(reviewText: string): Promise<any> {
    // Create focused thread for this single task
    const initialEvent: AgentEvent = {
      type: 'customer_review',
      data: { review_text: reviewText },
      timestamp: new Date(),
      id: `review-${Date.now()}`
    };
    
    const thread = this.threadManager.createThread(initialEvent);
    
    // Run the focused agent (max 5 steps for this simple task)
    await this.orchestrator.runAgent(thread.id, 'customer_review_triage', 5);
    
    // Extract the result
    const decisions = thread.events.filter(e => e.type === 'agent_decision');
    const lastDecision = decisions[decisions.length - 1];
    
    return lastDecision?.data;
  }
}
```

### 5.2 Example Implementation

```typescript
// src/examples/review-processor.ts
async function demonstrateReviewProcessing() {
  // Setup
  const promptManager = new PromptManager();
  const toolRegistry = new ToolRegistry();
  const threadManager = new ThreadManager();
  const llmClient = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  
  // Register tools
  toolRegistry.register(createHighPriorityTicket);
  toolRegistry.register(requestHumanInput);
  
  const orchestrator = new AgentOrchestrator(
    promptManager, 
    toolRegistry, 
    threadManager, 
    llmClient
  );
  
  const reviewAgent = new CustomerReviewTriageAgent(orchestrator, threadManager);
  
  // Process a review
  const result = await reviewAgent.processReview(
    "The Super Juicer 5000 started sparking and smoking! This is dangerous!"
  );
  
  console.log('Review processing result:', result);
}
```

## Phase 6: Advanced Features (Week 10-12)

### 6.1 Factor 13: Context Pre-fetch

```typescript
// src/context/context-prefetch.ts
export class ContextPrefetcher {
  private cache: Map<string, any> = new Map();
  
  async prefetchLikelyContext(agentType: string, userInput: string): Promise<Record<string, any>> {
    const prefetchedData: Record<string, any> = {};
    
    // Based on agent type and input, prefetch likely needed data
    if (agentType === 'customer_review_triage') {
      // Prefetch product catalog, common issues, etc.
      prefetchedData.common_issues = await this.getCommonIssues();
      prefetchedData.current_date = new Date().toISOString();
    }
    
    if (agentType === 'deployment_agent') {
      // Prefetch git tags, deployment status, etc.
      prefetchedData.git_tags = await this.getGitTags();
      prefetchedData.deployment_status = await this.getDeploymentStatus();
    }
    
    return prefetchedData;
  }
  
  private async getCommonIssues(): Promise<string[]> {
    return ['product_defect', 'shipping_issue', 'billing_problem', 'user_error'];
  }
  
  private async getGitTags(): Promise<Array<{name: string, commit: string}>> {
    // Simulate git tag fetching
    return [
      { name: 'v1.2.3', commit: 'abc123' },
      { name: 'v1.2.2', commit: 'def456' }
    ];
  }
  
  private async getDeploymentStatus(): Promise<{environment: string, status: string}> {
    return { environment: 'staging', status: 'healthy' };
  }
}
```

### 6.2 Testing Framework

```typescript
// tests/agent.test.ts
import { describe, it, expect, beforeEach } from '@jest/globals';

describe('CustomerReviewTriageAgent', () => {
  let agent: CustomerReviewTriageAgent;
  
  beforeEach(() => {
    // Setup test environment
  });
  
  it('should categorize urgent product defect correctly', async () => {
    const result = await agent.processReview(
      "The product caught fire! This is extremely dangerous!"
    );
    
    expect(result.category).toBe('urgent_action_required');
    expect(result.urgency).toBeGreaterThan(4);
    expect(result.suggested_next_action).toBe('create_high_priority_ticket');
  });
  
  it('should handle positive feedback appropriately', async () => {
    const result = await agent.processReview(
      "Love this product! Works perfectly and great customer service!"
    );
    
    expect(result.category).toBe('praise');
    expect(result.sentiment).toBe('positive');
    expect(result.suggested_next_action).toBe('thank_customer');
  });
});
```

## Learning Path & Best Practices

### Week 1-2: Foundation
- [ ] Set up project structure
- [ ] Implement basic types and interfaces
- [ ] Create your first prompt template
- [ ] Build basic context formatting

### Week 3-4: Core Mechanics
- [ ] Implement tool registry and execution
- [ ] Build control flow orchestrator
- [ ] Create state management system
- [ ] Test with simple examples

### Week 5-6: Production Features
- [ ] Add human-in-the-loop capabilities
- [ ] Implement error handling and self-healing
- [ ] Build pause/resume functionality
- [ ] Add safety controls for high-stakes operations

### Week 7-8: Agent Development
- [ ] Create your first micro-agent
- [ ] Implement multiple small, focused agents
- [ ] Build agent composition patterns
- [ ] Add comprehensive testing

### Week 9-10: Advanced Features
- [ ] Implement context pre-fetching
- [ ] Add multi-channel triggering
- [ ] Build monitoring and observability
- [ ] Optimize for production deployment

### Week 11-12: Real-World Integration
- [ ] Deploy to production environment
- [ ] Integrate with existing systems
- [ ] Add comprehensive logging and metrics
- [ ] Implement continuous improvement processes

## Key Success Metrics

1. **Agent Reliability**: < 5% failure rate on well-defined tasks
2. **Context Management**: No degradation in performance up to max context
3. **Human Integration**: < 2 second response time for approval requests
4. **Error Recovery**: 90%+ successful self-healing from recoverable errors
5. **Maintainability**: New agents deployable within 1 day

## Next Steps

1. Start with Phase 1 foundation setup
2. Build one micro-agent end-to-end before adding complexity
3. Focus on context engineering - this is the most critical skill
4. Test extensively with real-world scenarios
5. Gradually add production features as you gain confidence

Remember: The goal is not to build the most complex agent, but the most reliable one that solves real problems in production environments.