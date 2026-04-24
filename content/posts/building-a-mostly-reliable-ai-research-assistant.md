+++
title = "Building a (mostly) reliable research assistant"
date = 2026-04-24
+++

If you install software from a public package registry (npm, PyPI, Crates, etc.), every security decision that registry makes sits inside your trust boundary. You don't run the infrastructure, you don't set the publisher policies, you don't decide what "verified" means, and whatever the registry (fails to) enforce travels straight into your build.

This is a well-studied surface. Sonatype tracked more than 700 thousand malicious open-source packages between 2019 and the end of 2024, with over half a million new ones in the last year of that window alone! That's a 156 percent year-over-year growth rate.[^sonatype] There are some existing concepts that can help: SLSA gives us a vocabulary for provenance[^slsa], for example, and Sigstore gives us signing and transparency logs.[^sigstore] Both are real, serious, and widely adopted. But provenance and signing don't themselves answer the question I keep bumping into, which is *what is the actual security posture of package registries?*

So I decided to try to answer that question. I started where all good plans start: a list. I call it the Software Supply Chain Security Scorecard (SSCSS). It's a set of 52 binary questions across eight sections, each probing one concrete security control a registry either enforces or doesn't: *Does the source enforce multi-factor authentication for publishers? Does it require identity federation for organisational accounts? Does it display a verified publisher identity to consumers? Does it have a published vulnerability disclosure policy?* Yes, no, or unknown. The questions are binary, but the work of answering them is anything but.

