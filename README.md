# Rye Agent Skills

Agent skills for [Rye's Universal Checkout API](https://docs.rye.com) — turn any product URL into a completed purchase.

Built on the [agentskills.io](https://agentskills.io) open standard. Works with Claude Code, Cursor, Codex, and any compatible agent.

## Installation

```bash
npx skills add rye-com/agent-skills
```

## Available Skills

<details>
<summary><strong>rye-overview</strong></summary>

Learn what Rye is, explore capabilities, and find the right path for your use case.

**Use when:**

- You want to understand what Rye does and how it works
- You're exploring whether Rye fits your use case
- You need help choosing between integration approaches
- You want to see what you can build (shopping assistants, dropshipping, chat commerce, etc.)

</details>

<details>
<summary><strong>rye-universal-checkout</strong></summary>

Integrate Rye checkout into apps or buy products programmatically via AI agents. Contains references for developer integration, agent commerce, and the full API.

**Use when:**

- Integrating Rye checkout into an existing codebase
- Building an AI agent that purchases products on behalf of users
- Looking up product data (price, availability, images) from any URL
- Writing a script that buys a product from any merchant

**References included:**

- Developer Integration Workflow (analyze → plan → implement → verify)
- Agent Commerce Workflow (lookup → buy → confirm)
- Full API Reference (endpoints, SDKs, examples)

</details>

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**

```
What is Rye? Can it work for my use case?
```

```
Integrate Rye checkout into my Next.js app
```

```
Look up this product and tell me if it's available: https://www.amazon.com/Apple-MX532LL-A-AirTag/dp/B0CWXNS552/
```

```
Build me a Python script that buys a product from any URL
```

```
Buy this for me: https://flybyjing.com/collections/shop/products/the-big-boi
```

```
Help me go from Rye staging to production
```

## License

MIT
