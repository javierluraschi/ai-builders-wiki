# AI Builders Wiki

**A practical, sourced, vendor-neutral wiki for people who build with AI: what the pieces are, how agents work, who makes the models, what vendors do with your data, and how to get an AI app online safely.** It is written for developers, and for founders who build with AI without being full-time programmers.

**How it is written.** Anyone can propose a change by pull request, written with or without AI help; maintainers review it against the sources before merging. Every claim about a vendor links to that vendor's own page, with the date it was checked, and each page's footer says how it was written and when it was last checked. Facts in this field change monthly, model names especially. Nothing here is legal or security advice.

## Start here

New to the subject? Read in this order:

1. [Glossary](Glossary.md): the basic words
2. [What generative AI is](Generative-AI.md)
3. [What an AI agent is](AI-agent.md)
4. [From idea to deployed AI app](Idea-to-deployed-AI-app.md)

## All pages

Also in the [sidebar](_Sidebar.md). Browse by [tag](Tags.md).

<!-- INDEX:START -->

### [Foundations](Category-Foundations.md)

- [Glossary: the basic words](Glossary.md): The core terms a newcomer meets, each defined in plain words, with links to the pages that go deeper.
- [What generative AI is, and why people build with it](Generative-AI.md): What the models do, what developers and founders use them for, where they fail, and how the bill works.
- [Ways to use AI: chat, projects, coding agents, IDEs, APIs, SDKs](Ways-to-use-AI.md): The surfaces through which people and programs use models, and what each one is for.
- [The cloud, for AI apps](Cloud-for-AI-apps.md): Three different jobs: hosting your app, calling a model, running a model on GPUs. And serverless vs containers vs VMs.

### [Agents](Category-Agents.md)

- [What an AI agent is](AI-agent.md): How an agent differs from a chatbot and from a workflow: tools, the loop, and levels of autonomy.
- [Multi-agent systems: do AI agents work with each other?](Multi-agent-systems.md): Manager and helpers, handoffs, cross-organization agents, frameworks, and what the evidence says works.
- [Agent protocols: MCP and A2A](Agent-protocols-MCP-and-A2A.md): What the Model Context Protocol and the Agent2Agent protocol are, how they differ, and their status as of October 2026.

### [Models and providers](Category-Models-and-providers.md)

- [AI providers and models: who makes what, and how to choose](AI-providers-and-models.md): The main model makers as of October 2026, open-weight vs closed, where to get them, and how to pick one.
- [Model names and versions](Model-names-and-versions.md): How model families, tiers, snapshots, aliases and retirement dates work, vendor by vendor.

### [Security and data](Category-Security-and-data.md)

- [Security for AI apps and agents](Security-for-AI-apps.md): The risks that are specific to apps that call models and to agents that act: prompt injection, secrets, scopes, tool permissions, supply chain, runaway costs.
- [Data protection when you use AI](Data-protection.md): The five questions to ask any AI vendor, consumer vs business terms, and what to do in your own app.
- [Data protection by vendor](Data-protection-by-vendor.md): Training defaults, opt-outs, retention, zero data retention and data location for each major AI vendor, every cell sourced.

### [Guides](Category-Guides.md)

- [From idea to deployed AI app: a typical workflow](Idea-to-deployed-AI-app.md): One concrete path, end to end: build with an assistant, keys and secrets, deploy, logs, costs, updates.
- [Why a deploy looks broken in one browser but fine in another](Stale-cache-after-a-deploy.md): A new HTML page paired with an old cached CSS or JS file: how to spot it, and how to give assets a URL that changes when they do.
- [Where your keys come from: attribution for AI apps](Where-your-keys-come-from.md): When agents sign up instead of people, record a link ref, the agent and model the caller reports, and its HTTP client family, then count them by day.
- [Querying a web crawl from a small app](Querying-a-web-crawl-from-a-small-app.md): Read Common Crawl's Parquet index and hyperlink graph directly from a small container with DuckDB, without copying the data — and without the big join that runs a 256 MB container out of memory.

### Tags

[agents](Tag-agents.md) · [analytics](Tag-analytics.md) · [basics](Tag-basics.md) · [coding-agents](Tag-coding-agents.md) · [costs](Tag-costs.md) · [data](Tag-data.md) · [hosting](Tag-hosting.md) · [mcp](Tag-mcp.md) · [models](Tag-models.md) · [privacy](Tag-privacy.md) · [providers](Tag-providers.md) · [secrets](Tag-secrets.md) · [security](Tag-security.md)

<!-- INDEX:END -->

## How this wiki works

- **Pages** are Markdown files at the top level of the repository, named in wiki style (`AI-agent.md`). Each starts with a breadcrumb, answers its question in the first sentence, and ends with *Related pages*, *See also*, *Sources* and a footer.
- **Categories and tags** live in each page's front matter. They were chosen from the pages as they were written, and will change.
- **Navigation** (this index, the sidebar, the category and tag pages, the breadcrumbs and the glossary links under *See also*) is generated by `python3 scripts/build_wiki.py`, which also checks that every internal link resolves.

## Contribute

This wiki is open to everyone. Fix a sentence, update a fact that changed, add a page, or translate one: open a pull request, or an issue if you'd rather point at a problem. How, in [CONTRIBUTING.md](CONTRIBUTING.md).

## About this wiki

Started by the team behind [Cloudish](https://cloudish.ai/?ref=cc-wiki), at [Hal9](https://hal9.com). Cloudish appears on a few pages as one example among other hosts, and says so where it does. If a page favours anyone unfairly, please [open an issue](CONTRIBUTING.md#report-a-mistake).

## Licence

Text is licensed under [CC BY 4.0](LICENSE). The build script is under the MIT licence.
