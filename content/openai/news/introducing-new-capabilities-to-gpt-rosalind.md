Title: Introducing new capabilities to GPT-Rosalind

URL Source: https://openai.com/index/introducing-new-capabilities-to-gpt-rosalind

Markdown Content:
# Introducing new capabilities to GPT‑Rosalind

Bringing greater intelligence grounded in real scientific workflows for the life sciences industry.

**Update on September 11, 2026***:* *GPT‑Rosalind**, our specialized model for life sciences research, is coming out of research preview and is now available globally to eligible organizations through our trusted-access program. All eligible organizations will continue to get access to the latest Rosalind models as they’re released. Published pricing will take effect on* *October 5, 2026**. Review* [__*GPT‑Rosalind pricing*__(opens in a new window)](https://developers.openai.com/api/docs/pricing?latest-pricing=standard).

We’re introducing a new model update to our [GPT‑Rosalind](https://openai.com/rosalind/) series purpose-built for life sciences research at enterprise scale. It combines GPT‑5.5’s agentic coding and tool-use capabilities with stronger model intelligence in core drug-discovery domains such as medicinal chemistry and genomics, while advancing performance across broader life sciences analysis, design, and experimental workflows.

Progress in life sciences depends on synthesizing data and evidence across scales and modalities: molecules, genes, pathways, and living systems. In our evaluations, the updated GPT‑Rosalind shows broad performance gains on research tasks from biology experts, complex medicinal chemistry queries, quantitative biology, and wet lab troubleshooting.

GPT‑Rosalind is available to eligible organizations globally through our trusted-access deployment structure.

In order to measure and continuously improve the real-world impact of GPT‑Rosalind, we designed LifeSciBench, an externally expert-judged benchmark focused on foundational aspects in life sciences research. Unlike existing benchmarks that evaluate a single component of model performance or biological domain in isolation, LifeSciBench takes an end-to-end view of scientifically valuable work by drawing tasks from six workflow areas central to life sciences research: evidence handling, analysis, design and optimization, scientific reasoning, validation and operations, and translation and communication. We use this benchmark to align progress with the needs and realities of life sciences research.

GPT‑Rosalind leads performance across scientifically-valuable tasks identified by industry and academic experts.

GPT‑Rosalind achieves industry-leading performance in medicinal chemistry, a field focused on turning molecules into useful drugs. We designed MedChemBench to reflect realistic medicinal chemistry workflows, evaluating multimodal chemical structure understanding; structure-activity relationship (SAR); prediction of drug potency, toxicity, and absorption, distribution, metabolism, excretion (ADME); multiparameter lead-optimization decision-making; and retrosynthesis. GPT‑Rosalind out-performs GPT‑5.5 at 27.5% vs. 25.1% on MedChemBench, while using 7.2% fewer tokens.

GPT‑Rosalind shows better multimodal synthesis and mechanistic reasoning in medicinal chemistry.

On GeneBench, our agentic evaluation on long horizon, end-to-end analysis in genomics and quantitative biology, GPT‑Rosalind uses 31% fewer tokens than GPT‑5.5 while achieving a higher accuracy of 21.6% vs. 20.4%. GeneBench assesses agentic performance on long-horizon quantitative tasks: based on realistic scientific data, can an agent plan valid analysis, QC, modeling, and corrections to arrive at decision-relative answers? Included problems span a variety of domains, including functional genomics, spatial transcriptomics, proteomics, epigenomics, and applied genetics.

GPT‑Rosalind uses 31% fewer tokens than GPT‑5.5 while improving accuracy.

We introduce a new evaluation to test GPT‑Rosalind’s ability to help scientists conducting lab work in the real world. LabWorkBench tests the model's ability to link perturbations to experimental outcomes in real wet lab protocols used by scientists, for the purposes ranging from troubleshooting to optimization. The data used by LabWorkBench are proprietary and thus uncontaminated. GPT‑Rosalind scores 63.2% vs. GPT‑5.5 at 55.8%, while using 5.3% fewer tokens.

On real wet lab protocol assistance, GPT‑Rosalind shows significant gains over GPT‑5.5 while improving token efficiency.

We built the [Life Sciences Research(opens in a new window)](https://github.com/openai/plugins/tree/main/plugins/life-science-research) and [Life Sciences NGS Analysis(opens in a new window)](https://github.com/openai/plugins/tree/main/plugins/ngs-analysis) plugins to extend the increased intelligence of [GPT‑Rosalind](https://openai.com/rosalind/) with a practical execution layer for repeatable scientific workflows. Together, these plugins bring sourced evidence retrieval, biological interpretation, and bioinformatics execution into the same workspace, helping researchers connect external evidence with internal omics analyses while preserving artifacts and provenance. All users can now access both plugins through Codex. Qualified GPT‑Rosalind enterprise users can additionally use GPT‑Rosalind to power these plugins.

To better leverage Codex as a dynamic workbench for scientists, we added interactive viewers for biologically native file types. The initial set of sequence, alignment, and structure viewers are designed to keep scientists close to the evidence as GPT‑Rosalind reasons across a workflow and directly answer follow-up questions using the active viewer in-context.

The demo above shows these capabilities in action, orchestrated by GPT‑Rosalind. We follow a scientist investigating a liquid tumor biopsy to identify mutations and other molecular changes that could inform treatment. The Life Sciences NGS Analysis plugin turns a review of processed ctDNA records into an interactive notebook, surfacing recurring alterations, low-frequency calls, and sample trajectories that focus the investigation on KRAS G12C. From there, the Life Sciences Research plugin adds sourced target, inhibitor, and resistance context, while the native sequence, alignment, and structure viewers allow the scientist to inspect mutant residue 12, its conservation across the RAS family, and the inhibitor-bound pocket directly. The workflow concludes by translating that evidence into concrete follow-up options, with each step and artifact available for expert review.

![A computer screen shows a workspace instructing the use of an NGS Analysis plugin to explore ctDNA mutation data. The screen includes several bar charts labeled "Top detailed histologies" and "Top altered genes by mutated cfDNA samples," displaying data on cancer types and gene alterations. Text describes the dataset, key findings, and analysis parameters.](https://images.ctfassets.net/kftzwdyauwt9/1dnSr1zzUcQuhNIOhqRnWC/fb69d717b0141f89edb5f5a48fa66692/Rosalind-55-Layout-Article-Plugin-Figure-NGS-Analysis.png?w=3840&q=90&fm=webp)

Life Sciences NGS Analysis plugin

scRNA-seq QC & Annotation

![Screenshot of a split-screen bioinformatics workflow. The left panel shows an AI assistant summarizing a completed single-cell RNA sequencing (scRNA-seq) quality-control analysis, including generated files, QC metrics, UMAP visualizations, and cell-type annotations. The right panel displays an “scRNA QC Review” report with histograms for total counts, detected genes, and mitochondrial percentage, alongside bar charts showing QC pass/fail counts and filtered cell populations. The interface is displayed against a blue-and-green gradient background.](https://images.ctfassets.net/kftzwdyauwt9/3kELBgyLQu8SKx7vfwyv9g/6703dd86fd1a6f0e1307dea674aee42e/Rosalind5.5_scRNA_UseCase.png?w=3840&q=90&fm=webp)

Turn a 10x-style matrix bundle into QC-filtered single-cell artifacts, annotations, and UMAPs you can inspect and revise in Codex. The Life Sciences NGS Analysis plugin routes the request to scrna-seq-qc, chooses QC thresholds from the data, preserves provenance around filtering and annotation, and surfaces blockers such as missing doublet-detection dependencies.

Bulk RNA-seq FASTQ QC

![Split-screen view of an RNA-seq workflow: an AI assistant summarizes completed bulk RNA-seq quality-control results on the left, while an interactive MultiQC report with sequencing statistics and Salmon metrics is displayed on the right.](https://images.ctfassets.net/kftzwdyauwt9/2fAmR7xZ8JhGsUz4DMA5ir/695bc7f91e6b7696afa6316dd60c74eb/Rosalind5.5_bulkRNA_UseCase.png?w=3840&q=90&fm=webp)

Turn a bulk RNA-seq sample sheet, FASTQ bundle, and reference files into a QC-reviewed counts bundle you can inspect and reuse in Codex. The Life Sciences NGS Analysis plugin routes the request, validates the inputs, and returns an auditable run envelope with MultiQC, Salmon matrices, provenance, and explicit caveats.

We are expanding access to the [GPT‑Rosalind](https://openai.com/rosalind/) series to eligible organizations globally. GPT‑Rosalind will be available through our trusted-access deployment structure for organizations that are conducting legitimate scientific research with clear public benefit, have strong governance and safety oversight, and controlled access with enterprise-grade security.

As part of this global expansion, we’re excited to help support Novo Nordisk’s mission of bringing innovative treatment options to patients faster by helping scale their medical research with GPT‑Rosalind. Novo Nordisk is leveraging frontier AI capabilities to help researchers analyze complex datasets, uncover useful patterns, and test hypotheses more quickly. GPT‑Rosalind’s stronger biological understanding will help teams connect evidence across literature, genomics, transcriptomics, sequence, structure, and experimental results, making it easier to move from data to clearer research decisions.

“Life sciences research is complex, data-rich, and interdisciplinary. To deliver meaningful value for researchers, advanced AI models must be grounded in trusted scientific data, connected to validated tools, and integrated into the real-world workflows researchers use every day. We’re pleased with our partnership with OpenAI and the opportunity to explore how GPT‑Rosalind can support more rigorous, practical approaches to drug discovery.”

Mishal Patel, Group Vice President, AI & Digital Innovation, R&D - Novo Nordisk

We are also now offering an OpenAI managed workspace for qualified organizations without an Enterprise account.

The updated GPT‑Rosalind is the next step in our broader commitment to building AI systems that can help accelerate scientific discovery while ensuring that advanced biological capabilities are deployed with appropriate safeguards. We will continue improving the model’s biological reasoning, expanding support for tool-heavy and long-horizon research workflows, and working with qualified organizations across regions to evaluate real-world impact.

This also means applying life sciences AI to high-impact public-benefit work, from drug discovery and translational medicine to public health, preparedness, and biodefense. Through [Rosalind Biodefense](https://openai.com/index/strengthening-societal-resilience-with-rosalind-biodefense/) and our trusted-access deployment model, we aim to put frontier biological capabilities in the hands of the researchers, institutions, and defenders working to improve human health and strengthen societal resilience.

We will continue building [GPT‑Rosalind](https://openai.com/rosalind/) to become a more capable partner across the full life cycle of scientific research, helping scientists move more quickly from the right questions to clearer evidence, better experiments, and ultimately new treatments for patients.

## Keep reading

[View all](https://openai.com/news/)
