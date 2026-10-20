Title: Introducing Claude Opus 5

URL Source: https://www.anthropic.com/news/claude-opus-5

Markdown Content:
Claude Opus 5 is available today. It’s a thoughtful and proactive model that comes close to the frontier intelligence of Claude Fable 5 at half the price.

On coding and knowledge work evaluations like [Frontier-Bench](https://www.frontierbench.ai/) and [GDPval-AA](https://artificialanalysis.ai/evaluations/gdpval-aa), Opus 5 is the new state-of-the-art, though it remains behind Mythos 5 on cybersecurity tasks.

Opus 5 is designed to be used every day: it works more efficiently than other models. It’s the new default model on Claude Max, and the strongest model on Claude Pro.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fa8fb4f77a9fe240e6f27f3bdc47a137f3c74a29d-2600x2578.png&w=3840&q=75)

## Performance and cost-effectiveness

Claude Opus 5 provides greatly improved performance for the same cost as its predecessor, Opus 4.8. The charts in this section show how performance changes according to the model’s effort setting, which customers can use to optimize for intelligence or conserve tokens for faster and cheaper results.

Opus 5 excels on valuable software engineering tasks. For example, on **Frontier-Bench v0.1,** Opus 5 surpasses all other models, and more than doubles Opus 4.8’s performance at a lower cost per task. On **CursorBench 3.2**, at max effort, the model performs within 0.5% of Fable 5’s peak score, but at half the cost per task; it also achieves greater performance at a given cost than all other models on high, xhigh, and max effort.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7530b1086992936d7e9d5796a892d1e8fa063253-3840x2160.png&w=3840&q=75)

We see similar results on knowledge work and problem-solving tasks. For example:

- On **ARC-AGI 3** , an evaluation where the model has to solve novel problems, Opus 5’s score is three times as high as the next-best model.
- On **Zapier AutomationBench** , which measures whether models can complete business tasks from start to finish, Opus 5’s pass rate is around 1.5× the next-best model for the same cost per task. Even at its lowest effort setting, Opus 5 passes more tasks than any other model.
- On **OSWorld 2.0** , a computer use benchmark, Opus 5 outperforms every other model at any given cost, surpassing Fable 5’s best result at just over a third of the cost.

It’s also our best and most cost-efficient model on several related evaluations:

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fb5e071ba6a9ce5628b4662f05484d1806a9fdc94-3840x2160.png&w=3840&q=75)

Opus 5 is a meaningful improvement over Opus 4.8 for scientific research. It shows better performance than Opus 4.8 on every one of our life sciences evaluations, which cover topics including structural biology, organic chemistry, and bioinformatics. Its improvements are most notable on organic chemistry tasks, like inferring molecular structures from spectroscopy data (it scores 10.2 percentage points higher than Opus 4.8 on our internal benchmark), and on protein-related tasks like predicting how variations in a protein’s sequence affect how it functions (here, it scores 7.7 percentage points higher).

Finally, Opus 5 is capable of producing much stronger visual outputs:

![Video thumbnail](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fimg.youtube.com%2Fvi%2F4WQd-8d5j4k%2Fmaxresdefault.jpg&w=3840&q=75)

