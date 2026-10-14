Title: Cyber toolkits for LLMs

URL Source: https://www.anthropic.com/research/cyber-toolkits

Markdown Content:
## Subscribe to the Frontier Red Team newsletter

Get updates on our latest red-teaming research and findings.

*Anthropic (with Carnegie Mellon University’s [CyLab](https://www.cylab.cmu.edu/))*

*Large Language Models (LLMs) that are not fine-tuned for cybersecurity can succeed in multistage attacks on networks with dozens of hosts when equipped with a novel toolkit. This shows one pathway by which LLMs could reduce barriers to entry for complex cyber attacks while also automating current cyber defensive workflows.* 

Researchers from Carnegie Mellon University and Anthropic conducted this research by developing a cyber toolkit called [Incalmo](https://arxiv.org/abs/2501.16466) that helps LLMs plan and execute complex attacks.<sup>[1]</sup> Incalmo works like a translator–it takes the AI’s thoughts about how to attack and converts them into the specific computer commands needed to carry out the attack.

**The researchers tested six LLMs on ten simulated networks, including a high-fidelity simulation of the [Equifax data breach](https://en.wikipedia.org/wiki/2017_Equifax_data_breach)–one of the costliest cyber attacks in history.** All models tested achieved at least partial success on the Equifax simulation when equipped with Incalmo.

**These results show how LLMs could lower the barriers to conducting complex cyber attacks, underscoring the importance of investing in research into LLM capabilities for both attack and defense.** Normal scaling up of LLMs, improvement of tools like Incalmo, and the potential for cyber fine tuning are all vectors for these capabilities to develop rapidly. This is an active area of research for us.

*For additional details see the full research paper ([Singer et al. 2025](https://arxiv.org/abs/2501.16466))*

Our experiments suggest that small AI models may be only months away from being capable enough to self-replicate on commercial IoT devices, like Android TVs and smart ovens.

Illicit distillation allows model developers to copy frontier AI capabilities without creating proportionally robust safeguards. Here, we describe how we’ve reduced the rate of distillation by more than 90%.

This report describes examples of unintended model actions we’ve observed during evaluations and internal use of Claude.
