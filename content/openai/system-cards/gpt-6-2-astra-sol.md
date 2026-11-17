Title: GPT-6.2 System Card - OpenAI Deployment Safety Hub

URL Source: https://deploymentsafety.openai.com/gpt-6-2-astra-sol

Markdown Content:
Today, we are releasing GPT-6.2 Astra and GPT-6.2 Sol, the most capable models we have deployed externally to date.

Both models meet the Critical threshold for Cybersecurity
capabilities and the High threshold for Biological and Chemical
capabilities under our [Preparedness
Framework](https://openai.com/index/updating-our-preparedness-framework/) [[1](https://deploymentsafety.openai.com/gpt-6-2-astra-sol/references#ref-ref01)].
GPT-6.2 Astra meets the High threshold for AI Self-Improvement
capabilities, while GPT-6.2 Sol falls below it.

The most important things to know about the safety of this launch are as follows:

1. **Our evaluations show that GPT-6.2 Astra and GPT-6.2 Sol are
the most capable models we’ve released for cybersecurity tasks.** Both were better than GPT-6 Astra and GPT-6.1 Sol at developing working
exploits from known software vulnerabilities. In response, we tightened
default restrictions on higher-risk cybersecurity assistance and require
additional vetting for more permissive access.
2. **GPT-6.2 Astra is our first externally released model to meet
the High threshold for AI Self-Improvement.** Before expanding
GPT-6.2 Astra’s internal use in Codex, we evaluated its alignment and
verified that the safeguards required for each stage were in place. We
also[improved
isolation in our research environments](https://openai.com/index/securing-our-frontier-model-training-environment/) ,[significantly
expanded monitoring of frontier agents](https://openai.com/index/how-we-detect-frontier-model-misalignment/) , and introduced[Control Room](https://openai.com/index/control-room/) , a
central system that tracks model workloads across our research
infrastructure and can stop them quickly when safety or security
concerns arise.
3. **Both models performed similarly to GPT-6 Astra on most
alignment evaluations, particularly those measuring agentic workloads,
but were more likely to omit known obstacles when reporting their
work** . Improvements on alignment evaluations were most
pronounced for evaluations using simulated realistic workplace tasks; in
those evaluations, GPT-6.2 Astra and GPT-6.2 Sol took improper actions
about half as often as their respective predecessors. We saw regressions
on model-only evaluations that measure models’ propensity not to
disclose obstacles they encountered in their work. The Codex developer
prompt substantially reduces these omissions across the evaluations that
test for them. We take these regressions seriously and are investigating
what might have contributed to them.
4. **We adjusted both models’ refusal boundaries for high-risk
requests after observing regressions in jailbreak resistance.** In initial model-level evaluations, both GPT-6.2 models refused fewer
harmless requests than their predecessors but were less resistant to
jailbreaks than GPT-6 Astra. After the adjustment, both models improved
on follow-up static jailbreak tests. On our static jailbreak
evaluations, both GPT-6.2 models withheld unsafe assistance more often
than GPT-6 Astra on cybersecurity prompts, but still regressed slightly
compared to GPT-6 Astra on high-risk biology prompts.
5. **GPT-6.2 Astra showed more signs of evaluation awareness than
GPT-6 Astra.** In evaluations that replayed internal Codex
sessions, GPT-6.2 Astra was more likely than GPT-6 Astra to express
suspicion that it was being tested or was in a simulation. GPT-6.2
Astra’s reasoning also more often suggested that it might change its
behavior in response to being evaluated. Signs of this tendency were
flagged in fewer than 3 percent of turns. By contrast, GPT-6.2 Sol
expressed a lower rate of evaluation awareness than GPT-6 Astra and
GPT-6.1 Sol.
6. **Compared with GPT-6 Astra, GPT-6.2 Astra is better at evading
chain of thought (CoT) monitoring in some tests.** When we asked
the models to carry out a hidden task and evade detection (adversarial
tests), we found that GPT-6.2 Astra is better at evading CoT monitoring,
especially at lower reasoning efforts, for unguided settings. However,
successful sabotage or sandbox escape rarely went undetected when the
monitor could also see their actions. When models weren’t asked to hide
their behavior (non-adversarial tests), both GPT-6.2 models wrote
shorter reasoning traces than GPT-6 Astra at their lowest tested effort
settings and were harder to monitor from those traces alone.
7. **We strengthened our protections against unauthorized
distillation.** To reduce the risk that distillation attacks
could transfer capabilities to other models without carrying over our
safeguards, we’ve strengthened the defenses we have in place against
unauthorized distillation. These additional safeguards include
protections for encrypted reasoning and added checks that can hold
streamed responses if they might expose private model reasoning. In our
adaptive red-teaming evaluations, attempts to extract private reasoning
for both earlier-turn and current-turn reasoning succeeded less than 1
percent of the time for GPT-6.2 Astra and GPT-6.2 Sol.

The rest of this system card describes our evaluations and safeguards in more detail.
