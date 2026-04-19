# Nouns CLI Unification

## Purpose

This note starts the CLI v2 design pass for the remaining major GC nouns after
the Pack/Registry work:

- `city`
- `rig`
- `session`
- `agent`
- `formula`
- `order`
- `skill`
- `mcp`

The goal is not to lock a command tree immediately. The first task is to make
sure the noun boundaries, targeting rules, and object lifecycles are coherent
enough that a command surface can stay stable once we expose it.

## Starting Point

The Pack/Registry pass converged on a few principles that are likely useful
here too:

- separate local workflow nouns from lookup/admin nouns when expectations
  differ materially
- prefer exact, teachable signatures over overly clever polymorphism
- let ambient behavior do useful work, but only when the ambient target is
  obvious
- keep review focused on command names, signatures, and output shape before
  implementation details

## Corrected Starting Frame

After the first pass, a few corrections seem important enough to treat as the
new baseline:

- `session` is the primary runtime noun
- `agent` is better understood as a session template/stamping noun than as the
  runtime thing itself
- `session` and `agent` still need to be treated as peers in the CLI design;
  any symmetry or asymmetry between them should be deliberate rather than
  accidental
- `skill`, `mcp`, and overlays are not obvious peer nouns; they look more like
  agent/session components
- the machine-known nouns appear much smaller than the first pass assumed

This note treats those as the new working assumptions.

## Machine-Wide Concepts

At the moment, the machine-wide concepts appear to be:

| Noun | Why it is machine-known |
|---|---|
| `city` | unit of deployment and management |
| `rig` | needed so any directory can be mapped back to an in-scope city/rig |
| `pack` | registry and cache behavior make it machine-known |

Things that do **not** appear machine-wide in their own right:

- `skill`
- `agent`
- `mcp`
- overlays

Their footprint seems to come transitively from the pack or session they are
attached to, not from a separate machine-level registry of their own.

## In-Scope Nouns

The nouns below appear to matter most for the next pass.

| Noun | Initial role guess |
|---|---|
| `city` | root operational/project container |
| `rig` | named scoped environment or subdivision within a city |
| `session` | live runtime instance |
| `agent` | session template / stamping definition |
| `formula` | reusable declarative logic unit |
| `order` | invocation or execution request |
| `skill` | portable capability bundle attached through agent/session composition |
| `mcp` | external tool/provider integration attached through agent/session composition |
| `overlay` | static stamped material attached through pack/agent/session composition |

This table is intentionally provisional. One of the main risks in the current
surface is that some of these nouns may really be:

- first-class nouns
- configuration attached to another noun
- execution subcommands rather than nouns

There is also a fourth category worth watching for:

- high-frequency action verbs that may deserve top-level ergonomic treatment
  even if they are not nouns in the same sense as the objects above

## Early Design Pressure

These are the main issues I see before we start inventing signatures.

### 1. Some nouns are containers, some are runtime actors, some are template components

The set mixes at least three kinds of things:

- containers: `city`, `rig`
- runtime actors: `session`
- template/config: `agent`, `formula`, `order`
- template components: `skill`, `mcp`, overlays

If the CLI treats all of them as peer nouns with the same command grammar, we
risk a surface that is mechanically consistent but semantically awkward.

### 2. Ownership boundaries are not obvious yet

Some likely questions:

- does `rig` stand alone, or is it always addressed through a city?
- is `session` a top-level noun with city/rig targeting, or a subresource?
- is `agent` a top-level noun, or a city/rig subresource?
- if `session` and `agent` are peers, where do shared verbs intentionally
  align and where should they diverge?
- does `formula` live on its own, or under the city/rig that owns it?
- is `mcp` machine-level configuration, city-level configuration, or both?

Before choosing verbs, we probably need a simple ownership map.

### 3. Ambient targeting may work for some nouns but not others

The Pack pass settled on an ambient pack plus explicit `--rig`.

For these nouns, the likely story is less uniform:

- `city` may be ambient
- `rig` may be ambient for inspection but explicit for mutation
- `session` probably should not silently target "whichever one is nearby"
- `agent` probably wants stronger targeting than a plain ambient lookup
- `mcp` may have no good ambient target at all

