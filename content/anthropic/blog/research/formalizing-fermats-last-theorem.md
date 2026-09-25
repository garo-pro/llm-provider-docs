Title: Formalizing Fermat's Last Theorem

URL Source: https://www.anthropic.com/research/formalizing-fermats-last-theorem

Markdown Content:
## Subscribe to Anthropic Science

Features on AI-assisted discoveries, practical workflows, and field notes across the sciences.

*We are sharing the first complete computer-checked proof of Fermat’s Last Theorem. Claude worked largely autonomously over 11 days to write the proof in the Lean programming language. Below, we describe how the formalization was done and share some thoughts about what this work could mean for research mathematics.*Around 1637, Pierre de Fermat jotted down a claim in the margin of his copy of Diophantus’s Arithmetica

A decade later, Dutch computer scientist Jan Bergstra proposed “formalizing” Wiles’s proof: converting the mathematical reasoning into a form computers can check automatically. Since then, mathematicians have been developing the methods needed to encode such a complex proof, including a multi-year community effort kicked off in 2024 by Kevin Buzzard at Imperial College London [to complete the formalization](https://lean-lang.org/use-cases/flt/) using the [Lean proof assistant](<https://en.wikipedia.org/wiki/Lean_(proof_assistant)>).

Recently, Tianyi Peng, an Anthropic researcher whose group at Columbia University builds tools for AI formalization, set out to test whether Claude could make progress on formalizing FLT.<sup>[1](https://www.anthropic.com#footnote-1)</sup> The result went further than he expected. In 11 days, working largely autonomously, Claude produced the first end-to-end, computer-checked proof of FLT. Along the way, it wrote 13 million lines of Lean and proved 29,500 intermediate theorems.

We shared the [resulting proof](https://github.com/anthropics/fermats-last-theorem) with Kevin Buzzard, who said:

This extraordinary autoformalization achievement, which Anthropic researchers say only took 11 days, proves Fermat’s Last Theorem with no assumptions other than the axioms of mathematics. Along the way we see autoformalization of algebra, harmonic analysis, geometry and number theory, and we learn that AI autoformalization artefacts are now robust enough to be built upon; the proof is multi-layered.

Automatically formalizing a proof as complex as FLT is a significant step towards a future in which all of mathematics can be readily checked. As AI produces ever more proofs, the ability to easily formalize work can lighten the burden of evaluating new results (a process that can take years). We are hopeful that it will become easier, not harder, to trust the body of knowledge upon which mathematics is built.

Unlike [recent AI-driven](https://www.anthropic.com/research/riemann-zeta) work on the Riemann hypothesis, which produced novel *mathematics*, what’s novel here is the *verification*—checking a mathematical proof as one would check a mathematical computation with a calculator. Proving math theorems requires assembling complex logical chains, and if a single link is broken, everything that follows it might turn out to be false. Understanding a novel result deeply enough to be confident in its correctness can take months, or even years, of work.

Fermat’s Last Theorem is an illustrative example.<sup>[2](https://www.anthropic.com#footnote-2)</sup> Fermat wrote down the theorem’s statement in the margin of a book, alongside a tantalizing note:

I have discovered a truly marvelous proof of this, which this margin is too narrow to contain.

For over 350 years, generations of mathematicians searched for a proof of FLT, marvelous or otherwise. In 1908, a prize of 100,000 German gold marks (the equivalent of 1–2 million dollars today) was announced for anyone who could produce a correct proof, and 621 *incorrect* attempts were produced in the first year alone.

In June 1993, Wiles presented what he believed to be the first correct proof of FLT in a three-day series of lectures. Two months into an intensive verification effort by several mathematicians, a reviewer asked Wiles a question that exposed a critical gap. Wiles spent a year trying to fix it, first alone and then with his former student Richard Taylor. He was on the brink of abandoning the project when he finally realized an approach he’d discarded earlier could fix the proof. 

Wiles published the first correct proof of FLT in May 1995; it relied on modern mathematical techniques that were far beyond what would have been known to Fermat in 1637. Since an elementary proof has not been found after centuries of trying, the mathematical community now believes Fermat’s own original “marvelous proof” [was incorrect](https://mathenchant.wordpress.com/2016/05/16/fermats-last-theorem-the-curious-incident-of-the-boasting-frenchman/).

One way to check a proof’s correctness is to ask a computer to do it. Proof assistants like Lean verify the logic of a proof algorithmically, demonstrating its correctness beyond a doubt. The difficult part for humans is rewriting the proof so Lean can understand it. While a proof written for human readers will skip many obvious steps, Lean needs to see every step, no matter how trivial. Human proofs also build on centuries of published work, while a formalization starts from the tiny fraction of math that’s been formalized already.

For FLT, the formalization process was expected to take years. Just the [blueprint](https://imperialcollegelondon.github.io/FLT/blueprint.pdf) the mathematical community has been using to describe the initial phase of the project runs to 86 pages.

Claude completed the proof in 11 days, producing computer-verifiable proofs of 30,300 theorems along the way (using 29,500 in the final proof). Dozens of Claude agents collaborated to define concepts, prove intermediate theorems, and use those theorems to prove ever harder statements. At 13 million lines of Lean code, Claude’s proof is over 5x the size of Mathlib, the principal community library of mathematical proofs this theorem builds on.[3](https://www.anthropic.com#footnote-3)

Claude’s proof follows [a simplified version of Wiles’s proof from Darmon, Diamond, and Taylor](https://www.math.mcgill.ca/darmon/pub/Articles/Expository/05.DDT/paper.pdf). Mathematical input from humans was limited to occasional high-level instructions from Tianyi: “Jacobian as a scheme sounds high priority,” “push [the] Mazur [theorem] to be done soon.” You can find excerpts of Claude’s thinking [here](https://www-cdn.anthropic.com/9e431dff043da6538d99d6c2d231b670aa3da263.pdf).

```
“THE FLT root reads Proved on the site. Historic moment (modulo re-check).”
“!!! The FLT ROOT 62eb32c0 reads PROVED. R = T closed and cascaded to the root. This is the campaign's goal: e2e FLT on prove2me.”
“🏁🏁🏁The FLT root reads PROVED on prove2me at 02:00:57Z Aug-18 (10:00:57pm ET Aug-17). Historic moment for this campaign.”
```
*Excerpts of Claude’s thinking as it realizes what it has just accomplished.*

A number of Claude’s initial attempts failed: while agents had some early success, they quickly lost track of the project’s state and stopped collaborating effectively. Their failed efforts contributed ~7% of the non-boilerplate lines in the final proof.

The effort succeeded when we switched to using [Prove2Me](https://prove2me.vercel.app/), an open collaborative platform for formalizing mathematics designed by Tianyi Peng and his collaborators at Columbia University. Prove2Me helped by:

With Prove2Me and a Claude Code-based multi-agent harness, a team of agents completed the proof in a little under two weeks, consuming about six billion output tokens from a general-purpose internal research model roughly comparable to Claude Fable 5.1. The finished proof was checked by Lean; it uses just Lean’s three standard axioms, and a [comparator](https://github.com/leanprover/comparator) confirmed that the theorem’s statement matches Mathlib’s own statement of FLT.

The speed with which we were able to produce this proof demonstrates that it is now possible to formalize large swaths of mathematics, which may both catch errors in the common body of mathematical proofs and reduce the burden of refereeing new work. After reviewing Claude’s Lean proof, Kevin Buzzard told us:

If the automatic formalization of FLT is possible now, then we have taken a big step towards automatic formalization of the modern mathematical literature. Such autoformalization techniques will lead to new tools, rooting out errors in the current mathematical corpus and lightening the load of referees. The techniques will also enable us to rigorously check LLM-generated mathematics, which is currently typically an extremely costly human-led process.

Formalization is also a major factor in how humans can gain confidence in AI-generated mathematical results. As AI and AI-assisted mathematicians produce more (purported) proofs than ever before, AI-assisted formalization takes part of the load off human reviewers. We expect it will become common to produce a formalized proof alongside any write-up intended for a human reader. Although we do not think a formalized proof should replace a human-understandable exposition, it may be the only feasible way for the mathematical community to keep up with AI-generated contributions.

Writing Lean also seems to help Claude prove novel results. Many of our recent Claude-authored results have been formalized in parallel with their proofs, and Claude appears to use these partial proofs to independently check its hypotheses much like it writes numerical simulations to check that it’s on the right track.

Formalizing FLT was a token-intensive project, but it is also the largest Lean proof ever constructed. Anthropic researchers did a small experiment using three personal Claude Max plans to formalize applications of the Hardy-Littlewood Circle Method. Collaborating entirely through Prove2Me, the agents jointly completed a formalization of [Vinogradov’s Three Primes Theorem](https://en.wikipedia.org/wiki/Vinogradov%27s_theorem) in just three days. We think with the right scaffold, collaborative formalization of major results with consumer AI subscriptions is achievable.

To this end, [Anthropic](https://www.anthropic.com/news/expanding-support-for-scientists) as well as [other labs](https://openai.com/index/chatgpt-for-academic-researchers/) have recently expanded their support for external researchers—including mathematicians working on pure math and formalization—with free and discounted subscriptions and research credits. We also offer [dedicated grants](https://www.anthropic.com/news/ai-for-science-program) for larger scientific projects, which could include formalizing other major theorems or improving Lean or Mathlib.

With AI rapidly changing what it looks like to do math research, mathematicians—at Anthropic and elsewhere—[are grappling with what that means for their work](https://leidendeclaration.ai/). Formalization, however, is a place where we feel unambiguously good about the role of AI. As formalization becomes a more commonplace tool, we are hopeful that it will help maintain trust in the common body of mathematical knowledge.

Our formalization effort is a small piece of the long history of Fermat’s theorem and the development of formal mathematics. The first full proof from Andrew Wiles together with Richard Taylor was a culmination of more than 300 years of mathematics, integrating ideas from Gerhard Frey, Jean-Pierre Serre, Ken Ribet, Barry Mazur, Robert Langlands, Jerrold Tunnell, Yutaka Taniyama, Goro Shimura, and André Weil, among others. Claude’s proof follows [the exposition by Henri Darmon, Fred Diamond, and Richard Taylor.](https://www.math.mcgill.ca/darmon/pub/Articles/Expository/05.DDT/paper.pdf)

Our proof adapts pieces from the [Imperial College London FLT project](https://github.com/ImperialCollegeLondon/FLT) led by Kevin Buzzard and the [flt-regular project](https://github.com/leanprover-community/flt-regular). Lean and Mathlib are both their own labors of love and have received contributions from hundreds of mathematicians, many working with the [Lean FRO](https://lean-lang.org/fro/). We thank Kevin Buzzard for reviewing the proof and for his comments.

The full proof is available on [GitHub](https://github.com/anthropics/fermats-last-theorem) along with a written walk-through of the proof.

Claude made the open-source models that scientists use to predict and design biomolecules faster and more memory-efficient. Claude optimized more than 30 of these models in just under four weeks, speeding them up roughly 4x on average. It also created a low-memory mode that enables the accurate prediction of biomolecular systems larger than 10,000 tokens (amino acids, nucleotides, and atoms from small molecules and ions) on a single NVIDIA GPU node.

Anthropic’s Frontier Red Team developed new evaluations to measure AI capabilities in tactical intelligence targeting and conventional weapons development.
