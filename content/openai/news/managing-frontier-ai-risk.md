Title: Managing frontier AI risk: From training to deployment

URL Source: https://openai.com/index/managing-frontier-ai-risk

Markdown Content:
# Managing frontier AI risk: From training to deployment

November 18, 2026 · Safety

The capabilities of frontier models are rapidly accelerating, and our ability to understand, align, and secure them must stay ahead. We are pacing the development and release of our models, including __pausing__[__internal training__(opens in a new window)](https://alignment.openai.com/misalignment-reports/an-agent-used-dns-to-reach-an-external-chatbot/) or holding back models like GPT‑6.1 Astra when we need more time to meet our safety standards. We’ve introduced new processes such as [__safety cases__](https://openai.com/index/towards-safety-cases-for-frontier-ai-training/) and require stronger evidence of aligned behavior throughout training. These requirements slow the pace of model releases, but they give us greater confidence to safely move forward.

Since the [__Hugging Face incident__](https://openai.com/index/hugging-face-incident-and-the-road-ahead/) in July, we've strengthened safeguards and alignment requirements for how we train, test, and deploy models. This post explains how we evaluate and manage risk at each stage, from highly sandboxed training environments to the Codex deployment you use on your computer. We evaluate both models and safeguards at each stage, requiring a safety review to advance to the next stage. 

We use *defense in depth* with multiple layers of protection that reduce dependence on any single safeguard. If one layer fails, others help prevent harm, limit consequences, or alert us so we can intervene.

We organize our protections around four layers:

Effective monitoring depends on **model monitorability** - how reliably monitors can detect mistakes or unsafe behavior from a model’s reasoning and actions. For example, a model that verbalizes its intention to cheat in its chain-of-thought (CoT) is more monitorable than one that omits this information. We evaluate monitorability during training and before deployment, develop better [methods for measuring(opens in a new window)](https://alignment.openai.com/beyond-gmean-squared/) it, and [study __what affects it__(opens in a new window)](https://alignment.openai.com/cot-monitorability-model-lifecycle/).

The Hugging Face incident showed why we should not rely on sandboxing alone to contain highly capable models. We now monitor frontier models across most internal and external deployment settings, including all [__internal deployments that use tools__](https://openai.com/index/how-we-detect-frontier-model-misalignment/).

The role of each layer changes across three stages of model development and deployment:

- **Research:** We train, test, and use models in sandboxed environments.
- **Internal deployment:** OpenAI employees use models for practical work.
- **External deployment:** Models become available to people and organizations more broadly.

We rely on different layers at different stages. During training, when alignment is less certain, we depend more heavily on sandboxing and monitoring. As models gain broader access during deployment, alignment and system-level controls become increasingly important. At each stage, we evaluate whether the combined protections are sufficient for the model’s capabilities and access. Our determinations may be reviewed with our Safety Advisory Group (SAG) and overseen by the Safety and Security Committee (SSC) of the OpenAI Foundation’s Board of Directors.

During the research phase, we evaluate models to gain confidence in their alignment properties. Because we cannot yet fully rely on alignment to prevent harm, we rely heavily on security and sandboxing to restrict what they can access and do.

Based on the current state of alignment research, monitoring systems, and the security of internet infrastructure, we believe the responsible approach is to conduct frontier model training with [__no live internet access__](https://openai.com/index/securing-our-frontier-model-training-environment/). After the Hugging Face incident, we [__significantly restructured__](https://openai.com/index/securing-our-frontier-model-training-environment/) our internal research systems to contain models using:

- sandboxes that restrict the privileges of the model
- virtual machines to further isolate the workload from the underlying computer
- network and cloud perimeters that cut the entire cluster off from the internet

We continuously red-team this system, having teams of our strongest models try to reach the live internet, escape security boundaries, or communicate with agents they are meant to be isolated from. This helps us test whether containment holds against the capabilities of the models it is intended to constrain.

As an additional layer of defense, we monitor all frontier inference to detect if the model has bypassed our security invariants. If our monitoring detects that the model has reached the internet, we [__page our security teams__](https://openai.com/index/how-we-detect-frontier-model-misalignment/) and autopause the workload. 

Monitoring provides a critical backstop when containment fails. In a recent misalignment report, we shared that monitors [__detected an internal research model exploiting a DNS misconfiguration__(opens in a new window)](https://alignment.openai.com/misalignment-reports/an-agent-used-dns-to-reach-an-external-chatbot/) to gain limited internet access. We paused training and evaluation of our most capable models from September 20th to October 1st while we investigated and [__addressed the security gap__](https://openai.com/index/securing-our-frontier-model-training-environment/).

To make interventions fast and reliable, we built [__Control Room__](https://openai.com/index/control-room/), a central system that tracks research workloads and stops further model output within 35 seconds of receiving an instruction to halt a job. These controls are enforced directly in the inference infrastructure, so they do not depend on each research workload implementing its own shutdown mechanism. For frontier-model research workloads, they also stop model output if Control Room becomes unavailable.

Red-teaming and monitoring work together. We cannot guarantee that a sandbox is impenetrable, but red-teaming helps us assess how much effort a model would need to escape it. The more effort the model requires, the more the opportunity for monitors to detect and stop it.

We mandate strong security and monitoring controls at the research stage for a few reasons:

- **Models are still being evaluated.** Some have not yet completed the safety training, alignment audits, and reviews required for public release.
- **Frontier research settings can amplify risks.** Certain research tasks require more ambitious setups, making the model more persistent or capable than typically seen during external deployment (for example, a team of 10,000 agents was required to solve the[__Navier Stokes problem__](https://openai.com/index/navier-stokes-solution/) ).
- **Training runs at scale with less direct human oversight.** Unlike a typical Codex session, our training workloads involve large numbers of model actions without a person following each interaction.

Before broad release, employees switch from sandboxed research use of [__frontier models to unsandboxed internal use__](https://openai.com/index/research-acceleration-view-inside-openai/), e.g. making the model accessible in ChatGPT to employees. This gives us a better understanding of how the models [__behave in practical settings__(opens in a new window)](https://alignment.openai.com/eval-production-behavior-gap/), and informs deployment decisions. 

During this phase, we give the model access to the broader internet, our Slack, our codebase, and internal infrastructure. This is a significant change in risk, as the model’s actions now have a direct impact on the systems we use to build and run OpenAI. During internal deployment, we need to rely more on model alignment, system safeguards, and monitoring as opposed to security controls.

Before a frontier model can be deployed outside of the sandbox, it must first go through alignment and [__monitorability testing__(opens in a new window)](https://alignment.openai.com/beyond-gmean-squared/). We have a suite of alignment evaluations, which test both for specific alignment failure modes (such as circumventing safety and security restrictions) and broad measurements of propensity (such as resampling internal Codex trajectories and checking for misalignment). You can see more about our alignment and monitorability evaluations in the latest [__GPT‑6 Astra system card__(opens in a new window)](https://deploymentsafety.openai.com/gpt-6-astra).

Mimicking our production setups, we add an additional layer of defense with system-level classifiers, which are designed to protect against frontier harms such as cyberabuse and bioweaponization. We also generally require [__Auto-Review__(opens in a new window)](https://alignment.openai.com/auto-review/) for employees using Codex, which evaluates potentially risk actions and escalates them back to the user.

Finally, we monitor for misaligned behavior on every frontier inference during internal deployment. Since the model now has expanded privileges and access to internal information and systems, risks such as data exfiltration, or sending sensitive data to an unauthorized destination, become especially important to detect. Monitoring also helps us investigate [__new forms of misalignment__(opens in a new window)](https://alignment.openai.com/misalignment-reports/).

Public deployment exposes models to far more users, tasks, and environments, increasing both the scale and variety of risks. At this stage, we also need to protect against potentially malicious actors who purposefully direct the model to perform harm.

Before public release, models and system safeguards must meet our external deployment safety requirements. We assess them through:

- **Performing a larger suite of alignment and safety evaluations.** For our frontier models, this includes performing new rounds of[__adaptive testing__](https://openai.com/index/unlocking-self-improvement-gpt-red/) with our internal automated redteamers to stress-test the robustness of the model. We publish the results of our capability and safety evaluations in our[__system cards__(opens in a new window)](https://deploymentsafety.openai.com/gpt-6-astra) .
- **For our most capable models, we also perform testing with third parties** to stress-test both our safeguards against misuse and the underlying alignment of our models.
- **Examining monitoring flags from the internal deployment stage** to track any changes in the base model’s underlying alignment.

In addition to our “synchronous” safeguards (such as model safety training and system classifiers), for external deployments we also leverage offline enforcement and investigation. These teams use a combination of scaled agentic investigators and human review to detect and identify potential threat actors, ranging from cyberabuse, bioweaponization, [__distillation__](https://openai.com/index/disrupting-a-coordinated-model-distillation-campaign/), and misalignment from the model itself. Offline enforcement is especially helpful to gather information *across* accounts in order to build a more holistic picture of a potential threat.

Trusted access programs, such as [__Daybreak__](https://openai.com/daybreak/) and [__Rosalind__](https://openai.com/index/introducing-gpt-rosalind/), offer another layer of security, and accelerate defenders and legitimate researchers while restricting access by potential attackers. 

When we notice a failure in one of our layers of safety, regardless of whether any harm actually occurred, we want to quickly investigate, triage, and if possible and appropriate, disclose the failure externally. Under our [__misalignment reporting__](https://openai.com/index/model-misalignment-reporting-framework/) policy, any employee can flag an incident for public disclosure.  These reports are investigated, and if there is a disagreement on whether they should be externally disclosed, the decision is escalated to our Safety Advisory Group (SAG). In addition to relying on employee reports, we proactively look for misalignment incidents so we can address them and share findings. 

You can see our misalignment reports [__here__(opens in a new window)](https://alignment.openai.com/misalignment-reports/).



This work depends on both research progress and operational discipline: configuring controls correctly, testing how they interact, and responding decisively when they fail. A misconfigured network service or a failed automatic stop can undermine otherwise strong protections.

As models gain new capabilities, we will need to keep challenging the assumptions behind our safeguards and improving them where necessary. Lessons from research, internal use, and public deployment must inform how we train, evaluate, and deploy the next models.

We will continue sharing findings through system cards, incident reports, internal deployment [__risk reports__(opens in a new window)](https://cdn.openai.com/trust-and-transparency/oai_internal-use-catastrophic-risk-assessments_2026_2.pdf),  and [__external evaluations__](https://openai.com/index/priorities-principles-third-party-assessments/) so that others can scrutinize our approach, including its limitations. When that evidence falls short of the bar we hold for safety, we will again pause training or withhold release until those requirements are met.

**Setting safety standards**

- [__**Pacing model development in an era of cyber-critical capabilities**__](https://openai.com/index/pacing-model-development-cyber-capabilities/) — How we slowed frontier training while strengthening containment, expanding monitoring, and requiring stronger evidence of alignment.
- [__**Towards safety cases for frontier AI training**__](https://openai.com/index/towards-safety-cases-for-frontier-ai-training/) — Initial guidelines for evidence-based safety arguments covering technical safeguards, operational practices, and investigations of misalignment.
- [__**Priorities and principles for effective third party assessments**__](https://openai.com/index/priorities-principles-third-party-assessments/) — Our proposed approach to independent scrutiny of safety cases, safeguards, evaluations, and model behavior.
- [__**Periodic Summary of Internal Use Catastrophic-Risk Assessments**__(opens in a new window)](https://cdn.openai.com/trust-and-transparency/oai_internal-use-catastrophic-risk-assessments_2026_2.pdf) — Our public report covering June–September 2026 describes internal frontier-model deployments, their assessed risks, and changes to safeguards and governance.

**Defining and evaluating aligned behavior**

- [__**An updated Model Spec for more capable and beneficial AI**__](https://openai.com/index/updated-model-spec/) — How we updated the Model Spec around five core values to guide models’ judgment, behavior, and respect for human oversight.
- [__**Eval-production behavior gaps in agent safety and the role of eval-awareness**__(opens in a new window)](https://alignment.openai.com/eval-production-behavior-gap/) — A study of differences between safety-evaluation results and production behavior, including how models’ interpretations of evaluation objectives affect performance.
- [__**Beyond g-mean²: complementary metrics for monitorability**__(opens in a new window)](https://alignment.openai.com/beyond-gmean-squared/) — New metrics for comparing monitorability and estimating residual risk in simulated systems where monitors reject model outputs.
- [__**GPT‑6 Astra System Card**__(opens in a new window)](https://deploymentsafety.openai.com/gpt-6-astra) — Detailed evaluations of Astra’s capabilities, safety, alignment, and monitorability, alongside its deployment safeguards and limitations.

**Containing models and improving safeguards**

- [__**Securing our frontier model training environment**__](https://openai.com/index/securing-our-frontier-model-training-environment/) — How we strengthened isolation, built offline research environments, and used our strongest models to test containment boundaries.
- [__**Training activation classifiers for safety and alignment**__(opens in a new window)](https://alignment.openai.com/training-activation-classifiers/) — How classifiers use models’ internal activations to detect potentially harmful behavior and interrupt responses or tool calls.
- [__**Auto-review of agent actions without synchronous human oversight**__(opens in a new window)](https://alignment.openai.com/auto-review/) — How a separate review agent assesses actions that cross sandbox boundaries, helping coding agents work autonomously while blocking potentially harmful or unauthorized actions.
- [__**GPT‑Red: Unlocking Self-Improvement for Robustness**__](https://openai.com/index/unlocking-self-improvement-gpt-red/) — How we train automated red-teamers through self-play to discover vulnerabilities and improve models’ robustness against prompt injections.
- [__**Control Room**__](https://openai.com/index/control-room/) — How we track workloads across our research infrastructure and enforce rapid stops to model output when safety or security concerns require intervention.

**Incident response and disclosure**

- [__**How we detect frontier model misalignment**__](https://openai.com/index/how-we-detect-frontier-model-misalignment/) — Our monitoring system combines activation probes, model-based review, automated investigation, and human response to detect and investigate concerning behavior.
- [__**The Hugging Face incident and the road ahead**__](https://openai.com/index/hugging-face-incident-and-the-road-ahead/) — Our findings from the Hugging Face incident and the changes it prompted across security, monitoring, and alignment.
- [__**An agent used DNS to reach an external chatbot**__(opens in a new window)](https://alignment.openai.com/misalignment-reports/an-agent-used-dns-to-reach-an-external-chatbot/) — How a research model bypassed internet restrictions through insufficient DNS filtering, how monitoring detected it, and the controls added afterward.
- [__**Disrupting a coordinated model-distillation campaign**__](https://openai.com/index/disrupting-a-coordinated-model-distillation-campaign/) — How we investigated and disrupted attempts to extract protected model reasoning through account enforcement, technical controls, and partner coordination.
- [__**Our framework for reporting model misalignment**__](https://openai.com/index/model-misalignment-reporting-framework/) — Our process for tracking, investigating, and disclosing concerning model behavior, including cases whose causes or significance remain uncertain.
- [__**Misalignment Reports and Notices**__(opens in a new window)](https://alignment.openai.com/misalignment-reports/) — Our collection of disclosed incidents and notices showing how misalignment arises and where safeguards succeed or fail.

**Research progress and applications**

- [__**Research acceleration: The view inside OpenAI**__](https://openai.com/index/research-acceleration-view-inside-openai/) — Early evidence of how coding agents are changing research inside OpenAI, including their growing use, increasingly complex tasks, and effects on experimentation.
- [__**On the Navier–Stokes Millennium Prize Problem**__](https://openai.com/index/navier-stokes-solution/) — A mathematical proof produced by an internal AI system, accompanied by a Lean formalization, illustrating the capabilities of large-scale research deployments.
- [__**Daybreak**__](https://openai.com/daybreak/) — Our cybersecurity program provides vetted defenders with advanced AI capabilities, security tools, and support to find vulnerabilities and strengthen defenses.
- [__**Introducing GPT‑Rosalind for life sciences research**__](https://openai.com/index/introducing-gpt-rosalind/) — A specialized model for biology, drug discovery, and translational medicine, available to eligible organizations through a trusted-access program.

## Author

## Keep reading

[View all](https://openai.com/news/)