The CLI will feel much better if we explicitly decide where ambient behavior is
safe versus where exact addressing is required.

### 4. Some nouns may want singular object views, others collection-first views

Examples:

- `session` likely needs strong single-object commands
- `agent` likely needs strong single-object commands too, but of a different kind
- `formula` may want both collection and single-object workflows
- `skill` may be more inspection/promotion-oriented than mutation-oriented
- `mcp` may be mostly list/show/configure/test

This matters because we should not assume every noun wants:

- `add`
- `remove`
- `list`
- `show`

as its natural first-release surface.

### 5. Declarative vs operational verbs will probably diverge

Some nouns likely want configuration verbs:

- add
- remove
- list
- show

Others likely want operational verbs:

- run
- test
- start
- stop
- attach
- dispatch

If a noun mixes both, we should decide whether:

- the noun owns both workflows
- or one workflow belongs under another noun

We should also leave room for a small set of highly ergonomic "do this now"
operations to remain especially prominent, even if they do not map cleanly to
one noun family. The product simplification pass surfaced `mail`, `sling`, and
possibly `nudge` as candidates for this exception.

### 6. Relationship visibility will matter early

These nouns appear highly connected:

- sessions run in rigs or cities
- agents stamp sessions
- orders may invoke formulas
- skills may depend on MCP connections
- formulas may be referenced by orders, agents, or sessions

We should expect relationship-view commands or output to matter fairly early,
even if we do not design them immediately.

### 7. Machine config vs project config may reappear

The Pack/Registry work found a productive split between:

- machine-known config
- project-scoped/local workflow

That same fault line may show up here, especially around:

- `pack`
- perhaps `city` discovery data

We should stay alert for cases where a separate admin/config noun is cleaner
than forcing everything into one top-level surface.

## Initial Working Hypotheses

These are not decisions yet. They are the hypotheses I would want to test
first.

### Hypothesis 1: `city` and `rig` are target selectors before they are command families

It may turn out that:

- `city` is both a noun and an ambient root target
- `rig` is more often a refinement of another command than a full peer surface

If true, we should resist over-growing `rig` into a giant parallel command
tree too early.

### Hypothesis 2: `agent` is a first-class noun

`agent` feels likely to justify its own surface because:

- it is user-visible
- it has strong single-object identity
- it stamps session working directories and runtime defaults
- but it may be more template/provisioning than runtime lifecycle

The key design constraint is that `agent` should not simply become "the less
important cousin of session." If `session` and `agent` are peers, the command
surface should show that on purpose.

### Hypothesis 3: `session` is the most important runtime noun

`session` now looks like the live phenomenon that users actually inspect,
interact with, and manage. That likely means:

- session lifecycle verbs should be designed early
- session identity/addressability matters more than agent identity for runtime UX
- any CLI that over-centers `agent` risks confusing template and runtime layers

### Hypothesis 4: `formula` and `order` may form a producer/consumer pair

There is a decent chance that:

- `formula` is the reusable declared thing
- `order` is the invocation/execution thing

If so, their command grammars probably should not be identical.

### Hypothesis 5: `skill`, `mcp`, and overlays are better treated as agent/session components first

The current structure strongly suggests:

- overlays are init-time projection/stamping material
- skills are portable capabilities that can evolve over session lifetime
- MCP entries are agent/session-attached tool integrations

That means these may not want peer top-level noun status in POR, even if they
eventually deserve dedicated helper surfaces.

The open complication is that all three may also have pack-wide forms:

- pack-wide skills
- pack-wide MCP definitions or templates
- pack-wide overlays

So the real design task is probably not "bury these under agent/session and
move on." It is to find the shared ownership pattern across:

- pack-owned definitions
- agent-attached baseline composition
- session-effective/runtime realization

### Hypothesis 6: skill lifecycle is a key seam

Skills look different from overlays in one important way:

- overlays appear init-time only
- skills can change over a session's lifetime
- some earlier design work explicitly contemplated promoting session-created
  skills back into authored pack state

That suggests a likely lifecycle:

- baseline skills come from pack/city/agent definition
- sessions accumulate additional skills over time
- selected session-added skills may be explicitly promoted back into durable
  authored state