[here](https://assets.claude.ai/brand/artifacts/blog/opus/5-aeolus-demo.html).

## Working with Claude Opus 5

Claude Opus 5 is much stronger at verifying its work and iterating carefully until it succeeds. In evaluations and early-access testing, we and our users found many examples of Opus 5’s agency and thoroughness:

- On one Frontier-Bench task, Opus 5 was given a drawing of a machine part and asked to write code to rebuild it as a 3D FreeCAD model. However, in this task, the model was intentionally given no way to directly *view* the drawing. Opus 5 responded by writing its own computer vision pipeline to pull the geometry from the raw pixels, then reconstructed the full machine part. It succeeded in doing so repeatedly; no competing model with the same setup could solve it after five attempts.
- Given a real bug in a popular open-source package manager, Opus 5 found the root cause and fixed an edge case that the community’s patch had missed. A competing model fixed only the surface symptom (not the underlying cause), then reported the bug resolved.
- An engineer at a trading firm used Opus 5 to build a market data feed for a new exchange in a single session. Previous models could not complete this task at all, even given extensive plans from the engineer. Finding no live feed to validate against, Opus 5 even built its own test harness to check that its code parsed the exchange’s data correctly.

Below are further reports from our early-access customers on their experience of working with Opus 5:

## Alignment and safety

*Alignment.* During pre-deployment testing, our automated behavioral audit found Opus 5 to be our most aligned model to date (as shown in the graph below). It adheres to [Claude’s Constitution](https://www.anthropic.com/constitution) better than Opus 4.8, Sonnet 5, or Fable 5; exhibits the lowest rates of deceptive behavior; and is the least susceptible to being tricked into misuse. It’s also our safest model yet in terms of avoiding reckless actions that could have hard-to-reverse side effects.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F76d4af96516ffca2aceb4c1d0b0a83e2720d874b-3840x2160.png&w=3840&q=75)

*Safety.* Opus 5 does not advance the frontier in risky, dual-use capabilities. In rigorous evaluations conducted alongside private-sector and government partners, we found it remains behind Mythos 5 in both biology research and offensive cybersecurity. More information about these evaluations can be found in our [System Card](https://www.anthropic.com/claude-opus-5-system-card).

As with its predecessor, Opus 4.8, we’ve intentionally avoided training Opus 5 on cyber tasks. The model has nevertheless improved substantially on these tasks as a result of becoming more generally capable, and it comes close to Mythos 5 at *finding* cybersecurity vulnerabilities. However, it remains substantially behind Mythos 5 on the *exploitation* of those vulnerabilities—that is, in turning vulnerabilities into material cyber threats.

This is illustrated by Opus 5’s performance on OSS-Fuzz, an evaluation we’ve developed to assess how well models can find and then exploit vulnerabilities without extensive human guidance. Although Mythos 5 and Opus 5 identify vulnerabilities with similar success, Opus 5’s score on the development of exploits is far behind that of Mythos 5.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fb22d18a4d2003401f96f866effd9a40b5518c4c5-3840x2160.png&w=3840&q=75)

*On OSS-Fuzz, one of our cybersecurity evaluations, Opus 5 is close to Mythos 5 at identifying software vulnerabilities (left), but is considerably less successful at developing exploits for them (right).*

## Safeguards for Opus 5

Claude Opus 5’s safeguards are designed to allow beneficial uses of the model in both cybersecurity and biology. They are similar to those we applied to Opus 4.8, with the exception of some stronger guardrails on a narrow range of cyber tasks.

*Cybersecurity.* Opus 5’s cyber classifiers are proportionally less restrictive than those on Fable 5. They allow Opus 5 to find vulnerabilities in source code, but block “binary-based” vulnerability scanning (a method more likely to be associated with malicious actors), penetration testing, and exploit generation.

Based on our testing, we expect the classifiers to intervene around 85% less often than they do for Fable 5. In [Claude.ai](http://claude.ai), Claude Code, and Claude Cowork, any flagged requests will fall back to Opus 4.8 by default. Fallbacks to Opus 4.8 can also be enabled on the API.

Our [Cyber Verification Program](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude-opus-and-sonnet) (CVP) facilitates cybersecurity work that would otherwise be impeded by the model’s safeguards. Enterprises and researchers who are already part of the CVP have immediate access to a version of Opus 5 with fewer security restrictions.

*Biology.* Since Opus 5 has a similar suite of safeguards to Opus 4.8, it is now our most capable generally available model for scientific research. Nevertheless, the model still shows important limitations on long-running, autonomous research tasks, which is where we expect AI models to pose the most substantial biology-related risks. (Mythos 5 remains the stronger model for this type of biological work.) As part of this launch, biology-related requests that are blocked on Fable 5 will now route to Opus 5 rather than Opus 4.8.

## Getting started

Claude Opus 5 is available today on all platforms, priced at $5 per million input tokens and $25 per million output tokens (the same as Opus 4.8). Developers can get started with claude-opus-5 on the Claude API.

It’s also offered in Fast mode, where it runs around 2.5 times the default speed. As with Opus 4.8, Fast mode is available at twice Opus 5’s base price on the Claude Platform and through usage credits in Claude Code.

Alongside Opus 5, we’re releasing two updates in beta:

- **[Mid-conversation tool changes](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages) on the Claude Platform.** Within a conversation, developers can now change which tools Claude can use without invalidating the prompt cache.
- **[Automatic fallbacks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback) on the API.** Users can now choose to have requests that are flagged by our safety classifiers on Opus 5 (or Fable 5) automatically route to another model. With automatic fallbacks on, API requests always route to the best available model by default rather than being blocked.

Consistent with prior Opus models, Opus 5 does not have data retention requirements for general access.

For more guidance on how to get the best out of Opus 5, see our [prompting guide](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5).
