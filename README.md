# Errand — Physical-world execution for AI agents

Give your AI agent eyes and hands in Seoul. Dispatch a nearby real human for on-site checks, store or shelf verification, photos, queue checks, or pickups — and get GPS/time-verified evidence back.

Use Errand when a task cannot be completed entirely online and an agent needs physical presence at a real place. A nearby worker — an actual human with the Errand iOS app — travels to the location, shoots the photos in-app, and answers the questions you asked. The server checks where and when the photo was taken, an AI grades it against your criteria, and you get photos, structured answers, and a pass/fail verdict.

Ask Claude things like:

- "Is the pop-up at Gangnam Exit 11 open right now, and how long is the line?"
- "Photograph the menu board at this café and tell me the lunch prices."
- "Is this product actually on the shelf at that store, and at what price?"
- "Send someone to verify this storefront and return a GPS-verified photo."
- "Can a local human pick this item up for me near Gangnam?"

**Coverage is Seoul — Gangnam district and nearby — and nowhere else yet.** The coverage tool answers honestly, including zero, so Claude can tell you "nobody is reachable there" instead of guessing.

## Install

```
/plugin marketplace add ChloePark85/errand-plugin
/plugin install errand@errand
```

You will be asked for an Errand API key during installation.

## Getting a key

1. Go to <https://errand.be>, enter an email, and type the 6-digit code that arrives.
2. The key (`er_live_…`) is shown **once** — save it.
3. Add a prepaid balance at <https://errand.be/dashboard>. A first errand typically costs ₩4,000–₩12,000 in total.

Coverage checks work without a key, so Claude can answer "is this even possible?" before you sign up.

## What it costs

Korean won, prepaid. The worker receives the reward in full; your account is charged the reward plus a platform fee of 20% (minimum ₩500).

| Reward to the worker | You are charged |
|---|---|
| ₩5,000 | ₩6,000 |
| ₩10,000 | ₩12,000 |

The whole amount is held in escrow the moment the errand is created and **refunded in full** if it fails, expires, or is cancelled before someone claims it. The fee is only earned on a completed errand.

Every API key carries caps its owner sets — reward per task, daily budget, allowed task types, allowed area — so Claude cannot spend beyond what you pre-approved.

## What is in this plugin

- **MCP connector** to `https://errand.be/api/mcp` (remote, streamable HTTP), exposing seven tools: `errand_check_coverage`, `errand_list_capabilities`, `errand_quote`, `errand_dispatch`, `errand_get_status`, `errand_get_result`, `errand_cancel`.
- **A skill** that teaches Claude the flow — check coverage, ask the server for an exact quote, get your explicit yes before spending anything, then poll and report the result in plain language.

Only `errand_dispatch` moves money. `errand_quote` returns the exact total for free, and the skill instructs Claude to put that number in front of you and wait for a yes before dispatching.

## Privacy and safety

Photos are taken with the in-app camera only; gallery uploads are blocked. Workers see the mission text you write and the location — never your identity. You receive the evidence, not the worker's personal details.

- API reference: <https://errand.be/docs>
- Privacy policy: <https://errand.be/privacy>
- Terms: <https://errand.be/terms>
- Support: <support@errand.be>

Also listed in the [official MCP registry](https://registry.modelcontextprotocol.io) as `be.errand/errand`.

## License

MIT. See [LICENSE](LICENSE).
