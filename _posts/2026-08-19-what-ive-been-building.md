---
title: What I've been building lately (August 2026)
---

It's been a while since I've written on my blog!
A lot has changed in the software engineering industry in the last year with AI adoption everywhere, so there's a lot that I could write about.
But I've also spent a lot of time reading other folks' musings about AI, and at the current moment in the year it feels like the topic has been lingering in the air so long that the air has gotten a bit stale.
Perhaps I'll aggregate some of my favorite writings and link them in a future post.

Instead, I'd like to share about what I've been building, learning, and hacking on in the past half year or so.
Here's the table of contents -- so make yourself at home and feel free to jump around to whatever part(s) interest[s] you.

1. [tetromino.today](#tetrominotoday) - a vibe-coded daily puzzle game where you score points by optimally placing tetromino pieces
2. [stability-sim-lib](#stability-sim-lib) - a TypeScript library for simulating distributed systems
3. [Learning TLA+](#learning-tla) - my two month excursion self-teaching myself TLA+ on weekends
4. [precept](#precept) - an AI-based informal verification CLI tool for verifying properties throughout your codebase

There's actually several more side projects and excursions I was planning to include but didn't have time to write about.
But I want to get into the habit of publishing posts more often, so I'll try to write about the remaining stuff in a follow-up post when I find the time.

## tetromino.today

I vibe coded a game!

In [tetromino.today](https://tetromino.today/), a "daily" game inspired by the likes of [enclose.horse](https://enclose.horse/) and Wordle, your goal is to score as many points as possible by optimizing the layout of Tetris pieces on a grid.

Each day, there is a different grid layout (with different missing squares, different bonuses), as well as a different set of [tetrominos](https://en.wikipedia.org/wiki/Tetromino) worth varying numbers of points.
Your goal is to maximize your score by fitting and arranging pieces on the grid as best you can.
There's no time limit, so several times while playing I've found myself fiddling around for minutes to improve my score and find a better arrangement of pieces.

Once you submit your score, it's locked in, and you can see how it compares to others on the leaderboard, as well as what the optimal score is.

<center>

<img src="/blog/assets/images/tetromino-today.png" width="400">
<br>
<i>Screenshot of tetromino.today.</i>
<br>
<br>

</center>

This felt like the kind of game that I ordinarily might not have had enough motivation to build end-to-end unless I had access to AI agent.
The gameplay logic itself isn't that complex conceptually, so most of the effort to build the game was implementing the code so pieces can be placed on a grid in relation to a mouse cursor in a grid, and enforcing that tetromino don't end up with pieces on top of each other, pieces behave as expected when rotated, and so on.
There was also quite a bit of work involved to make the game's UI responsive and working on both desktop and mobile.

It's not trivial to get all of these parts working together harmoniously!
But most of the time I was more excited by the prospect of getting the game working and playable.
Having AI agents meant I could get faster to a version of the game that worked where I could experiment with tweaking the rules and the daily puzzle generator.

I think the experience of building this really surprised me because it was the first time I was using AI and *actually* letting go of the wheel and not trying to read every single line of code, and it ended up being totally OK.
The implementation wasn't one-shot by any means: after the game was implemented, at least a few hours of back and forth was spent iterating on the UI and fixing UI bugs, where I often gave my agent the description of various layout bugs and it figured out how to fix them through changes to the CSS and JavaScript.

All that said, I'd still like to rework the game in a few ways.
First, I'd like to redesign the UI to be more polished (a la [tiledwords.com](https://tiledwords.com/)).
Secondly, I'm currently hosting the service on a [fly.io](https://fly.io/) legacy hobby plan and storing user scores on a [neon.com](https://neon.com/) Postgres database, but the Neon free tier doesn't cover a full month's usage -- so I'm thinking of trying to use SQLite on top of a Fly volume instead, or trying out [turso.tech](https://turso.tech/)'s hosted SQLite free tier.

## stability-sim-lib

[stability-sim-lib](https://github.com/Chriscbr/stability-sim-lib) is a TypeScript library I used AI to extract out of a web app called [Stability Sim](https://stability-sim.systems/) by [Marc J Brooker](https://brooker.co.za/blog/) (well known engineer and writer in the distributed systems community).

As a quick background, Stability Sim is a "simple, interactive, simulator that allows you to explore some of the behaviors that cause long outages even in simple  distributed systems, and understand the pitfalls of caches, simple retry  strategies, round-robin load balancing, and other common patterns." [^1]
Basically it's a tool that lets you model the traffic patterns of high-level distributed systems and measure how each part of the system responds. And the simulation is visualized through a neat web app.

I had a lot of fun playing around with the examples provided in the app, but I imagined there are situations where you'd like an AI agent to try and use this simulator to model some traffic patterns and see how it would behave for you.
But one thing I've learned is that when you want to give an agent the ability to build and iterate on something, an agent is more effective when you can give it an API (or CLI, or MCP, or some other programmatic interface) than when you ask it to click and type in a browser.
So I thought I'd try using AI agents to see how easy it would be to extract the web app into a library that you can write code with.

Since the simulation was already built in TypeScript, I decided to keep the library in TypeScript.
AI is pretty good at porting code between languages though, so making the library available in another language probably isn't out of the question.

Below is an example of what a simulation written with the library looks like.
The API design is heavily inspired by [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html). [^2]

[^2]: I worked on the AWS CDK team for my first year out of college.

```typescript
import { Client, Distribution, LoadBalancer, LoadBalancerStrategy, Queue, Retry, Scenario, Server, ServerCrash, Traffic } from 'stability-sim';

const scenario = new Scenario({
  name: 'Metastable Retry Storm',
  endTime: 60,
  metricsWindowSize: 1,
  seed: 42,
});

const client = scenario.add(new Client('client-1', {
  trafficPattern: Traffic.openLoop({ meanArrivalRate: 70 }),
  retryStrategy: Retry.fixedN({ maxRetries: 3 }),
  timeout: 1,
}));

const loadBalancer = scenario.add(new LoadBalancer('lb-1', {
  strategy: LoadBalancerStrategy.ROUND_ROBIN,
}));

const queueA = scenario.add(new Queue('queue-a', {
  maxCapacity: 10_000,
  maxConcurrency: 5,
}));

const queueB = scenario.add(new Queue('queue-b', {
  maxCapacity: 10_000,
  maxConcurrency: 5,
}));

const serverA = scenario.add(new Server('srv-1', {
  serviceTimeDistribution: Distribution.exponential({ mean: 0.1 }),
  concurrencyLimit: 5,
}));

const serverB = scenario.add(new Server('srv-2', {
  serviceTimeDistribution: Distribution.exponential({ mean: 0.1 }),
  concurrencyLimit: 5,
}));

client.sendsTo(loadBalancer);
loadBalancer.routesTo(queueA);
loadBalancer.routesTo(queueB);
queueA.sendsTo(serverA);
queueB.sendsTo(serverB);

scenario.addFailure(new ServerCrash(serverA, {
  triggerTime: 5,
  recoveryTime: 10,
}));

const result = scenario.run({ snapshotInterval: 1 });
console.log(result.finalSnapshot);
```

In the end, the API worked as expected, and I published a [small NPM package](https://www.npmjs.com/package/stability-sim), though I'll admit I haven't used the library much yet.

One spin-off idea that I'd like to pursue in the future is I'd like to use Stability Sim to design an interactive online course (or micro-course, or series of interactive blogs) about lessons and mental models for building resilient distributed systems.
In a lot of ways, it's a fuzzy topic, and one I'm still learning about at my current job!
But my intuition here is that a lot of concepts would be easier to understand concepts and learn how to recognize distributed systems failure modes (like retry storms or connection pool exhaustion), as a student, when you can practice against dozens of examples.

[^1]: <https://x.com/MarcJBrooker/status/2044109805498052941>

## Learning TLA+

For this section there isn't a single "thing" I built per se, but I wanted to reflect on my journey earlier this summer to learn how to write specifications [^3] in [TLA+](https://en.wikipedia.org/wiki/TLA%2B).

[^3]: When I first wrote this blog post I kept using the term "TLA+ program" -- but I realized it didn't really sound right. I'm sure most people wouldn't get confused, since they are programs in the sense that they are files with code meant for a computer to read and interpret (in the same way that [Lean](https://en.wikipedia.org/wiki/Lean_(proof_assistant)) code could be called "programs"). But the word "program" usually implies some kind of side effect or some interactivity or computation based on inputs, so I think "specification" makes it clearer that the code you write is really just a formal description of a system, and nothing more.

My motivation was that with the explosion of AI agents, many folks in tech (including myself) realized how much easier it is to ship buggy code when it's not humans writing it.
Humans make plenty of bugs and mistakes too, let's be clear -- it's just that when AI make mistakes, the mistakes are less human-like and thus tend to be more surprising.
The bugs are also made at higher rates. [^4]
(I also attended a [meetup in NYC about software resilience hosted by AWS & Antithesis](https://youtu.be/xyR1S0a6GVE) which inspired me further to learn about formal modeling techniques.)

[^4]: An additional observation is that the mistakes AI agents make tend to be at higher levels of abstraction. Claude or ChatGPT is unlikely to make a typo error or to miss a file while doing a refactor, even when context sizes grow very large on long-horizon tasks. But it is likely to misunderstand design requirements that weren't made explicit. It's also more likely to design over-complicated abstractions and write verbose code. But this is just what I've seen from the current models available today.

Thus, the need for better tools for software correctness.
There's a lot of interesting tools and techniques in this space (type systems, linters, fuzz checkers, theorem provers, etc.) but the one I was most curious about was TLA+.

The main pitch for TLA+ is that it lets you write a declarative description of a system by representing it as a set of state machines and transitions.
In practice, "state machines and transitions" just means variables and rules for updating those variables.
It also lets you describe properties of the system, like "this kind of state should never happen" or "when operation X is performed, then state Y should eventually occur."
The syntax for writing the model and properties is somewhat dated, but if you can get past all that you'll discover that TLA+ is a highly general and surprisingly powerful tool.

My learning path was the following:

1. Read [Intro to TLA+ for the LLM Era: Prompt Your Way to Victory](https://emptysqua.re/blog/intro-to-tla-plus-for-the-llm-era/). This article assumes no prior TLA+ knowledge, and it works through a toy problem to give you an idea of what TLA+ specs are like, and introduces you to some high level concepts that you'll see again.
2. Go through the "Core" section on [learntla.org](https://learntla.com/core/index.html). This teaches the main TLA+ language (and the PlusCal dialect), though it assumes prior programming knowledge. It has some notes on how to set up TLA+ so you can run and edit specs locally.
3. Watch the [TLA+ Video Course](https://lamport.azurewebsites.net/video/videos.html). The material covered overlaps with learntla.org, but the videos are recorded by the creator of TLA+, [Leslie Lamport](https://en.wikipedia.org/wiki/Leslie_Lamport), so I found the pedagogical value very instructive. He also provides better explanations for advanced concepts like the different forms of fairness.

For every TLA+ spec I wrote while going through these materials, I saved it into a personal [tlaplus-playground](https://github.com/Chriscbr/tlaplus-playground) repo so I can modify or reference them later.

I think there's still a lot more algorithms and data structures I'd like to try modeling with TLA+, and some more content I'd like to read, but I feel confident that I have at least the foundation needed to read or write TLA+ specifications.
In particular, there were several resources I found online that I'd like to revisit at some point:

- [The PlusCal Tutorial](https://lamport.azurewebsites.net/tla/tutorial/contents.html) - reinforcement of existing concepts and pedagogy
- [Safety, Liveness, and Fairness](https://lamport.azurewebsites.net/tla/safety-liveness.pdf) - reinforcement and further background about advanced concepts
- [Stuttering Steps](https://lamport.azurewebsites.net/tla/stuttering.html) - further background about advanced concepts
- [Other talks and resources from Leslie Lamport's website](https://lamport.azurewebsites.net/tla/more-stuff.html) - examples and more
- [Specifying Systems](https://lamport.azurewebsites.net/tla/book.html?back-link=learning.html) - a book on TLA+ by Leslie Lamport
- [Practical TLA+](https://www.hillelwayne.com/post/practical-tla/) - a book on TLA+ by Hillel Wayne
- [Other examples from learntla.org](https://www.learntla.com/examples/index.html) - assorted blog posts and examples to work through
- [Introduction to Pragmatic Formal Modeling](https://elliotswart.github.io/pragmaticformalmodeling/) - works through concrete examples of modeling real world systems using TLA+
- [Apalache docs](https://apalache-mc.org/docs/index.html) - Apalache is the underlying model checker that actually does the "checking" work when you write a TLA+ specification. Their docs provides further examples and reference pages
- [Hillel Wayne blog posts](https://hillelwayne.com/post/) - includes many posts covering TLA+ examples and advice
- [TLA+ examples](https://github.com/tlaplus/Examples/tree/master) - examples of TLA+ programs

Some other TLA+ / formal modeling related spin off ideas still on my mind:

1. I'd like to create my own cheat sheet of TLA+ syntax and operators and standard library functions.
2. I'd like to learn more about similar model checking languages like [P](https://github.com/p-org/P) or [Quint](https://quint.sh/), and compare them side by side and build up a reference sheet of equivalencies and differences between the syntaxes of each tool.
3. I'd like to try my own hand at building a simpler syntax for TLA+. I believe this is similar to what Quint is trying to achieve (providing a more ergonomic syntax and better type safety story), but I'm not sure if I agree with all of their design choices / I'd also like to give a shot at a more Python-like syntax. [Fizzbee](https://github.com/fizzbee-io/fizzbee) also looks like an interesting attempt at this.

Finally (and most ambitiously) I'd like to experiment with building new developer tools to make model checking easier to adopt by ordinary programmers -- in the same spirit as [Jean Yang](https://www.jeanyang.com/)'s [Building for the 99% Developers](https://future.com/software-development-building-for-99-developers/).
At my current employer we write a lot of systems in Go and Rust.
I have seen some teams write TLA+ models for their systems.
But one of the main difficulties is that once the model is created, it tends to get thrown away or fall out of date, since there's usually little that can be done to keep these two artifacts synced together.

The "research question" here is: what ways are there to prevent programs and models from getting out of sync? How can we fundamentally colocate specification information and implementation information in a way that avoids constraining either the implementation world or the modeling world too much? I have no doubt that others have faced this question as well, so I'd like to read some of the existing literature and take a fresh look at the problem from first principles.

## precept

In the previous section, I explained how there has been a resurgence of energy in formal methods tooling following increased usage of AI coding tools.
Much of this energy has been focused on leveraging traditional formal methods techniques that existed before the 2020s -- stuff like model checking, type checking, static analysis, and so on.

However, concrete adoption of these tools is still limited, and I believe now that LLMs are so widespread, there are new opportunities to create developer tools that improve a codebase's robustness and maintainability using coding agent harnesses as first class citizens. [^5]
That's where [precept](https://github.com/Chriscbr/precept) comes in.

[^5]: If it's not clear from the previous section - my belief is that AI and formal methods can be complementary tools. One way to view `precept` is that it's *sort of* in a similar class of tools as AI code reviews. As many can attest, AI code reviews can be very helpful for catching bugs, but it's weaker than static analysis, type checking, and model checking in many ways.

`precept` is a CLI tool. It's really easy to use:

1. First, add comments to your code that look like `// INVARIANT: ...`, `// PRECONDITION: ...`, or `// POSTCONDITION: ...`. Each annotation is called a *claim*.
2. Then, run `precept verify <directory>` to have coding agent harnesses (like Claude Code or Codex) run in parallel to verify each claim in the codebase.

The harness, model, and reasoning effort used by precept can be configured by a configuration file [^6] or simply passed as options, like `--agent codex --model gpt-5.6-terra --effort low`.

When all of the agents are finished, it prints out a summarized result of verifying each claim. If the agent believes the claim holds, it will include supporting evidence, and it if believes the claim doesn't hold, a suggested counterexample will be included.
If it's not possible to fully validate the claim, it will generally flag the claim as `inconclusive` or `error` instead.

Here's an example of what the output looks like today:

```
✓ queue.Buffer [INVARIANT] (internal/queue/buffer.go:33)
  [outcome]: Holds
  [reason]: In the queue processor, PendingItems is read or mutated only before the processing goroutine starts, by that one goroutine, or after Stop waits for it to exit. Background delivery workers operate on a detached item slice, not PendingItems.
  [supporting evidence]:
    • Start reads the field before it launches b.process in a new goroutine. (internal/queue/buffer.go:62-68)
    • <snip - more reasons...>
  [agent session]: codex resume <session-id>

✗ queue.BuildBatches [INVARIANT] (internal/queue/batcher.go:444)
  [outcome]: Violated
  [reason]: A single item larger than the request-size limit is still added to a batch, so BuildBatches can return a batch whose total size exceeds maxRequestSizeBytes. It also permits one item when maxItemsPerBatch is zero.
  [supporting evidence]:
    • When the next item would exceed either limit, ... <snip>
  [counterexample]: Call BuildBatches with maxRequestSizeBytes set below the encoded empty request plus a valid QueueItem's size, maxItemsPerBatch=1, and one item that can be encoded. The size check resets the empty batch, then lines 477-479 add the oversized item to the new batch, which lines 481-482 return.
  [agent session]: codex resume <session-id>
```

By default, every coding agent subprocess is created with read-only permissions so they can only make judgements by reading the code.
But I plan to add support for customizing the sandboxing in case you'd like to allowlist certain MCP tools or actions for verification purposes.

I believe this kind of tool has a lot of potential for a few reasons.

First, it encourages properties and invariants to be colocated with relevant source code.
This isn't unique to precept, but I do think it's table stakes for making a tool useful in long-lived, evolving codebases.
Tools that expect you to write models, properties, or invariants in separate files are much more likely to get stale unless they are somehow being used to automatically code that the real system depends on.

Second, it's *extremely flexible*.
One of the biggest frictions for adopting verification tools based on code comments or annotations is the difficulty of expressing properties.
You're essentially learning a second coding language on top of the original source language, and that coding language might not even be able to express everything you want.
With precept, anything (from a syntactic pattern, to a mathematical rule, to a concurrency property, to a security property) can be expressed, because it depends on natural language text.

Lastly, it's non-invasive.
There was another tool in this space that was partial inspiration for precept called [aristo](https://github.com/aretta-ai/aristo).
Aristo centers on a similar idea where you add annotations/intents to your Rust code containing natural language descriptions.
But aristo requires you to use Rust-specific macros, and it adds these proof files of a proprietary format to your codebase after it runs.
`precept` is a lot lighter in comparison.
It just uses the comments built into your programing language.
Furthermore, the annotations tend to read like ordinary, human-written comments -- so they're useful while reading the codebase even if you don't have the `precept` CLI installed.

Ok, enough selling for now.
Anyways, I'm pretty bullish on it at least as an exploratory idea.
But there's a lot that still a lot more that I have to build.

For example, you might not want to verify every claim in your codebase on every single code change because that could cost a lot of tokens.
So I'd like to augment the tool to use static analysis to identify what functions have changed between a set of commits and the graph of impacted functions, so that you can run `precept verify --changed` to only re-verify claims whose code has changed.
Also, the tool only supports Go codebases today, but in principle it should be possible to support almost any language (even if not every static analysis feature works).

In any case, check out the [GitHub repository](https://github.com/Chriscbr/precept) to learn more.
It's just a plain old open source CLI written in Go.
Pull requests are welcome!

[^6]: Configuration file option not implemented yet.
