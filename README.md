# Learning LangGraph

![Python](https://img.shields.io/badge/Python-3.9+-blue.svg)
![LangGraph](https://img.shields.io/badge/LangGraph-Agents-purple.svg)
![Gemini](https://img.shields.io/badge/Gemini-2.0%20Flash-orange.svg)
![License](https://img.shields.io/badge/License-MIT-green.svg)

Learning LangGraph hands-on including: building single agents, multi-agent architectures, and pipeline workflows across a healthcare domain.

## 🎯 TL;DR

This project explores **LangGraph for agent orchestration** across progressively complex notebooks:

- **ReAct Agent**: Single agent with 6 medical tools in a Think → Act → Observe loop.
- **Router Pattern**: Domain classifier routes to one specialist agent per query.
- **Supervisor Pattern**: Central orchestrator calls multiple specialists and synthesizes results.
- **Handoffs (Swarm) Pattern**: Agents pass control directly to each other — no central controller.
- **Sequential Pipeline with Conditional Pipelines**: Fixed step sequence with a quality gate that loops back when research is incomplete.

## 💡 Problem/Motivation

### The Agent Architecture Challenge

Building LLM-powered agents requires choosing the right orchestration pattern for each use case:

- **Single agent**: Simple but limited — one agent can't specialize.
- **Multiple agents**: Powerful but complex — how should they coordinate?
- **Pipeline workflows**: Structured but rigid — how do you add flexibility?

### The Solution

This project demonstrates how LangGraph addresses these challenges through:

- ✅ **ReAct loops** for tool-calling agents that reason step-by-step.
- ✅ **Conditional edges** that route control based on LLM decisions or state. 
- ✅ **Comparing** model versions using LangFuse Datasets and Runs.  
- ✅ **Safety mechanisms** like handoff limits and loop counters to prevent infinite loops.
- ✅ **Quality gates** that evaluate output and loop back when it's not good enough.


## 📊 Domain: Healthcare Chatbot

### Tools (6 Mock Medical Tools)

All notebooks share the same set of tools:

| Tool | Purpose | Args |
|------|---------|------|
| `drug_lookup` | Look up medication info | `drug_name` |
| `drug_interaction_checker` | Check interactions between two drugs | `drug_a, drug_b` |
| `symptom_checker` | Find possible conditions from symptoms | `symptoms` |
| `condition_info` | Get details about a medical condition | `condition` |
| `interpret_lab_result` | Interpret a lab test value | `test_name, value` |

## 📁 Project Structure

```
Learning-LangGraph/
│
├── Agent.ipynb                                # Notebook 1: ReAct Agent
├── Router.ipynb                               # Notebook 2: Router Pattern
├── Supervisor.ipynb                           # Notebook 3: Supervisor Pattern
├── HandsOff.ipynb                             # Notebook 4: Handoffs (Swarm) Pattern
├── Sequential-Conditional-Loops.ipynb         # Notebook 5: Sequential Pipeline
│
├── requirements.txt
└── README.md
```

## 🔬 Architecture Patterns

### Notebook 1: ReAct Agent (`Agent.ipynb`)

Single agent with access to all 6 tools. Follows a Think → Act → Observe loop until it has enough information to answer.

```
User Question → Agent → [Tool Call → Tool Result]* → Final Answer
```

**Key concept**: One agent, one LLM, all tools. The agent decides which tools to call and when to stop.

<img width="984" height="380" alt="Captura de ecrã 2026-03-05, às 09 16 45" src="https://github.com/user-attachments/assets/eb05f50d-b485-4dde-93c3-2fd0b7b22553" />

<img width="447" height="510" alt="Captura de ecrã 2026-03-05, às 09 17 04" src="https://github.com/user-attachments/assets/847b53b9-cea8-4cd4-abf6-b1ad628ca08b" />

---

### Notebook 2: Router (`Router.ipynb`)

A router classifies the query into a domain (medication, diagnosis, lab, general) and sends it to ONE specialist. Each specialist only sees its own tools.

```
User Question → Router (classifies domain) → Specialist Agent → Answer
```

**Key concept**: Classification once, route once. No looping back. Each specialist is isolated with its own tools via `bind_tools()`.

**State**: `{messages, domain}` — router writes the domain, conditional edge reads it.

<img width="1288" height="430" alt="Captura de ecrã 2026-03-05, às 09 17 15" src="https://github.com/user-attachments/assets/777692ab-3b3b-47aa-966d-32dc4cad4b41" />

<img width="1021" height="530" alt="Captura de ecrã 2026-03-05, às 09 17 24" src="https://github.com/user-attachments/assets/6cc6d547-a015-453d-914f-5b63ac9367ee" />

---

### Notebook 3: Supervisor (`Supervisor.ipynb`)

A supervisor orchestrates specialists as tools. It can call MULTIPLE specialists per question and synthesizes their results. The supervisor stays in control throughout.

```
User Question → Supervisor → [Specialist Call → Result]* → Synthesized Answer
```

**Key concept**: Specialists are wrapped in `@tool` decorators — each runs its own internal ReAct loop but looks like a simple function to the supervisor. The supervisor can call multiple specialists and loop until it has enough info.

**State**: `{messages}` — simpler than Router. Supervisor reasons over conversation, no classification label needed.

<img width="1072" height="520" alt="Captura de ecrã 2026-03-05, às 18 24 11" src="https://github.com/user-attachments/assets/32521cae-5721-432b-bb58-f34b1164926b" />

<img width="726" height="706" alt="Captura de ecrã 2026-03-05, às 09 17 43" src="https://github.com/user-attachments/assets/b4216180-bbe3-41ed-b0c0-1d5995c75043" />

---

### Notebook 4: Handoffs / Swarm (`HandsOff.ipynb`)

No central controller. A triage agent picks the first specialist, then agents pass control directly to each other using handoff tools.

```
User Question → Triage → Specialist A → [hand off] → Specialist B → Answer
```

**Key concept**: Peer-to-peer control transfer. Handoff tools are "fake tools" — they return a string that the routing logic reads to decide which agent gets control next. Each specialist has BOTH domain tools AND handoff tools.

**State**: `{messages, current_agent}` — tracks who currently has control.

**Safety**: `MAX_HANDOFFS = 3` prevents infinite ping-pong between agents. Counted by scanning the message history.

<img width="1320" height="598" alt="Captura de ecrã 2026-03-05, às 09 17 55" src="https://github.com/user-attachments/assets/a2bb4194-b52d-44bc-90fc-6ec8cab19be9" />

<img width="1190" height="755" alt="Captura de ecrã 2026-03-05, às 09 18 36" src="https://github.com/user-attachments/assets/d58d21bc-d144-4933-b844-47ffb651b695" />

---

### Notebook 5: Sequential Pipeline (`Sequential_Pipeline.ipynb`)

A fixed sequence of steps with a conditional quality gate that can loop back.

```
User Question → Intake → Research ⟷ Tools → Quality Check → Report → Answer
                                         ↑                    │
                                         └── (if incomplete) ──┘
```

**Key concept**: The pipeline sequence (Intake → Research → Quality → Report) is enforced by fixed edges, not by the LLM. The LLM only decides within each step.

**Two loops**:
- **Inner loop (ReAct)**: Research ⟷ research_tools — the agent calls tools until done.
- **Outer loop (Conditional)**: Quality Check → Research — the quality gate sends the case back if research is incomplete.

**State**: Richest state of all notebooks — `{messages, symptoms, drugs, lab_values, research_findings, quality_approved, quality_feedback, research_loops}`. Each step reads from previous steps and writes for the next.

**Safety**: `MAX_RESEARCH_LOOPS = 3` prevents infinite quality loops.

<img width="1376" height="531" alt="Captura de ecrã 2026-03-05, às 09 18 47" src="https://github.com/user-attachments/assets/2bb8eb95-ad64-4816-9b6c-76b14567c90e" />

<img width="496" height="823" alt="Captura de ecrã 2026-03-05, às 09 18 56" src="https://github.com/user-attachments/assets/fb81d4e0-c7bb-4a21-bf59-4432c0256843" />

## 📈 Pattern Comparison

| Pattern | Control Flow | Multi-Domain | Strengths | Weaknesses |
|---------|-------------|-------------|-----------|------------|
| **ReAct** | One agent loops | No | Simple, flexible | No specialization |
| **Router** | Classify → one agent | Yes (isolated) | Fast, clean separation | Can't combine domains |
| **Supervisor** | Central brain loops | Yes (combined) | Synthesizes multi-domain answers | Single point of failure |
| **Handoffs** | Peer-to-peer | Yes (sequential) | Autonomous, no bottleneck | Ping-pong risk, hard to debug |
| **Pipeline** | Fixed sequence + quality gate | Yes (structured) | Predictable, quality-controlled | Rigid sequence, slower |

## 🚀 Getting Started

### Installation

```bash
# Clone the repository
git clone https://github.com/pedroalexleite/Learning-LangGraph.git
cd Learning-LangGraph

# Install dependencies
pip install -r requirements.txt
```

### Setup

Create a `.env` file in the root directory:

```
GEMINI_API_KEY=your_gemini_api_key
```

### Running the Notebooks

Run in order — each notebook builds on concepts from the previous:

```bash
jupyter notebook Agent.ipynb                            # 1. ReAct basics
jupyter notebook Router.ipynb                           # 2. Multi-agent routing
jupyter notebook Supervisor.ipynb                       # 3. Orchestration
jupyter notebook HandsOff.ipynb                         # 4. Peer-to-peer handoffs
jupyter notebook Sequential-Condtional-Loops.ipynb      # 5. Pipeline with quality gate
```

## 🤝 Contributing

Contributions are welcome! Please:
1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/NewPattern`).
3. Commit your changes (`git commit -m 'Add new orchestration pattern'`).
4. Push to the branch (`git push origin feature/NewPattern`).
5. Open a Pull Request.
