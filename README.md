# AI on Cloudflare Workers – A Style Guide

A **style guide** for building Cloudflare Worker projects that make use of:

- [Hono](https://hono.dev/) for routing
- [Workers AI Provider](https://www.npmjs.com/package/workers-ai-provider) for LLM interactions
- Durable Objects for stateful concurrency
- Workflows for multi-step orchestration

Adhering to these guidelines ensures consistency, maintainability, and clarity for teams creating Worker-based AI applications or standard data-handling workflows.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Libraries](#2-core-libraries)
3. [Basic Project Layout](#3-basic-project-layout)
4. [HTTP Handling with Hono](#4-http-handling-with-hono)
5. [LLM Interaction and AI Patterns](#5-llm-interaction-and-ai-patterns)
6. [Tool Usage Patterns](#6-tool-usage-patterns)
7. [Durable Objects](#7-durable-objects)
   - [How to Add a Durable Object](#71-how-to-add-a-durable-object)
8. [Workflows](#8-workflows)
   - [How to Add a Workflow](#81-how-to-add-a-workflow)
   - [Step-by-Step Patterns](#82-step-by-step-patterns)
9. [Parallel Calls and Aggregation](#9-parallel-calls-and-aggregation)
   - [Common Architecture Patterns](#91-common-architecture-patterns)
11. [Configuration Files and Deployment](#10-configuration-files-and-deployment)
12. [Style Conventions and Coding Tips](#11-style-conventions-and-coding-tips)

---

## 1. Overview

This style guide focuses on:

- Clear, maintainable TypeScript in Cloudflare Workers.
- Consistent patterns for HTTP routes, AI usage, and orchestration logic.
- Unified code structure for Workflows, Durable Objects, and LLM usage.

Each application should have a straightforward entry point (`src/index.ts`) plus a `wrangler.jsonc` or `wrangler.toml` that sets up compatibility dates, environment bindings, and so on.

---

## 2. Core Libraries

### 2.1 [Hono](https://github.com/honojs/hono)

- Lightweight router for Cloudflare Workers.
- Offers a succinct API for typical REST endpoints.
- Often used with:
  ```ts
  import { Hono } from "hono";
  import { cors } from "hono/cors";

  const app = new Hono();
  app.use(cors());
  ```

### 2.2 [Workers AI Provider](https://www.npmjs.com/package/workers-ai-provider)

- A standard library for calling large language models (LLMs) and other AI tasks from Cloudflare Workers.
- Generally integrated with:
  ```ts
  import { createWorkersAI } from "workers-ai-provider";
  ```
- Supports both **non-streaming** (`generateText()`) and **streaming** (`streamText()`) patterns.

### 2.3 [Zod](https://github.com/colinhacks/zod) (Optional)

- Used for schema validation or structured LLM outputs.
- Integrates well for “tool” usage or structured JSON extraction from LLMs.

### 2.4 Durable Objects & Workflows

- [Durable Objects](https://developers.cloudflare.com/durable-objects/) for concurrency and state.
- [Workflows](https://developers.cloudflare.com/workflows/) for multi-step orchestration, advanced scheduling, parallel calls, etc.

---

## 3. Basic Project Layout

A typical layout:

```
my-worker/
  ├─ wrangler.jsonc         // Worker config
  ├─ package.json           // Dependencies
  ├─ src/
  │   ├─ index.ts           // Main entry point
  │   ├─ types/             // (Optional) custom type definitions
  │   ├─ llm/ or utils/     // (Optional) AI logic or utility modules
  └─ ...
```

**Key points**:

- `wrangler.jsonc` or `.toml` (if you're stuck in 2024) sets `compatibility_date`, environment bindings, and any special flags.
- `src/index.ts` sets up your Hono `app`, routes, and environment binding usage.

---

## 4. HTTP Handling with Hono

To handle HTTP routes:

1. **Create an app**:
   ```ts
   import { Hono } from "hono";
   import { cors } from "hono/cors";

   const app = new Hono();
   app.use(cors());
   ```

2. **Add routes**:
   ```ts
   app.post("/my-route", async (c) => {
     const body = await c.req.json();
     // Perform logic
     return c.json({ success: true });
   });
   ```

3. **Export**:
   ```ts
   export default {
     fetch: app.fetch,
   } satisfies ExportedHandler<Env>;
   ```

**CORS**: Usually apply `cors()` at the top level if you expect browser requests from other origins.

---

## 5. LLM Interaction and AI Patterns

When using an AI binding, configure `wrangler.jsonc` with:

```jsonc
{
  "compatibility_date": "...",
  "ai": { "binding": "AI" },
  // ...
}
```

Then, in your Worker:

```ts
import { generateText, streamText } from "ai";
import { createWorkersAI } from "workers-ai-provider";

app.post("/generate", async (c) => {
  // parse user input
  const { prompt } = await c.req.json();

  // Connect to the AI binding
  const workersai = createWorkersAI({ binding: c.env.AI });
  const model = workersai("@cf/meta/llama-3.3-70b-instruct-fp8-fast");

  // Non-stream usage
  const result = await generateText({
    model,
    prompt,
  });
  return c.json(result);
});
```

**Streaming**:

```ts
app.post("/stream", async (c) => {
  const { prompt } = await c.req.json();
  const workersai = createWorkersAI({ binding: c.env.AI });
  const model = workersai("@cf/meta/llama-3.3-70b-instruct-fp8-fast");

  const streamed = streamText({ model, prompt });
  return streamed.toDataStreamResponse({
    headers: {
      "Content-Type": "text/x-unknown",
      "transfer-encoding": "chunked",
      "content-encoding": "identity",
    },
  });
});
```

**Response**: For streaming, you generally need chunked transfer encoding plus an identity content-encoding.

---

## 6. Tool Usage Patterns

When enabling an LLM to call “tools”, you:

1. **Define** a tool, typically validated via Zod (optional):
   ```ts
   import { tool } from "ai";
   import z from "zod";

   const weatherTool = tool({
     description: "Get the weather in a location",
     parameters: z.object({
       location: z.string(),
     }),
     execute: async ({ location }) => {
       // fetch or compute weather
       return { weather: "Sunny", location };
     },
   });
   ```

2. **Register** the tool in an LLM call:
   ```ts
   const result = await generateText({
     model,
     prompt: "What's the weather in London?",
     tools: { weather: weatherTool },
     maxSteps: 5,
   });
   ```

3. The LLM can call the tool with JSON input, after which the library merges the tool response into the conversation.

**Manual approach**:  
Alternatively, you can parse “tool usage” from the LLM yourself. This generally means:

- Prompt the LLM for a JSON describing the tool call.
- If the call is recognized, run the tool server-side.
- Provide the result back into the conversation.

But the built-in pattern from `ai` library simplifies that flow.

---

## 7. Durable Objects

Durable Objects provide shared, consistent state across multiple Workers. This is helpful when you need concurrency, scheduling, or stateful management.

### 7.1 How to Add a Durable Object

1. **Create** a class extending `DurableObject`:
   ```ts
   import { DurableObject, DurableObjectState } from "cloudflare:workers";

   export class MyDO extends DurableObject {
     constructor(protected state: DurableObjectState, protected env: Env) {
       super();
     }

     async fetch(request: Request) {
       // handle requests to this DO
       // you can store or retrieve state
       // e.g. this.state.storage.get(...)
       return new Response("Hello from DO");
     }
   }
   ```

2. **Declare** in `wrangler.jsonc` under `durable_objects`:
   ```jsonc
   {
     "durable_objects": {
       "bindings": [
         {
           "name": "MY_DO",
           "class_name": "MyDO"
         }
       ]
     }
   }
   ```

3. **Use** in your code:
   ```ts
   app.post("/init-do", async (c) => {
     const id = c.env.MY_DO.newUniqueId();
     const stub = c.env.MY_DO.get(id);
     const response = await stub.fetch("https://do-logic/");
     return response;
   });
   ```

**State** is managed via `this.state.storage` with synchronous or transactional methods.

---

## 8. Workflows

Workflows let you orchestrate multi-step tasks with:

- Automatic retries
- Step-based concurrency
- Waiting or sleeping mid-flow
- Parallel calls

### 8.1 How to Add a Workflow

1. In `wrangler.jsonc`, add:
   ```jsonc
   {
     "workflows": [
       {
         "binding": "MY_WORKFLOW",
         "class_name": "MyWorkflow"
       }
     ]
   }
   ```
2. **Create** a class extending `WorkflowEntrypoint`:
   ```ts
   import {
     WorkflowEntrypoint,
     type WorkflowEvent,
     type WorkflowStep,
   } from "cloudflare:workers";

   export class MyWorkflow extends WorkflowEntrypoint<Env, MyParams> {
     async run(event: WorkflowEvent<MyParams>, step: WorkflowStep) {
       // step logic
       // example usage:
       const result = await step.do("some-step", async () => {
         // do something
         return { success: true };
       });
       return result;
     }
   }
   ```

3. **Create** or **get** a workflow instance at runtime:
   ```ts
   app.post("/start-workflow", async (c) => {
     const instance = await c.env.MY_WORKFLOW.create({
       params: { userId: "abc" },
     });
     return c.json({ id: instance.id });
   });
   ```

### 8.2 Step-by-Step Patterns

Within `run(event, step)`:

- `await step.do("description", async () => { ... })` for each significant action.
- `await step.sleep("label", "5 seconds")` to pause.
- Return aggregated data at the end.

Use this for chaining LLM calls, external fetches, parallel tasks, etc.

---

## 9. Parallel Calls and Aggregation

Sometimes you want to run multiple tasks in parallel (e.g., multiple smaller LLM calls) then aggregate them:

1. **Parallel**:
   ```ts
   const results = await Promise.all([
     smallModelCall("angle1"),
     smallModelCall("angle2"),
   ]);
   ```
2. **Aggregate**:
   ```ts
   const final = await bigModelCall(`Combine the results: ${results.join(" - ")}`);
   return final;
   ```

Inside a Workflow, nest the `Promise.all` inside `step.do(...)` for a named step.

---


## 9.1 Common Architecture Patterns

These **common patterns** demonstrate architectures tested across AI Worker deployments.

### Structured Outputs

When you need predictable, typed output from LLMs:

```ts
app.post("/", async (c) => {
  const workersai = createWorkersAI({ binding: c.env.AI });
  const { object } = await generateObject({
    model: workersai("@cf/meta/llama-3.3-70b-instruct-fp8-fast"),
    schema: z.object({
      recipe: z.object({
        name: z.string(),
        ingredients: z.array(z.object({ 
          name: z.string(), 
          amount: z.string() 
        })),
        steps: z.array(z.string()),
      }),
    }),
    prompt: "Give me a recipe for sourdough bread.",
  });
  return c.json(object);
});
```

### Prompt Chaining

For multi-stage prompting with dependencies between steps:

```ts
class PromptChainingWorkflow extends WorkflowEntrypoint {
  async run(event: WorkflowEvent<{ prompt: string }>, step: WorkflowStep) {
    // Step 1: Generate outline
    const outlineObj = await step.do("generate outline", async () => {
      return await generateObject({
        model,
        schema: outlineSchema,
        prompt: `${prompt}\n\nPlease generate a detailed outline.`,
      });
    });

    // Step 2: Evaluate outline
    const criteriaObj = await step.do("evaluate outline", async () => {
      return await generateObject({
        model,
        schema: criteriaSchema,
        prompt: `Evaluate this outline:\n\n${JSON.stringify(outlineObj)}`,
      });
    });

    // Step 3: Generate final if criteria met
    if (criteriaObj.meetsCriteria) {
      return await step.do("generate final", async () => {
        return await generateObject({
          model,
          schema: finalSchema,
          prompt: `Using this outline:\n\n${JSON.stringify(outlineObj)}`,
        });
      });
    }
  }
}
```

### Parallelisation

When multiple perspectives or angles are needed:

```ts
class ParallelisationWorkflow extends WorkflowEntrypoint {
  async run(event: WorkflowEvent<{ prompt: string }>, step: WorkflowStep) {
    // Run parallel analysis with smaller models
    const angleOutputs = await step.do("parallel angle calls", async () => {
      const calls = [
        "technical perspective",
        "creative perspective", 
        "user-centric perspective"
      ].map(async (angle) => {
        const { object } = await generateObject({
          model: smallModel,
          schema: angleSchema,
          prompt: `Analyze from ${angle}:\n\n${prompt}`,
        });
        return object.output;
      });
      return Promise.all(calls);
    });

    // Aggregate with larger model
    return await step.do("aggregate responses", async () => {
      return await generateObject({
        model: bigModel,
        schema: finalSchema,
        prompt: `Synthesize these perspectives:\n\n${angleOutputs.join("\n\n")}`,
      });
    });
  }
}
```

### Orchestrator Workers

For coordinating multiple specialized Workers:

```ts
class OrchestratorWorkflow extends WorkflowEntrypoint {
  async run(event: WorkflowEvent<{ prompt: string }>, step: WorkflowStep) {
    // Break task into subtasks
    const orchestratorResult = await step.do("generate subtasks", async () => {
      return await generateObject({
        model: bigModel,
        schema: orchestratorSchema,
        prompt: `Break this task into subtasks:\n\n${prompt}`,
      });
    });

    // Execute parallel subtasks
    const workerResponses = await step.do("execute subtasks", async () => {
      return Promise.all(orchestratorResult.tasks.map(async (taskPrompt) => {
        const { object } = await generateObject({
          model: smallModel,
          schema: workerSchema,
          prompt: taskPrompt,
        });
        return object.response;
      }));
    });

    // Synthesize final response
    return await step.do("synthesize responses", async () => {
      return await generateObject({
        model: bigModel,
        schema: finalSchema,
        prompt: `Synthesize subtask results:\n\n${workerResponses.join("\n")}`,
      });
    });
  }
}
```

### Evaluator-Optimiser

For iterative improvement with quality checks:

```ts
class EvaluatorOptimiserWorkflow extends WorkflowEntrypoint {
  async run(event: WorkflowEvent<{ prompt: string }>, step: WorkflowStep) {
    // Initial draft
    const draftObj = await step.do("generate draft", async () => {
      return await generateObject({
        model: smallModel,
        schema: draftSchema,
        prompt: event.payload.prompt,
      });
    });

    // Evaluate draft
    const evaluationObj = await step.do("evaluate draft", async () => {
      return await generateObject({
        model: smallModel,
        schema: evaluationSchema,
        prompt: `Evaluate this draft:\n\n${draftObj.draft}`,
      });
    });

    // Optimize if needed
    if (evaluationObj.needsRevision) {
      return await step.do("optimize draft", async () => {
        return await generateObject({
          model: bigModel,
          schema: optimizedSchema,
          prompt: `Improve based on:\nDraft: ${draftObj.draft}\nFeedback: ${evaluationObj.feedback}`,
        });
      });
    }

    return draftObj;
  }
}
```

### Human-in-the-Loop

When human confirmation is needed before actions:

```ts
export class TaskManagerAgent extends Agent<{ AI: any }, TaskManagerState> {
  initialState = {
    tasks: [],
    confirmations: []
  };

  async query(query: string) {
    const { action } = await this.determineAction(query);

    if (action === "add") {
      const newConfirmation = {
        id: crypto.randomUUID(),
        action: "add",
        title: await this.extractTitle(query),
      };

      this.setState({
        ...this.state,
        confirmations: [...this.state.confirmations, newConfirmation],
      });

      return { confirmation: newConfirmation };
    }
  }

  async confirm(confirmationId: string, userConfirmed: boolean) {
    const confirmation = this.state.confirmations.find(c => c.id === confirmationId);

    if (userConfirmed && confirmation?.action === "add") {
      const result = this.addTask(confirmation.title);
      this.removeConfirmation(confirmationId);
      return result;
    }

    this.removeConfirmation(confirmationId);
    return "Action not confirmed";
  }
}
```

These patterns can be mixed and matched based on your specific needs. For example, you might use parallelisation within an evaluator-optimiser workflow, or add human-in-the-loop confirmation to an orchestrator pattern.

### Agentic Patterns

For more advanced cases where an LLM acts as an autonomous agent:

```ts
// Planning pattern - break tasks into subtasks
async function planAndExecute(task: string) {
  // First plan subtasks
  const plan = await generateObject({
    model,
    schema: z.object({
      subtasks: z.array(z.object({
        id: z.string(),
        task: z.string(),
        dependencies: z.array(z.string())
      }))
    }),
    prompt: `Break this task into subtasks: ${task}`
  });

  // Then execute in dependency order
  const results = new Map();
  for (const subtask of orderByDependencies(plan.subtasks)) {
    results.set(subtask.id, await executeSubtask(subtask, results));
  }
  return results;
}

// ReAct pattern - Reason, Act, Observe loop
async function reactLoop(goal: string) {
  let state = { goal, observations: [] };

  while (!goalAchieved(state)) {
    // Reason about current state
    const reasoning = await generateObject({
      model,
      schema: z.object({
        thought: z.string(),
        action: z.object({
          tool: z.string(),
          args: z.record(z.any())
        })
      }),
      prompt: `Goal: ${state.goal}\nObservations: ${state.observations.join("\n")}\nWhat should we do next?`
    });

    // Act using selected tool
    const result = await executeTool(reasoning.action);

    // Observe and update state
    state.observations.push(result);
  }

  return state;
}

// Reflection pattern - periodically assess and adjust
class ReflectiveAgent extends Agent {
  async act(input: string) {
    // Regular action
    const result = await this.executeAction(input);

    // Periodic reflection
    if (this.shouldReflect()) {
      const reflection = await generateObject({
        model,
        schema: z.object({
          assessment: z.string(),
          improvements: z.array(z.string())
        }),
        prompt: `Review recent actions:\n${this.getHistory()}\nWhat could be improved?`
      });

      // Update behavior based on reflection
      await this.adjustStrategy(reflection.improvements);
    }

    return result;
  }
}

// Scheduling pattern - manage time-based tasks
class SchedulerAgent extends Agent {
  async scheduleTask(description: string) {
    const schedule = await generateObject({
      model,
      schema: z.object({
        timing: z.object({
          type: z.enum(["immediate", "delayed", "recurring"]),
          value: z.string() // ISO date or cron expression
        }),
        priority: z.number(),
        dependencies: z.array(z.string())
      }),
      prompt: `When should we schedule: ${description}`
    });

    return this.addToSchedule({
      task: description,
      timing: schedule.timing,
      priority: schedule.priority
    });
  }
}
```

These patterns enable more sophisticated agent behaviors:

- **Planning**: Breaking complex tasks into manageable subtasks
- **ReAct**: Iterative cycle of Reasoning, Acting, and Observing 
- **Reflection**: Self-assessment and strategy adjustment
- **Scheduling**: Time-aware task management

Consider implementing these when your agent needs to:

- Handle multi-step tasks autonomously
- Learn from past interactions
- Manage complex schedules
- Make reasoned decisions about tool usage

The patterns can be combined - for example, a ReAct loop could include reflection steps, or a planner could work with a scheduler for time-sensitive tasks.

## 10. Configuration Files and Deployment

Typical deployment is done with:

- **wrangler.jsonc** or `.toml` for environment definitions:
  ```jsonc
  {
    "$schema": "https://raw.githubusercontent.com/cloudflare/workers-sdk/master/packages/wrangler/schemas/config-schema.json",
    "name": "my-worker",
    "main": "src/index.ts",
    "compatibility_date": "2025-01-29",
    "ai": { "binding": "AI" },
    "durable_objects": { ... },
    "workflows": [ ... ],
    "observability": { "enabled": true }
  }
  ```
- **package.json** for dependencies and scripts (`dev`, `build`, etc.).
- Use `wrangler deploy` to push your code.

---

## 11. Style Conventions and Coding Tips

- **Prefer TypeScript** for type safety. Include a `tsconfig.json`.
- **Hono** routing in `index.ts`:
  ```ts
  export default {
    fetch: app.fetch,
  };
  ```
- **Use** `async/await` for clarity.
- Keep route handlers minimal, with business logic in separate modules if needed.
- For streaming endpoints, always set the chunked `transfer-encoding`.
- For AI calls, keep the prompt or messages array well-defined. 
- Tools are optional but help keep logic modular.

**In summary**, maintain a consistent structure, keep code minimal and typed, and leverage Cloudflare’s features (Durable Objects, Workflows, AI binding) in standard ways. This uniform approach will simplify development and collaboration on Worker-based projects.