If this holds, `skill` is not just static pack content; it is a bridge between
runtime accumulation and durable authored configuration.

That also suggests we should be careful not to "simplify away" the current
skill idea. It is powerful enough that even if the current `gc skill` surface
is refactored heavily, the concept itself should survive.

### Hypothesis 7: session/agent should be designed as a pair

The product simplification pass reinforced this strongly:

- `session` is the live runtime object
- `agent` is the authored/template object
- users will compare them constantly

So the next pass should probably treat them as a paired design problem rather
than finishing one and only later revisiting the other.

## Skill Design / Release Reality Check

The earlier skill design work lives primarily in:

- `/Users/dbox/repos/gc/gascity/docs/packv2/doc-agent-v2.md`

Important companion docs:

- `/Users/dbox/repos/gc/gascity/docs/packv2/doc-loader-v2.md`
- `/Users/dbox/repos/gc/gascity/docs/packv2/doc-conformance-matrix.md`
- `/Users/dbox/repos/gc/gascity/docs/packv2/skew-analysis.md`

### What the design note says

The PackV2 agent design note says:

- skills use the Agent Skills standard
- skills can be city-wide or per-agent
- the first slice is current-city-pack only
- imported-pack skill catalogs are later
- the first skills CLI slice is list-only:
  - `gc skill list`
  - `gc skill list --agent <name>`
  - `gc skill list --session <id>`
- skill promotion is explicitly design-noted as a later flow
- promotion is an explicit human decision rather than automatic backflow

The note also frames overlays, skills, and MCP as assets that live under the
agent/city pack structure rather than as machine-global objects.

That is another sign the next pass should inspect all three together rather
than treating only skills as special.

### What the documented release reality says

The concrete release artifacts I could verify locally are the public tags
`v0.14.0` and `v0.14.1`.

Those release snapshots show something much narrower:

- `docs/packv2/doc-agent-v2.md` is not present in either release tag
- the released CLI reference still documents `gc skill` as the old curated
  topic viewer, not as a real skill noun
- the conformance/skew docs on `main` still mark pack `skills/` discovery and
  MCP TOML abstraction as documented-but-not-implemented

So the practical lesson is:

- the skill/session/agent design note is ahead of what the released product
  actually exposes
- the release reality is still pre-skill-product in CLI terms
- we should be careful to distinguish:
  - designed direction
  - current `main` docs
  - actually released behavior

That gap is useful. It means the next CLI design pass should not assume there
is already a stable, released skill noun we are constrained by.

## First Questions To Answer

The most valuable next-step questions appear to be:

1. Is `session` the primary runtime noun, with `agent` explicitly treated as
   template/provisioning?
2. If `session` and `agent` are peers, what symmetry or asymmetry between them
   should be explicit in the command surface?
3. Which nouns are truly top-level nouns versus attachments/subresources?
4. Which nouns are ambiently targetable?
5. Which nouns are primarily declarative versus operational?
6. Which relationships must be visible in POR output?
7. What is the explicit lifecycle for session-added skills and promotion back
   into pack-owned state?
8. What is the shared ownership pattern across pack-wide, agent-attached, and
   session-effective forms of skills, MCP, and overlays?

## Implications From The Product Simplification Pass

The parallel product simplification work suggests a few constraints that this
note should carry forward:

1. Some operations may deserve top-level ergonomic treatment even if they do
   not map neatly to one noun. We should keep that in mind before burying every
   action under a noun tree.
2. `mail`, `sling`, and possibly `nudge` feel like "finally use the city"
   operations. That makes their eventual placement partly an ergonomics
   question, not just a taxonomy question.
3. This noun work is likely the gating step before the broader product-zoning
   work can advance much further. That means it is worth being deliberate here,
   especially on the session/agent/skill/mcp/overlay cluster.

## Parked Questions

These are worth tracking, but they should not block the first command-surface
pass.

1. Whether some nouns should really collapse into subcommands of another noun.
2. Whether some machine-level configuration should live in file-backed admin
   surfaces analogous to the registry design.
3. Whether there is a consistent "show vs list vs search" grammar that can
   carry across all of these nouns without forcing bad semantics.
