Title: The Hot Mess of AI: How Does Misalignment Scale with Model Intelligence and Task Complexity?

URL Source: https://alignment.anthropic.com/2026/hot-mess-of-ai

Markdown Content:
Research done as part of the first [Anthropic Fellows
                    Program](https://alignment.anthropic.com/2024/anthropic-fellows-program/) during Summer 2025.
        

When AI systems fail, will they fail by systematically pursuing the wrong goals, or by being a hot mess? We decompose the errors of frontier reasoning models into bias (systematic) and variance (incoherent) components and find that, as tasks get harder and reasoning gets longer, model failures become increasingly dominated by incoherence rather than systematic misalignment.

As AI becomes more capable, we entrust it with increasingly consequential tasks. This makes understanding
            *how* these systems might fail even more critical for safety. A central concern in AI alignment
            is that
            superintelligent systems might coherently pursue misaligned goals: the classic [paperclip
                maximizer](https://en.wikipedia.org/wiki/Instrumental_convergence#Paperclip_maximizer) scenario. But there's another possibility: AI might fail not through systematic
            misalignment, but through *incoherence*—unpredictable, self-undermining behavior that doesn't
            optimize for any consistent objective. That is, AI might fail in the same way that humans often fail, by being a
            **hot mess**.
        

This paper builds on the [hot mess theory
                of misalignment](https://sohl-dickstein.github.io/2023/03/09/coherence.html) (Sohl-Dickstein, 2023), which surveyed experts to rank various entities (including
            humans, animals, machine learning models, and organizations) by intelligence and coherence independently. It
            found that *smarter* entities are subjectively judged to behave *less* coherently. We take
            this hypothesis from
            survey data to empirical measurement across frontier AI systems, asking: **As models become more
                intelligent and tackle harder tasks, do their
                failures look more like systematic misalignment, or more like a hot mess?**
        

To quantify incoherence we decompose AI errors using the classic bias-variance framework:

We define **error incoherence** as the fraction of error attributable to variance:

The error incoherence is therefore a metric between 0 and 1: a value of 0 means that all errors are systematic
            and produce identical outcomes (analogous to classic misalignment risk); an error incoherence of 1 means errors
            are random (the hot mess scenario). When we scale models, total error naturally decreases (both bias and variance
            drop). We primarily care about the *composition* of this error: what do errors look like, rather than how
            frequently they occur. Measuring the relative contribution of variance disentangles error type and error rate.

We evaluated frontier

Across all tasks and models, the longer models spend reasoning and taking actions, the more incoherent their errors become. This holds whether we measure reasoning tokens, agent actions, or optimizer steps.

How does error incoherence change with model scale? The answer depends on the experimental setup.

While this relationship requires further study, our observations suggest that scaling alone won't eliminate incoherence in errors. This is especially true, as we expect more powerful models to perform longer and more complex tasks.

We find that when models spontaneously reason longer on a problem (compared to their median), error incoherence spikes dramatically. Meanwhile, deliberately increasing reasoning budgets through API settings only modestly increases coherence in errors. The natural variation dominates.

Aggregating multiple samples reduces variance (as expected from theory), providing a path to more coherent behavior, though this may be impractical for real-world agentic tasks where actions are irreversible.

**Large transformer models (such as LLMs) are natively dynamical systems, not optimizers.** When a language model
            generates text or takes actions, it traces trajectories through a high-dimensional state space. It has to be
            *trained* to act as an optimizer, and *trained* to align with human intent. It's unclear which
            of these properties will be more robust as we scale.
        

Constraining a generic dynamical system to act as a coherent optimizer is extremely difficult. Often the number of constraints required for monotonic progress toward a goal grows exponentially with the dimensionality of the state space. We shouldn't expect AI to act as coherent optimizers without considerable effort, and this difficulty doesn't automatically decrease with scale.

To probe this directly, we designed a controlled experiment: train transformers to *explicitly*
            emulate an optimizer. We generate training data from steepest descent on a quadratic loss function, then
            train models of varying sizes to predict the next optimization step given the current state (essentially:
            training a "mesa-optimizer").

In these synthetic experiments we found:

Our results are a piece of evidence that future AI failures may look more like **industrial accidents** than
            **coherent pursuit of goals that were not trained for**. (Think: the AI intends to run the nuclear power
            plant, but gets distracted reading French poetry, and there is a meltdown.) However, coherent pursuit of poorly chosen goals that we trained for remains a problem. Specifically:
        

We use a bias-variance decomposition to systematically study how AI error incoherence scales with model intelligence and task complexity. We find that longer sequences of reasoning and actions consistently increase error incoherence, and that smarter models are not consistently more coherent in their errors.

We hope that these findings inform discussions about different AI risk scenarios and guide further research into understanding error incoherence and its mechanistic origins. The question of how current and future AI models fail – systematically or inconsistently – matters both for real-world deployment and how we prioritize safety work.

We thank Andrew Saxe, Brian Cheung, Kit Frasier-Taliente, Igor Shilov, Stewart Slocum, Aidan Ewart, David Duvenaud, and Tom Adamczewski for extremely helpful discussions on topics and results in this paper.