A quick note: This is not to be confused with the OpenSSF *Security Scorecard* project[^openssfurl]. This is an unrelated tool that scores the security hygiene of open-source *source-code repositories* (branch protection, signed releases, SAST configuration, and so on) rather than registries. Similar name but a different scope. (I didn't actually know it existed when I named mine.)

A good human assessment (i.e. me) takes upwards of a day and a half. It takes that long because *enforced* is not the same as *supported*; because documentation and behaviour don't always agree; because the useful evidence is usually scattered across docs, RFC repositories, governance pages, changelogs, and the occasional blog post from 2022; and because the honest answer to a surprising number of questions is still "yeah, I dunno" after hours of careful reading. Multiply that by the long tail of registries in general, and it rapidly becomes clear that if you want to cover this ecosystem with any breadth you need either a bigger team or a different approach.

Well I'm just one person, so I tried the different approach. I built an AI research agent that could work through the scorecard on its own. So this post is about what that took, which turned out to be much more than "wire up an LLM and let it go." The title's hedge is load-bearing: the agent I ended up with is *mostly* reliable, and the *mostly* is where the interesting lessons live (I think).

p.s. When I say that I built an agent, that makes it seems like it was a success from day one. It *really* was not. I'll walk through four attempts, and spoiler alert: the first three were failures.

## Attempt one: straight at the wall

The first version was made before I had thought carefully about anything. I fired up Claude Code and gave it a single prompt that said, roughly, *here is a list of 52 questions and the name of a target registry*, then I let it rip. You can imagine how that went. Context limits, rate limits, truncated output, the whole thing keeling over somewhere around question fifteen. I sat with the broken output for a while and felt stupid.

In retrospect, this was the best possible outcome. It failed immediately, unambiguously, and cheaply. No half-working demo to mistake for progress, no partial results to rationalise. It was the kind of failure that makes you stop typing and start thinking. The thing I was thinking about, initially, was the shape of the input: maybe the prompt was too long, maybe the questions were too varied, maybe I needed to batch. What I should have been thinking about (and wasn't yet) was the *shape of the problem*. I'd get there eventually, but I had to go the long way around first.

## Attempt two: the orchestration era

The second version was much more considered and approximately just as useless. Its architecture was the one you'd draw on a whiteboard if someone gave you the task for the first time: a lightweight orchestrator on Haiku, spawning a Sonnet researcher agent per scorecard section, each writing its results back to a shared JSON file. Eight parallel researchers, a coordinator, and a shared state file. Very reasonable and, as it turned out, very error-prone. The problems came in two waves.

The first wave was mechanical. Researcher agents would occasionally hang indefinitely, producing nothing and blocking the run. Others would hit a transient error mid-research, write a blank result, and report success upstream to the orchestrator, which would dutifully record the blank result as an answered question. The shared-state file became a source of subtle corruption as different researchers arrived at slightly different interpretations of its structure under slightly different success and failure conditions. The orchestrator grew more defensive with every iteration. Its prompt climbed from a few hundred lines of markdown to nearly a thousand as I added more rules for validating results, more recovery paths for hanging agents, more invariants to protect the state file. More instructions did not produce more reliable behaviour—they produced more surface area for things to go sideways on. The system became harder to reason about with every attempted fix.

I should have paused and asked why a thousand-line prompt was making things *worse*. I did not. I kept going. Eventually the mechanical problems settled down to a grim steady state where the system would complete a run without actually crashing, and I was able to sit down and read the results.

Which is when the second wave hit: The *actual research wasn't any good.*

I had confident "yes" answers backed by evidence that clearly said "no" if you read it. I had notes that bore no relation to the question they were attached to. I had URLs that did not exist. Bafflingly, I had answers where the "evidence" turned out, on inspection, to be the model narrating its own search process. Amazing. And when I ran the same assessment against the same registry three times, the results disagreed with each other—not only in their answers but in which questions they considered answerable at all. The inconsistencies were, themselves, inconsistent.

There is a name for this category of failure in the academic literature, and it is *not* hallucination, at least not in the useful sense of that word. Farquhar, Kossen, Kuhn, and Gal published a paper in *Nature* in 2024 that carves out a specific subtype they call *confabulation*: "arbitrary and incorrect generations" that vary between reruns of the same query.[^farquhar] Hallucination is the umbrella; confabulation is the subset where the wrongness isn't stable. The distinction matters, because confabulations are exactly the errors you'll catch if you run a query multiple times and compare (and exactly the ones you'll miss if you don't).

Why does this happen at all? An OpenAI paper from September 2025 makes a mechanistic argument I find persuasive: modern LLM training and evaluation procedures reward *guessing* over *acknowledging uncertainty*. "Language models are optimized to be good test-takers," write Kalai, Nachum, Vempala, and Zhang, "and guessing when uncertain improves test performance."[^kalai] The structural incentive of the training pipeline is to produce a confident answer, not to produce *no* answer when no answer is warranted. Confident wrongness isn't a bug; it's the predictable output of the optimisation you've run.

You can zoom out further. In the bluntly-titled 2024 paper *ChatGPT is bullshit*, Hicks, Humphries, and Slater argue that "hallucination" is actually the wrong metaphor for what LLMs do when they produce false text. Hallucination implies a perceptual system misperceiving reality. But LLMs don't perceive reality—they produce text without any epistemic stake in it at all. Their behaviour is better described, the authors argue, in the sense that Harry Frankfurt gave the word *bullshit* in 1986: "speech or text produced without concern for its truth." The bullshitter "does not care whether the things he says describe reality correctly. He just picks them out, or makes them up, to suit his purpose."[^hicks][^frankfurt]

This was the frame I needed. The agent wasn't *misperceiving* the documentation. It wasn't *trying to mislead*, either. It was a confident pattern-completer optimised to produce a plausible answer, and plausible answers are easier to generate than correct ones. Once I really internalised that, I stopped trying to make the system more *sophisticated* and started trying to make it more *accountable*.

I didn't get there yet, though. First I needed to take a detour through the land of milk and honey, also known as production-grade infrastructure.

## Attempt three: production-grade reliability theatre

I was struggling enough with attempt two that I decided to change tools entirely. I switched to Codex and spent a couple of weeks building what I thought was going to be the serious version.

What I produced was several thousand lines of best-practice Python: a clean orchestration layer, JSON schema enforcement on every boundary, a test suite with fixtures for each question type, local MCP servers, and a sandboxed research environment with controlled network egress. It was the kind of codebase you would be quietly pleased to ship. It had docstrings. It had a linter config. The CI ran on every commit.

It made no difference.

Every time I encountered a new failure mode, more code got written to handle it. The infrastructure became genuinely robust! I could catch a hung researcher, retry a malformed search, validate a schema, roll back a corrupted state file. That was all lovely and made my nerd brain hum, but none of that touched the thing that was actually broken. The research quality didn't move. The model still confabulated. The confident-yes-backed-by-evidence-that-says-no problem was exactly as prevalent as it had been in attempt two, now wrapped in a prettier wrapper.

This is the central engineering lesson I took from the project, and I'd like to state it as plainly as I can: **You cannot code your way out of an LLM quality problem.**

Mechanical failures and quality failures look similar from the outside. Both produce wrong answers. Both break user-visible behaviour. Both make you feel like you need to *fix* something. But they have entirely different causes.

A *mechanical failure* is a hung process, a schema violation, a corrupted file, a network timeout, etc. These are the kinds of things our field has spent fifty years learning to handle, and for which the standard defensive-engineering toolkit works well. A *quality failure* is the model confidently producing content that is wrong. No amount of retry logic, schema validation, sandboxing, or test coverage touches that, because none of those tools are in the loop where the wrongness actually gets introduced. They're upstream and downstream of it, not inside it.

This is not a new observation. Chip Huyen made it in its more general ML form back in 2022: "ML systems often fail silently … Users encounter painfully wrong outputs without realizing they're wrong, making performance expectation violations harder to detect than operational failures."[^huyen] Hamel Husain and Shreya Shankar (who between them have written much of the current practitioner canon on evaluating LLM applications) make a sharper version of the point: the most important activity in evals is *error analysis*, not infrastructure. "Start with error analysis, not infrastructure. Spend 30 minutes manually reviewing 20-50 LLM outputs whenever you make significant changes."[^husainshankar] Eugene Yan says it a third way: "While building with AI can feel like magic, building AI products still takes elbow grease. If teams don't apply the scientific method, practice eval-driven development, and monitor the system's output, buying or building yet another evaluation tool won't save the product."[^yan]

This was the pivot. The problem was never the infrastructure. It was the method.

### Factory vs lever

When I came back to Claude Code from Codex, I noticed something I hadn't seen clearly before. Both are terminal REPLs over an LLM. Both do broadly similar things. But they have fundamentally different defaults.

**Codex reaches for infrastructure.** Given a problem, it wants to build a factory: structured code, generator modules, layered abstractions, a test harness. It is very good at that, and if the problem you have is "produce a well-engineered software artefact," the factory instinct is the right one.

**Claude Code reaches for the simplest mechanism that works.** Given the same problem, it gravitates toward a single-file script, a direct invocation, the least amount of apparatus that can plausibly do the job. It will *build* infrastructure if you ask it to, but it won't go looking for excuses to.

This isn't a claim about the underlying models — Codex and Claude Code both run various members of the same family of frontier LLMs. It's a claim about the *harness*. The HumanLayer team have an apt framing: "The most well-known examples are Anthropic's Claude, where the same Claude model powers different interfaces with radically different behavior due to the harness."[^harness] What you think of as a model's personality is very often a harness's personality.

For a problem where the hard part was infrastructure, Codex's defaults would have been the right ones. For a problem where the hard part was research *quality*, I needed a lever, not a factory. I had been fighting a tool's instinct rather than matching it. Switching tools didn't solve my problem — it just moved me from a harness whose defaults were working against me to one whose defaults were working with me.

## Attempt four: starting from constraints, not architecture

For the fourth attempt I did something I should have done at the start and hadn't: I stopped designing and wrote down the constraints.

Three pages of them. Not requirements in the usual sense — no "the system shall" language. Just a list of problems I had hit and things I needed any real solution to satisfy. *Answers must be grounded in verifiable quotes, not in inference. The same question run twice should produce the same answer, or at least tell me honestly that it doesn't know. The agent should be able to resume if my laptop goes to sleep. I should be able to read the agent's reasoning trail after the fact, in full.* And so on.

One constraint, once I wrote it down, changed everything: **Time doesn't matter.**

I was assessing six registries, with more on the roadmap. I was not in a hurry. A single agent that finished a run in forty minutes of patient, serial work was strictly better than a fleet of eight parallel agents that finished in four but disagreed with themselves half of the time.

This eliminated parallelism. Which eliminated the orchestrator. Which eliminated the shared-state file. Which eliminated most of the surface area where attempt two had gone wrong. I realised that *a single agent, working slowly and correctly, through all 52 questions in series.* was not a compromise; it was the right answer to the actual problem, which I had been talking myself out of by building from the start.

The design principle that crystallised out of this (and which is a Datadog maxim): **simple, but not simplistic.** Not a short prompt that glosses over the hard parts. Not a naïve architecture that ignores the failure modes. Just the minimum complexity the problem actually demands, applied precisely where it's needed.

Anthropic's engineering team published a post called *Building effective agents* in December 2024 that captures the same sentiment from the other direction. Schluntz and Zhang write: "The most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns... Success in the LLM space isn't about building the most sophisticated system. It's about building the right system for your needs."[^anthropicagents] Simon Willison, after two years of refusing to use the word "agent" because nobody could agree on a definition, recently arrived at one that is about as tight as I've seen: "An LLM agent runs tools in a loop to achieve a goal."[^willisondef] That's the whole abstraction. Everything else is implementation detail.

The system I ended up with has two skill/agent pairs. In Claude Code terminology, a *skill* is a named entry point (i.e. something you invoke with `/skill-name`) that loads context and sets a workflow in motion. An *agent* is a specialised subprocess with its own system prompt, model assignment, and tool permissions. Skills are the user-facing verb and agents are the thing doing the work.

```
/build-starter-kit <registry>  →  starter-kit-builder (Sonnet)
                                        ↓
                                  internal/starter-kits/<registry>.json
                                        ↓
/assess <registry>             →  assessment (Opus)
                                        ↓
                                  assessments/<registry>/runtime-<ts>.json
                                  assessments/<registry>/scorecard-<ts>.json
```

### Start at the start

**`/build-starter-kit`** runs first, once per registry. Its entire job is source configuration: *where should the assessment agent look?* It maps out the registry's documentation structure, identifies the handful of hub URLs that matter (documentation root, blog root, security page, policy index), locates the RFC or proposal system if one exists, determines which URL patterns to block (package pages, user profiles, search results — anything that reflects user-generated content rather than registry policy), and captures a short interview's worth of human-only knowledge: any questions you already know the answer to from private sources, any questions known to be permanently unanswerable. It writes a starter-kit JSON and stops.

The brief is explicit about what it is *not*: the starter-kit builder does not read policy content, does not try to answer scorecard questions, does not populate anything from its own research. If it finds itself going deep into the registry's documentation to determine whether a security control exists, it has gone out of scope. The prompt says this three separate times in three slightly different ways because I found that the model, given the option, would drift toward helpful research every single time.

I run the starter kit on Sonnet. This is because Sonnet is extremely good at driving at the most obvious interpretation of a task by the most direct path; that tendency is a liability for the open-ended research job, but it is *exactly* the right instinct for source configuration, where the most obvious hub URLs usually are the right ones and you don't want the model to get clever.

Here's what a sample starter kit looks like, trimmed for length:

```json
{
  "schema_version": "1.0",
  "registry": "rubygems",
  "registry_url": "https://rubygems.org",
  "notes": "Operated by Ruby Central (nonprofit). Completed first external security audit by Trail of Bits in 2024. Trusted Publishing via OIDC launched December 2023. Mandatory MFA enforced for top-100 gem owners...",
  "sources": {
    "primary": [
      { "url": "https://rubygems.org/pages/security",     "description": "Security disclosure policy and HackerOne details" },
      { "url": "https://rubygems.org/policies",           "description": "Policies hub: acceptable use, copyright, privacy, ToS" },
      { "url": "https://guides.rubygems.org",             "description": "Documentation hub: security, trusted publishing, MFA" },
      { "url": "https://blog.rubygems.org",               "description": "Official blog: releases, security updates, features" }
    ],
    "secondary": [
      { "url": "https://github.com/rubygems/rubygems.org", "description": "Registry source code, issues, PRs" }
    ],
    "blocked": [
      { "pattern": "rubygems.org/gems/*",    "reason": "Individual gem pages are user content, not policy" },
      { "pattern": "rubygems.org/profiles/*","reason": "User profile pages are user content, not policy" }
    ]
  },
  "rfcs": {
    "url": "https://github.com/rubygems/rfcs",
    "acceptance_criteria": "RFC PR merged into the text/ directory. Merged RFCs become 'active' after a 7-day FCP.",
    "notes": "Concrete examples: text/0007-mfa-rollout.md (MFA rollout), text/0010-OIDC.md (trusted publishing)."
  },
  "pre_answered": {},
  "evidence_gaps": {}
}
```

Nothing exotic. A list of hubs, a list of blocked patterns, an RFC pointer with a concrete definition of what "accepted" means, and room for human-only knowledge. It takes about ten minutes to produce and it is the single most important piece of context the assessment agent will ever see.

### Do the actual thing

**`/assess`** is where the real work happens. The assessment agent works through all 52 questions in section order. For each question it follows a source hierarchy: primary sources first (the hubs the starter kit identified), then RFCs if applicable, then secondary sources, and only as a last resort the open web. It is not permitted to answer a question from inference; every answer must be accompanied by a direct quote from a real URL. It produces two files: a `scorecard-*.json` with the final answers and their citations, and a `runtime-*.json` with the complete evidence trail — every query issued, every URL visited, every candidate quote, every decision point. Same timestamp for both. The runtime file is resumable: if the run is interrupted, I can pick it up on the same question where it stopped.

The assessment agent runs on Opus. Opus has a specific trait that Sonnet does not, which is that it likes to play with language. Left to its own devices, it will re-frame a question, re-phrase a search query, try a synonym, worry at a word. That is a liability when you need a direct path to an obvious answer. It is a *feature* when you need to interrogate a specific claim from different angles before you trust it. I'll come back to that.

One piece of context that matters before the next section. Earlier I said that running the same assessment three times produced three different answers. That wasn't a quirk of the attempt-two architecture; it's a property of LLM inference in general, and it's worth understanding why. Horace He and colleagues at Thinking Machines Lab published an essay in September 2025 called *Defeating Nondeterminism in LLM Inference* that pins the blame on a specific cause: "The primary reason nearly all LLM inference endpoints are nondeterministic is that the load (and thus batch-size) nondeterministically varies."[^he] The floating-point non-associativity story you hear in passing (different GPUs, different rounding) is the symptom; the underlying cause is that production inference systems batch requests together and the batch composition varies. Which means — and this is the useful bit — run-to-run variance is a *systems-design property*, not a property of the universe. It can be reduced, designed around, or embraced. You just have to pick.

Thus, my design does not try to eliminate variance. It logs it exhaustively and exploits it.

## Getting the answers right: operational versus behavioural prompting

The assessment agent's system prompt is about two hundred and fifty lines of markdown. Every line is load-bearing because every line addresses a specific failure I observed in a previous run's evidence trail. The prompt has grown, but grown in a different sense from the attempt-two orchestrator: not accumulating defensive handlers, but encoding lessons.

The prompt splits cleanly into two kinds of instructions, and the distinction between them is the single thing I most wish I'd understood much earlier in my journey of discovery: operational vs behavioural instructions.

**Operational instructions** tell the agent *what to do and in what order, using which tools*. They are procedural: *parse these arguments; load the starter kit file; initialise the runtime file with this schema; call `jq` to extract questions matching the filter; write the runtime file after each completed section*, etc. Operational instructions are the kind of thing you'd write for a shell script if a shell script could handle the task, and indeed wherever a piece of operational behaviour is genuinely deterministic the right move is to pull it out into an actual shell script and have the agent invoke that. A specific example is *"Use `jq` to manipulate JSON. Do not parse or edit JSON as text."* because the model, left to its own devices, will happily reimplement JSON parsing and get it subtly wrong.

This generalises into a principle I've ended up putting at the top of most agent prompts I write now: **tooling over LLM.** The model should be the orchestrator and the reasoner of last resort. For deterministic sub-tasks—parsing, filtering, counting, scheduling, any operation where we have fifty years of reliable tooling already—use that tool. Don't let the model confabulate its way through work that a deterministic two-line `jq` expression would get right every time.

**Behavioural instructions** tell the agent *how to think about the problem*. They are about judgement: what counts as evidence, when to keep searching, how to interpret ambiguous documentation, what "enforce" means in contrast to "support." They cannot be replaced with a shell script, because the work they describe is exactly the work we are hiring the model for.

You can see the split most clearly in how I write new rules. Every time I review a runtime file and find a failure mode, I ask: *is this a thing the model got wrong because I told it to do the wrong procedure, or because it reasoned in a way I don't want?* Procedural failures get operational rules: *"When you cite evidence, record `url`, `quote`, and `source_type` in a single object, not as parallel lists."* Reasoning failures get behavioural rules: *"A quote must be content from the source, not a description of your own search process."* Confusing the two produces prompts that are both longer and less effective.

### A small team of researchers with varying expertise

I find it useful to think about agent prompts the way I'd think about briefing a team of contractors. You would not walk into a room of six analysts and deliver one monolithic paragraph of instructions. You would break the work into specified sub-tasks with clear handoffs; you would attach behavioural norms to each one: *"be skeptical of vendor marketing," "quote sources verbatim," "keep searching when the evidence is ambiguous"*, and so forth. Naturally, each member of the team has varying expertise, which means things that seem obvious might need to be spelled out (sometimes in excruciating detail).

The agent prompt is therefore that briefing document, written for an extremely fast but extremely literal contractor who has never worked with you before and forgets everything between engagements. Things like, "please don't make things up," "don't stop at the first promising result," "here is what I mean by 'enforce'". You have to say these things to the agent, because it will *assume* whatever you don't specify, and those assumptions are probably wrong. The prompt length is important. If it's too long the model will get confused; but if it's too short, the assumptions creep in. Unfortunately there is no One True Way to get this right—you just have to tune it as you go.

### Non-determinism can be your friend

The single most interesting thing in the behavioural half of the prompt is the way it treats non-determinism. I had spent attempts two and three trying to *suppress* the model's variance, and it never worked. In attempt four, I flipped the polarity. Every question gets re-phrased in three different ways before any searching happens at all.

The instruction, verbatim from the prompt:

```
**Step 1 — Generate queries**: produce 3 query variations from the question
text covering different angles:
- Direct phrasing of the question
- Synonym-heavy rephrasing
- A different aspect or implication of the question
```

Three variations, three different search passes per source, all three run even when the first one returns a promising result. This is weaponised non-determinism. The model's tendency to riff on language—the very thing that made Opus annoying for the starter-kit job—becomes the mechanism by which we can interrogate a claim from angles that I wouldn't have thought of. If three different phrasings of *"does this registry enforce MFA for publishers?"* all return the same answer from the same source, I can believe the answer. If they don't, I have contradictory evidence to work through, which is fine. Contradictory evidence is information. Missing evidence is not.

The underlying pattern has academic support. Gao and colleagues published a paper in 2022 called *Precise Zero-Shot Dense Retrieval without Relevance Labels* that introduces a technique called HyDE — the model generates *hypothetical answer documents* for a query, and those documents are then used as the retrieval key.[^hyde] It's a particular application of a general insight: one phrasing of a question is a sample, not a signal. Sample more.

And running all three variations *even after a promising first result* is a direct hedge against confirmation bias, which is a real and present threat with these models. The prompt is explicit about it:

```
Run ALL 3 even after a promising result on query 1 — seek contradictions,
not just confirmation.
```

This is one of the patterns of what I think of as a **rule ↔ failure-mode table**: a handful of behavioural lines, each of which exists because the model, unsupervised, failed in that specific way often enough that I had to write the rule to make it stop.

| Rule | Failure mode it addresses |
|---|---|
| "Inference is not evidence. Every answer requires a direct quote from a source URL." | Model confabulates plausible-sounding justifications in place of sourcing. |
| "Run all three query variations even after a promising first result." | Confirmation bias — model stops searching the moment it finds a confirming sentence. |
| "A quote must be content from the source, not a description of your own search process." | Model narrates its own activity and treats the narration as evidence. |
| "Ambiguous evidence means keep searching, not stop." | Default inclination toward closure — model wants to finish a question, not leave it open. |
| "Corroborating evidence from a different URL is desirable; from the same URL it is noise." | Model double-cites the same page to appear to have more evidence than it does. |
| "When a page contains both historical/transitional and current-state language, always prefer the current-state sentence." | Model quotes an older announcement while ignoring the newer one on the same page. |

Every one of these rules took at least one painful runtime-file review to discover, and most of them I got wrong the first time. "A quote must be content from the source, not a description of your own search process" in particular was a bizarre pathology to encounter: the model would, when it couldn't find evidence, *write a sentence describing its own search* and put that in the evidence field. Things like: *"Review of the security documentation did not reveal an enforcement policy for MFA."* That is not evidence, but it *looks* like evidence to the model, which is much worse.

There's a deep result here that Shankar, Zamfirescu-Pereira, Hartmann, Parameswaran, and Arawjo named in a UIST 2024 paper called *Who Validates the Validators?*: they call it **criteria drift**. Users building LLM applications, they observed, "need criteria to grade outputs, but grading outputs helps users define criteria."[^shankar] You cannot write all of your evaluation rules in advance. You have to look at outputs, notice where they go wrong, and write a rule. Then you look at more outputs, notice that a rule you wrote has a loophole, and refine it. The rule ↔ failure-mode table above didn't come from thinking carefully about the problem. It came from reading hundreds of runtime files and writing down what I saw.

This is a big part of what observability looks like for an LLM agent. Not a dashboard of request latencies (though that can be interesting, of course). Instead, what was useful to me was a full transcript of every decision the agent made, in a schema I can read, for questions I know the answer to, so I can compare what the agent did to what it should have done. Every entry in the runtime file is a sample in the distribution of the agent's behaviour, and the prompt is the mechanism I have to *shape* that distribution.

### What it actually looks like

Here's a slice of a real runtime file: the agent's work on Q1.1 ("Does the source enforce multi-factor authentication for publishers?") from a RubyGems assessment.

```json
"Q1.1": {
  "question": "Does the source enforce multi-factor authentication (MFA) for publishers?",
  "status": "answered",
  "answer": "NO",
  "searches": [
    { "query": "site:guides.rubygems.org MFA enforce multi-factor authentication required publishers",
      "url_scope": "guides.rubygems.org",
      "result_url": "https://guides.rubygems.org/mfa-requirement-opt-in/" },
    { "query": "site:blog.rubygems.org MFA enforce mandatory multi-factor authentication all users",
      "url_scope": "blog.rubygems.org",
      "result_url": "https://blog.rubygems.org/2022/08/15/requiring-mfa-on-popular-gems.html" },
    { "query": "site:blog.rubygems.org mandatory MFA all gem owners require 2024 2025 2026",
      "url_scope": "blog.rubygems.org",
      "result_url": "https://blog.rubygems.org/2023/12/14/trusted-publishing.html" },
    { "query": "rubygems.org mandatory MFA all users 2025 2026 enforce requirement",
      "url_scope": null,
      "result_url": "https://rubycentral.org/news/securing-rubys-future..." },
    { "query": "rubygems.org MFA enforcement expanded lowered threshold 2024 2025 all gem owners",
      "url_scope": null,
      "result_url": null },
    { "query": "site:github.com/rubygems/rfcs MFA rollout mandatory all users",
      "url_scope": "github.com/rubygems/rfcs",
      "result_url": "https://github.com/rubygems/rfcs/blob/master/text/0007-mfa-rollout.md" }
  ],
  "evidence": [
    { "url": "https://blog.rubygems.org/2022/08/15/requiring-mfa-on-popular-gems.html",
      "quote": "Today (August 15th, 2022), we will begin to enforce MFA on owners of gems with over 180 million total downloads.",
      "source_type": "primary" },
    { "url": "https://guides.rubygems.org/mfa-requirement-opt-in/",
      "quote": "You can opt-in a gem you are managing by releasing a version that has metadata.rubygems_mfa_required set to true.",
      "source_type": "primary" },
    { "url": "https://github.com/rubygems/rfcs/blob/master/text/0007-mfa-rollout.md",
      "quote": "Once these policy changes are fully complete for maintainers of the most popular gems, we intend to increase coverage by extending the MFA requirement to more gems in future.",
      "source_type": "rfc" }
  ],
  "notes": "RubyGems enforces MFA only for owners of gems exceeding 180 million cumulative downloads (roughly the top 100 gems), not for all publishers. For other gems, MFA enforcement is opt-in via the rubygems_mfa_required metadata flag set by individual gem maintainers. The accepted RFC (0007-mfa-rollout) describes phased expansion but defers broader enforcement to a future RFC that has not materialised. Ruby Central encourages MFA adoption but has not mandated it universally."
}
```

Six searches across three domain-scoped sources, two open-web fallbacks, and an RFC. Three evidence items from three different URLs, each with a direct quote and a source-type tag. A notes field that a human reviewer can read and understand the answer from. The full trail is much more informative than a yes/no bit on its own would be, because the reasoning trail is right there.

### Trust, but verify

One behaviour this example demonstrates that I want to call out specifically is the *theory-vs-reality gap*. This one is informal security folk knowledge and, as far as I can tell, isn't systematised anywhere in the literature. The security community has rigorous frameworks for lots of things, but the gap between what a registry (or anything, really) *documents about itself* and what it *actually does* remains a largely informal concern.

The agent surfaces it every few registries. Q1.1 is a soft example: RubyGems documents a phased MFA policy that *sounds* mandatory at a quick read but turns out, on careful reading, to apply only to the top 100 gems by download count. Other questions have turned up harder examples. Treating documentation as a starting point for investigation rather than ground truth, and forcing the agent to quote the enforcement language itself rather than paraphrasing its flavour, is one of the main things that lifts the agent above a glorified documentation summariser.

There's a closely related failure mode worth calling out because it's specific to this kind of research and because an LLM handles it badly by default: *absence of evidence.* If the agent searches the documentation for "vulnerability disclosure policy" and finds nothing, a human researcher reads that as meaningful evidence that the policy probably doesn't exist. The agent reads it as evidence that *it hasn't found the answer yet*. It keeps searching, dutifully, and will eventually reach for the open web or mark the question unknown rather than drawing the conclusion a human would draw in thirty seconds. Altman and Bland's famous note on this distinction — *Absence of evidence is not evidence of absence* — was written about clinical trials in the BMJ in 1995, but it applies here, too: a null result is not the same as a negative result, and how you handle the distinction is a design choice.[^altman] I resolve it by instructing the agent to exhaust its sources before marking a question `unknown`, and by treating `unknown` as a first-class answer—not a failure mode, not something to avoid, but an honest reflection of what the public documentation does and doesn't tell us.

## The human in the loop

Even with all of the above, the agent gets some questions wrong. This is expected, because confabulation is a property of the underlying system, not a bug I can fix in a prompt. The question is what you *do* with the wrongness.

What I do is run each assessment multiple times. Typically three, sometimes more if I have the patience. Each run is cheap: I'm not in a hurry, the agent works in series, I can do something else while it runs. I do this because the runs tend to disagree with each other in a very specific way.

Most questions stabilise. Let's say that 90% of them answer the same way every time, with the same evidence, and when I spot-check those against the documentation they're right. A small group (typically three to five questions, depending on the target) flip between runs. Sometimes between YES and NO, sometimes between one of those and UNKNOWN, sometimes cycling through all three.

Those flips are precisely what Farquhar et al. call *confabulations*: rerun-variant arbitrary wrong answers. The flip rate is, in effect, a probabilistic quality signal that doesn't require a ground-truth oracle. I don't need an answer key. I only need to notice that the model hasn't converged on an answer across multiple tries, and that by itself tells me the question is above the agent's reliable resolution.

Those flipping questions are exactly where I spend my review effort. I go look at them myself, read the documentation carefully, follow the evidence the agent did and didn't find, and produce an answer I'm willing to stand behind. Then I put that answer into the starter kit's `pre_answered` block, with the URL and the quote I used to resolve it. The next run (and every run after it) starts by copying that pre-answered entry into the scorecard verbatim and moving on.

This is the part I like best about the design. Every assessment makes the next one *faster and more accurate*. The system accumulates a data asset over time; the starter kit starts out containing hubs and blocked patterns, and over the course of a few assessments, it gains a growing set of human-resolved answers to the specific questions the agent couldn't handle. My human effort isn't thrown away on each run: it's captured in a durable input to the next one.

I hesitate to call this a ratchet because ratchet implies irreversibility, and human-resolved answers can be wrong or go stale. The starter kit's pre-answered section is versioned with the registry's policy as of a given date, and we'll need to revisit it when things change. But within a normal cadence of review, the direction is one-way: the work the agent can do reliably, it does, at scale; the work it can't, I do, once, and capture the result. Breadth from the agent, depth from me, and both compounding into a system that covers more surface area with every registry we touch.

## The agent itself is an attack surface

I wasn't originally going to mention this, but it's important—especially for an audience that thinks about security for a living. Automated agents that have access to the open web are *dangerous*.

An LLM agent that *browses the web* (i.e. reads arbitrary pages, follows links, extracts text it then feeds back into its own reasoning) sits inside a threat model that is fundamentally different from one that only operates on trusted local inputs. Simon Willison coined a phrase for it in June 2025 that I've found clarifying: the *lethal trifecta*. Three capabilities that, when present together, make an agent a serious security liability:

1. Access to your private data
2. Exposure to untrusted content
3. The ability to externally communicate in a way that could be used to steal your data[^willisontrifecta]

A registry-assessment agent has all three. It has private data: my local environment, whatever else is on my machine that Claude Code can see. It has exposure to untrusted content: every web page it fetches is attacker-controlled from the agent's perspective. And it obviously has external communication: it can make outbound HTTP requests.

The specific threat this creates is *indirect prompt injection*. A hostile web page can contain instructions aimed at the agent rather than the human reader. Stuff hidden in HTML comments, like in text with CSS that sets it to `display: none`, in image alt text, or in JavaScript that modifies the rendered content. Those instructions get read by the agent as part of its retrieved context. Greshake and colleagues formalised this class of attack in a 2023 paper called *Not what you've signed up for*: "LLM-Integrated Applications blur the line between data and instructions... Adversaries can remotely exploit LLM-integrated applications by strategically injecting prompts into data likely to be retrieved."[^greshake]

It is not a hypothetical concern. Brave Software's security researchers published a technical write-up in August 2025 of an indirect-prompt-injection attack against Perplexity's Comet agentic browser. The money line: "When an AI assistant follows malicious instructions from untrusted webpage content, traditional protections such as same-origin policy (SOP) or cross-origin resource sharing (CORS) are all effectively useless."[^comet] The browser-level security model we've spent thirty years building does not apply at the level where an AI agent is making decisions, because the decisions are happening in LLM-land, not in browser-land.

My response to this is to run Claude Code, and therefore the assessment agent, inside a sandbox called Nono. Nono is a capability-based sandbox that uses kernel-level isolation (Landlock on Linux, Seatbelt (`sandbox-exec`) on macOS) to restrict a process to an explicit allow-list of filesystem paths and network endpoints.[^nono] Once a profile is applied, the restrictions are irreversible and inherited by every child process. There's no API to loosen them from inside, including for Nono itself. If the agent is instructed by a hostile page to exfiltrate my SSH keys, it can try, but the attempt is blocked at the kernel level before any connection is made. Nono is maintained by Luke Hinds, who also created Sigstore, so the provenance on the sandbox itself is reassuring.

I don't mention Nono as a product pitch. I mention it because the threat model is real, it is named, and I would rather run a browsing agent inside a kernel-enforced capability sandbox than trust the LLM to follow my instructions under adversarial conditions. The lethal trifecta is a concrete list of capabilities; if your agent has all three, it is worth taking a beat to think about what happens when the untrusted content in the middle position starts fighting back.

Anyway, back to the regularly scheduled programming.

## OK, but does it work?

I ran the same 52-question set through Gemini Deep Research as a sanity check before I started work on the fourth attempt. Gemini Deep Research is a reasonable state-of-the-art benchmark for AI-assisted research (it's the product that Google points you at if you ask for "do some research on topic X and produce a report.") I fed it the full 52 questions and the same registry context I was giving my own system, and compared the outputs.

The results were instructive. Gemini hallucinated questions that weren't on the scorecard. It produced sections that were off-topic for the questions it did answer. Some answers were wrong in ways that were hard to miss; others were wrong in ways I only noticed because I happened to know the answer. And it disagreed with itself on reruns. Its run-to-run drift was roughly 20 percent, so about one question in five would change its answer between runs, or not get answered at all. This gave me a concrete target, and my 7 to 10% drift was a clear improvement on that benchmark.

But the drift rate isn't really the headline. The headline is that I know *which* questions drift. The four-to-five-questions-per-run figure isn't a diffuse uncertainty spread over the whole scorecard; it's a concentrated uncertainty in a specific subset of questions, which is simple to enumerate. My review is targeted rather than distrustful. I get to trust most of questions and spend my time examining the handful that actually need it.

Concretely: what used to take a day and a half of careful reading now takes about an hour of agent time (plus the compute cost, which is genuinely negligible at this scale) and two to three hours of human review. That's a 3× to 5× speedup on the part of the work that *can* be accelerated, and meaningful ecosystem coverage that would otherwise require headcount I don't have.

## Research assistant, not leader

I want to be honest about what this system is and isn't.

It is a research *assistant*. It has meaningfully accelerated my ability to produce assessments across an ecosystem that would otherwise require either much more time or much more labour. It is *not a replacement for human judgement*, and given the current state of the art, I don't believe it can be. The questions where sources contradict one another and someone has to decide which to trust; where you need to understand that a registry was acquired by a company with a different security culture and factor that into how you read their documentation; where a new threat has emerged that isn't on the 52-question list yet… That all still requires a human who understands the domain, has read the security community's work on similar registries, and can integrate signals that aren't in any web-accessible document.

The split that actually works, after four attempts at getting it right, is this: the agent handles the grind, but the hard calls remain mine.

I think that's fine. "Research assistant, not leader" is an honest description of what AI tooling can do well in a domain that requires nuanced judgement. The temptation to overclaim—to pretend the agent is doing the research rather than accelerating the parts of research that benefit from being done at scale—is real, and the corrective is a small dose of deflation. This thing doesn't do my job. It makes me faster at the parts of my job that can be made faster, and it captures my slow-and-careful work in a form that speeds up the next round. That's enough.

## Closing: what generalises

While the outcome is specific, the lessons aren't. They are, I hope, useful if you are building any kind of research or evaluation agent—or more generally, any system where an LLM has to produce answers a human will rely on.

**Mechanical failures and quality failures look similar from the outside but have entirely different causes.** Separate them early, because the toolkits are different and the techniques don't transfer. A hung process wants a retry; a confabulation wants a prompt rule. If you're shipping more schema validation to fix confabulations, you are doing the wrong thing.

**Tools have defaults.** Choose them by their instincts, not by their spec sheets. The same model can behave very differently in different harnesses, and if your harness's default instinct is fighting your problem you will spend most of your energy fighting the harness. Pick the one that matches your problem's shape.

**Operational and behavioural instructions are different jobs.** Write them separately. Procedural rules want to be simple and unambiguous ("use `jq`, don't parse JSON yourself"); behavioural rules want the kind of precision that comes from reviewing failures ("a quote must be content from the source, not a description of your search process"). If your prompt feels bloated, it may be because the two kinds of rule are tangled together and neither is doing its job well.

**Observability is not optional for agents.** A structured runtime file (every search, every URL, every candidate quote, every decision) is not paranoia and it is not for debugging after the fact. It is the raw material you use to understand what the model actually does, which is what turns prompt refinement from guesswork into iteration. If you can't see what the agent did, you can't tell it to do better.

**Non-determinism is both antagonist and tool.** Crush it where you need reproducibility: structured outputs, deterministic tools, runtime logs you can replay. Weaponise it where you need creativity: ask the same question three different ways, sample more phrasings than you think you need, let the model's tendency to riff work for you. Both at once, applied to different layers of the same system.

**The best human-in-the-loop designs capture the human's work as durable input to the next run.** If your system relies on a human to resolve the cases the agent can't handle, the cost of that human's work should decrease over time as the system learns which kinds of cases it can't handle and pre-routes them. A system that makes you re-resolve the same problem on every run is a worse system than one that gets out of your way as you teach it.

I came to this project expecting to build something clever and instead learnt several old engineering lessons by running straight into them. While few of the lessons were genuinely new, all of them were surprising, in context, because LLMs are new enough that we haven't yet mapped our existing engineering intuitions onto them correctly. You cannot code your way out of a quality problem. You cannot make a non-deterministic system deterministic by putting schemas around it. Observability beats sophistication. Small sharp tools beat frameworks. Write a runtime log. Measure twice, cut once.

---

## Notes

[^sonatype]: Sonatype, *10th Annual State of the Software Supply Chain Report*, 2024. https://www.sonatype.com/state-of-the-software-supply-chain

[^slsa]: *Supply-chain Levels for Software Artifacts* (SLSA), v1.2. https://slsa.dev. "SLSA is a set of incrementally adoptable guidelines for supply chain security, established by industry consensus."

[^sigstore]: The Sigstore project. https://docs.sigstore.dev/about/overview/. An OpenSSF-hosted project for signing and verifying software artefacts.

[^openssfurl]: OpenSSF Scorecard. https://github.com/ossf/scorecard

[^farquhar]: Sebastian Farquhar, Jannik Kossen, Lorenz Kuhn, and Yarin Gal, "Detecting hallucinations in large language models using semantic entropy," *Nature* 630, 19 June 2024. https://www.nature.com/articles/s41586-024-07421-0. "We develop entropy-based uncertainty estimators for LLMs to detect a subset of hallucinations — confabulations — which are arbitrary and incorrect generations."

[^kalai]: Adam Tauman Kalai, Ofir Nachum, Santosh S. Vempala, and Edwin Zhang (OpenAI), "Why Language Models Hallucinate," arXiv:2509.04664, 4 September 2025. https://arxiv.org/abs/2509.04664

[^hicks]: Michael Townsen Hicks, James Humphries, and Joe Slater, "ChatGPT is bullshit," *Ethics and Information Technology* 26:38, 8 June 2024. https://link.springer.com/article/10.1007/s10676-024-09775-5. "The falsehoods and overall activity of large language models is better understood as bullshit in the sense explored by Frankfurt... the models are in an important way indifferent to the truth of their outputs."

[^frankfurt]: Harry G. Frankfurt, *On Bullshit* (Princeton University Press, 2005; originally a 1986 essay). "He does not care whether the things he says describe reality correctly. He just picks them out, or makes them up, to suit his purpose."

[^huyen]: Chip Huyen, "Data Distribution Shifts and Monitoring," 7 February 2022. https://huyenchip.com/2022/02/07/data-distribution-shifts-and-monitoring.html

[^husainshankar]: Hamel Husain and Shreya Shankar, "LLM Evals: Everything You Need to Know," January 2026. https://hamel.dev/blog/posts/evals-faq/

[^yan]: Eugene Yan, "An LLM-as-Judge Won't Save The Product — Fixing Your Process Will," April 2025. https://eugeneyan.com/writing/eval-process/

[^harness]: HumanLayer, "Skill Issue: Harness Engineering for Coding Agents," March 2026. https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents

[^anthropicagents]: Erik Schluntz and Barry Zhang, "Building effective agents," Anthropic Engineering, 19 December 2024. https://www.anthropic.com/engineering/building-effective-agents

[^willisondef]: Simon Willison, "I think 'agent' may finally have a widely enough agreed upon definition to be useful jargon now," 18 September 2025. https://simonw.substack.com/p/i-think-agent-may-finally-have-a

[^he]: Horace He et al. (Thinking Machines Lab), "Defeating Nondeterminism in LLM Inference," 10 September 2025. https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/

[^hyde]: Luyu Gao, Xueguang Ma, Jimmy Lin, and Jamie Callan, "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE), arXiv:2212.10496, 2022. https://arxiv.org/abs/2212.10496

[^shankar]: Shreya Shankar, J.D. Zamfirescu-Pereira, Björn Hartmann, Aditya G. Parameswaran, and Ian Arawjo, "Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences," UIST 2024 / arXiv:2404.12272. https://arxiv.org/abs/2404.12272

[^altman]: Douglas G. Altman and J. Martin Bland, "Absence of evidence is not evidence of absence," *BMJ* 311:485, 19 August 1995. https://www.bmj.com/content/311/7003/485

[^willisontrifecta]: Simon Willison, "The lethal trifecta for AI agents: private data, untrusted content, and external communication," 16 June 2025. https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/

[^greshake]: Kai Greshake, Sahar Abdelnabi, Shailesh Mishra, Christoph Endres, Thorsten Holz, and Mario Fritz, "Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection," arXiv:2302.12173, February 2023. https://arxiv.org/abs/2302.12173

[^comet]: Artem Chaikin and Shivan Kaul Sahib, "Agentic Browser Security: Indirect Prompt Injection in Perplexity Comet," Brave Blog, 20 August 2025. https://brave.com/blog/comet-prompt-injection/

[^nono]: Nono sandbox. https://github.com/always-further/nono. Capability-based sandbox using Landlock (Linux) and Seatbelt (macOS).
