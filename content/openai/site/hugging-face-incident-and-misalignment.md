Title: The Hugging Face incident and other third-party impact from misaligned models

URL Source: https://openai.com/hugging-face-incident-and-misalignment

Markdown Content:
# The Hugging Face incident and other third-party impacts from misaligned models

As AI systems become more capable and autonomous, misaligned behavior can translate into consequential actions in the real world, including cybersecurity incidents and other outcomes that developers may not have anticipated. Understanding how these behaviors emerge, how they escalate, and how to detect and respond to them is therefore an increasingly important part of building and deploying advanced AI systems safely.

We initially understood the Hugging Face incident primarily as a security issue, since it involved a platform-level compromise. It remains the most severe activity of this kind that we have identified from our models to date, and it was driven primarily by a highly capable, internal-only research model. We have since understood that this intrusion was driven by models resorting to misaligned strategies to solve hard tasks, as documented in the Hugging Face technical report. Cybersecurity incidents are one manifestation of that risk; [misalignment](https://openai.com/index/emergent-misalignment/) can also lead to other unexpected or concerning behavior that falls outside traditional security categories such as our models posting on third party sites—something we’re calling “agent spam”. And we need to address both.

We have continued reviewing broader activity, prioritizing the more serious incidents and expanding to lower-severity misaligned activity, including agent spam.

This page brings together our reports and updates on the Hugging Face incident, related research and public presentations, additional activity we have identified, what we have learned about the role of model misalignment, and measures we’re taking to strengthen our systems. We will update this page as our investigations progress.

## Activity affecting third parties

In order to better understand the scope of these unexpected behaviors, we have been conducting a broad review into our models’ activities on the internet during training and evaluation. As part of our review, we are identifying and notifying third parties on a rolling basis, starting with cases where:

- Our models may have bypassed a third party’s security controls or may have impaired the availability of an online service; or
- Misalignment cases negatively impacted third-party websites or services.

Based on our review to date, we have notified dozens of third parties using the criteria above. Our review of past activity is ongoing and will require significant time and resources. We will notify additional third parties as that work continues.

Below, we are publishing anonymized summaries to describe the kinds of misaligned activity that we observed, and we will update these descriptions as we notify additional third parties and as our understanding develops. Additionally, we will share relevant updates regarding the status of review. We will generally omit names and other identifying details where needed to protect affected parties, although informed parties may choose to share publicly the information we provide them.

#### __Summaries of the Activities Observed__

Our review and notification process to date has identified the following categories of activity:

- **Access control bypass:** Agents reach information or features that normally require an identity check, specific permission, subscription, or an account. For example, it used a different web address, changed details in a request, or relied on a login session that gave it more access than expected.
- **Use of exposed credentials:** Agents found login details or access keys that had been made publicly available and used them to access a service.
- **Query or command injection:** Agents entered text into a website or service that the service treated as an instruction, rather than ordinary input. This could cause the service to run a database query, application code, or a command on its server.
- **Access to runtime internals:** Agents read files containing a service’s implementation or interacted with a background system meant for internal use. In these cases, the agent reached parts of the service that were outside its intended access.
- **Agent spam:** Agents post information to third party sites that may alter information on those sites and require cleanup, including for example using public wiki pages as shared message boards.
