# Product Prompt Archive

## Original Prompt

```text
This is my idea: The "Tactical Edge" Soccer Analyst. This is a live companion app for Premier League matches that provides predictive insights and tactical breakdowns during gameplay.

LLM: Llama 3.3 to ingest live match statistics (possession, xG, heatmaps) and provide natural language tactical analysis.

Workflow / Coordination: Use Workers to poll live sports APIs and trigger Workflows whenever a significant event (goal, red card, substitution) occurs to recalculate match predictions.

User Input: A Voice-to-Text interface (using Cloudflare’s AI speech-to-text models) where a user can ask, "How will the substitution of a midfielder affect the defensive line?"

Memory / State: Use Durable Objects to track the "Match Momentum." The state would store the last 10 minutes of match events to give the LLM context for its next prediction.

The main specs are in the given file SPECS.MD. Do not look on README as it contains nothing right now. Also, create a file called PROMPTS.md and include this prompt onto the file.
```

## Implementation Planning Prompt

```text
Now I want you to begin creating an implementations plan for every single part of this product. I want you to use the SPECS.MD file to derive the plan.

In the plan, for each phase, include what technologies should be used and also what will be done. Include this prompt as another prompt in the PROMPTS.MD file.
```
