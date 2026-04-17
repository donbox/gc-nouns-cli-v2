# Nouns CLI Unification

## Purpose

This note starts the CLI v2 design pass for the remaining major GC nouns after
the Pack/Registry work:

- `city`
- `rig`
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

## In-Scope Nouns

The nouns below appear to matter most for the next pass.

| Noun | Initial role guess |
|---|---|
| `city` | root operational/project container |
| `rig` | named scoped environment or subdivision within a city |
| `agent` | runtime worker or persona attached to a city/rig |
| `formula` | reusable declarative logic unit |
| `order` | invocation or execution request |
| `skill` | reusable capability bundle |
| `mcp` | external tool/provider integration |

This table is intentionally provisional. One of the main risks in the current
surface is that some of these nouns may really be:

- first-class nouns
- configuration attached to another noun
- execution subcommands rather than nouns

## Early Design Pressure

These are the main issues I see before we start inventing signatures.

### 1. Some nouns are containers, some are declarations, some are runtime actors

The set mixes at least three kinds of things:

- containers: `city`, `rig`
- runtime actors: `agent`
- declarations/config: `formula`, `order`, `skill`, `mcp`

If the CLI treats all of them as peer nouns with the same command grammar, we
risk a surface that is mechanically consistent but semantically awkward.

### 2. Ownership boundaries are not obvious yet

Some likely questions:

- does `rig` stand alone, or is it always addressed through a city?
- is `agent` a top-level noun, or a city/rig subresource?
- does `formula` live on its own, or under the city/rig that owns it?
- is `mcp` machine-level configuration, city-level configuration, or both?

Before choosing verbs, we probably need a simple ownership map.

### 3. Ambient targeting may work for some nouns but not others

The Pack pass settled on an ambient pack plus explicit `--rig`.

For these nouns, the likely story is less uniform:

- `city` may be ambient
- `rig` may be ambient for inspection but explicit for mutation
- `agent` probably should not silently target "whichever one is nearby"
- `mcp` may have no good ambient target at all

The CLI will feel much better if we explicitly decide where ambient behavior is
safe versus where exact addressing is required.

### 4. Some nouns may want singular object views, others collection-first views

Examples:

- `agent` likely needs strong single-object commands
- `formula` may want both collection and single-object workflows
- `skill` may be more discovery/import-oriented than mutation-oriented
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

### 6. Relationship visibility will matter early

These nouns appear highly connected:

- agents run in rigs or cities
- orders may invoke formulas
- skills may depend on MCP connections
- formulas may be referenced by orders or agents

We should expect relationship-view commands or output to matter fairly early,
even if we do not design them immediately.

### 7. Machine config vs project config may reappear

The Pack/Registry work found a productive split between:

- machine-known config
- project-scoped/local workflow

That same fault line may show up here, especially around:

- `skill`
- `mcp`
- perhaps `agent` templates or defaults

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
- it likely mixes config and operational verbs

### Hypothesis 3: `formula` and `order` may form a producer/consumer pair

There is a decent chance that:

- `formula` is the reusable declared thing
- `order` is the invocation/execution thing

If so, their command grammars probably should not be identical.

### Hypothesis 4: `skill` and `mcp` may want stronger config/discovery affordances

These nouns may end up closer to:

- import/install/search/list/show

than to:

- start/stop/run

That means we should be careful not to let runtime nouns dictate their
surface.

## First Questions To Answer

The most valuable next-step questions appear to be:

1. Which nouns are truly top-level nouns versus attachments/subresources?
2. Which nouns are ambiently targetable?
3. Which nouns are primarily declarative versus operational?
4. Where do machine-level settings need to exist, if anywhere?
5. Which relationships must be visible in POR output?

## Parked Questions

These are worth tracking, but they should not block the first command-surface
pass.

1. Whether some nouns should really collapse into subcommands of another noun.
2. Whether some machine-level configuration should live in file-backed admin
   surfaces analogous to the registry design.
3. Whether there is a consistent "show vs list vs search" grammar that can
   carry across all of these nouns without forcing bad semantics.
