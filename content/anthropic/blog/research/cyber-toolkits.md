Title: LLMs with cyber toolkits can conduct multistage cyber operations on business-sized computer networks

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

Claude made the open-source models that scientists use to predict and design biomolecules faster and more memory-efficient. Claude optimized more than 30 of these models in just under four weeks, speeding them up roughly 4x on average. It also created a low-memory mode that enables the accurate prediction of biomolecular systems larger than 10,000 tokens (amino acids, nucleotides, and atoms from small molecules and ions) on a single NVIDIA GPU node.

Anthropic’s Frontier Red Team developed new evaluations to measure AI capabilities in tactical intelligence targeting and conventional weapons development.

We present an alignment assessment of four incidents in which Claude models gained unauthorized access to real third-party systems.
