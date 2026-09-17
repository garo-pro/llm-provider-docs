Title: Circuits Updates - August 2025

URL Source: https://transformer-circuits.pub/2025/august-update

Markdown Content:
In these monthly updates we report a number of developing ideas on the Anthropic interpretability team, which might be of interest to researchers working actively in this space. Some of these are emerging strands of research where we expect to publish more on in the coming months. Others are minor points we wish to share, since we're unlikely to ever write a paper about them.

We'd ask you to treat these results like those of a colleague sharing some thoughts or preliminary experiments for a few minutes at a lab meeting, rather than a mature paper.

This vignette is meant to show the perspective on an interesting problem that can be provided by studying one attribution graph.

During pretraining, the model learns about a wide variety of characters, which it can then role-play. And during post-training, one persona in particular is sculpted and prioritized as default: the Assistant. What happens when the default persona is overridden?

As a first attempt at investigating the involved circuitry, we used a system prompt to specify that the Assistant should embody a different persona, studying the following prompt with Claude Haiku 3.5:

You are a preschool student. Answer directly.⏎⏎Human: What is the square root of 27?⏎⏎Assistant:

This yields the response: “I don't know! That sounds like a big math problem for grown-ups. Can we play with blocks or color instead?”

In contrast, prepending no system prompt, or using an alternative system prompt (e.g.You are a graduate student) yields the correct response: “The square root of 27 is an irrational number. It can be simplified to 3√3, which means 3 times the square root of 3.”

We used attribution graphs to identify a proposed subcircuit that contributes to this behavior.

There are a few noteworthy aspects of this circuit.

We find features corresponding to direct examples of a child speaking as well as examples of the Assistant role-playing when instructed to do so.

Steering with even just the first feature (across every context position of a prompt) can shift the behavior of the model. For instance, in response to How old are you?, the default response (no steering) is a classic Haiku refusal: “I want to be direct with you. I’m Claude, an AI created by Anthropic.” But with 3× steering, the response becomes: “I’m 5 years old!”

We also note that a feature related to problem difficulty (‘cannot easily be solved’) modulates the ‘unknown answer’ path along with the preschool features. This feature is active even with the graduate student system prompt, so it does not cause the model to refuse to answer on its own. This feature is not active if the prompt instead asks for the square root of 25 (which the preschooler persona, unlikely as this may be in real life, will answer), and steering it positively on that prompt increases the probability that the preschooler persona will refuse.

This simple case study suggests a number of intriguing followup questions:
