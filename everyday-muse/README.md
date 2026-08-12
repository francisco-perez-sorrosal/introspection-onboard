# everyday-muse

Notices a quiet truth about ordinary life, then finds a real, sourced quote that echoes it.

Run it bare and it invents one specific observation about everyday human life, searches the live web for a published quotation that resonates with it, and pairs the two. Pass a theme and the observation is about that instead.

```
When you are waiting for something small — a kettle, a traffic light, a page
to load — your body quietly assumes a posture you never decided on, the same
one every time, a private statue of yourself that no one has ever described
to you.

    "Our convictions become platitudes ground out on a barrel-organ, our
     ideals become starchy habits, enthusiasm stiffens into automatic
     gestures. The source of the water of life seeps away."
    — Carl Jung, Symbols of Transformation

Source: https://carljungdepthpsychologysite.blog/2020/10/14/symbols-5
```

## The one guarantee

**Every quote is real, attributed, and linked — or the agent says it found none.**

It may only quote text that actually appeared in search results. It never writes a quotation from memory, never repairs a half-remembered line, and never attaches an author the search did not attribute. A run that ends *"I could not find a sourced quote for this one"* is a correct run; an invented attribution is a failed one.

It also distinguishes *the search broke* from *the web had nothing* — those are different answers and it reports them differently.

## Run it

```bash
introspection local --runtime everyday-muse                  # interactive
introspection local --runtime everyday-muse -p "Go."         # one-shot
introspection local --runtime everyday-muse -p "resilience"  # themed
```

`--runtime` selects this recipe's manifest; this repository holds more than one.

## Requirements

`PARALLEL_API_KEY` must be set in the environment. The agent's only external capability is the [Parallel Search API](https://docs.parallel.ai), reached through the `parallel_search` tool in `extensions/parallel-search.ts`. The key is read from the environment by reference and never written into the package.

Without it the agent still runs and still produces an observation — the search reports a configuration error, and the agent reports honestly that it could not source a quote.

## Layout

```
everyday-muse/
├── SYSTEM.md                      the mission, the hard boundary, the output shape
├── agents/agent.yaml              model + the single-tool allowlist
└── extensions/parallel-search.ts  the Parallel Search tool
```

The agent holds exactly one tool. It has no filesystem, shell, or edit access — it reads the web and writes a paragraph.

Behavior lives in three places, and which one you change decides what you have to restart: `SYSTEM.md` and `agents/agent.yaml` need a fresh session, and the manifest at `../.introspection/everyday-muse.yaml` needs a new runtime version.

## Deploying

`introspection check` validates the package. Deployment is the `deploy` workflow — the recipe is registered at its path in this repository, so the monorepo's other agents deploy independently.

---

Built from the Apache-2.0 licensed [`template-starter`](https://github.com/introspection-recipes) recipe; see `LICENSE`.
