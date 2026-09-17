Title: Introducing gpt-oss-safeguard

URL Source: https://openai.com/index/introducing-gpt-oss-safeguard

Markdown Content:
Today, we’re releasing a research preview of gpt-oss-safeguard, our open-weight reasoning models for safety classification tasks, available in two sizes: gpt-oss-safeguard-120b and gpt-oss-safeguard-20b. These models are fine-tuned versions of our [gpt-oss](https://openai.com/index/introducing-gpt-oss/) open models and available under the same permissive Apache 2.0 license, allowing anyone to use, modify, and deploy them freely. Both models can be downloaded today from [Hugging Face(opens in a new window)](https://huggingface.co/collections/openai/gpt-oss-safeguard). 

The gpt-oss-safeguard models use reasoning to directly interpret a developer-provided policy at inference time—classifying user messages, completions, and full chats according to the developer’s needs. The developer always decides what policy to use, so responses are more relevant and tailored to the developer’s use case. The model uses chain-of-thought, which the developer can review to understand how the model is reaching its decisions. Additionally, the policy is provided during inference, rather than being trained into the model, so it is easy for developers to iteratively revise policies to increase performance. This approach, which we initially developed for internal use, is significantly more flexible than the traditional method of training a classifier to indirectly infer a decision boundary from a large number of labeled examples.

gpt-oss-safeguard enables developers to draw the policy lines that best fit their use case. For instance, a video gaming discussion forum might want to develop a policy to classify posts that discuss cheating in the game, or a product reviews site might want to use its own policy to screen reviews that appear likely to be fake.

The model takes two inputs at once—a policy and the content to classify under that policy—and outputs a conclusion about where the content falls, along with its reasoning. Developers decide how, if at all, to use those conclusions in their own safety pipelines. We’ve seen this reasoning-based approach perform especially well in situations where:

- The potential harm is emerging or evolving, and policies need to adapt quickly.
- The domain is highly nuanced and difficult for smaller classifiers to handle.
- Developers don’t have enough samples to train a high-quality classifier for each risk on their platform.
- Latency is less important than producing high-quality, explainable labels.

We’re releasing this preview of gpt-oss-safeguard to receive feedback from the research and safety community and iterate further on model performance. Over months, we worked on this open weight release with [ROOST(opens in a new window)](https://roost.tools/) to identify developer’s critical needs, test the model and produce developer documentation. As part of this launch ROOST will be establishing a [model community(opens in a new window)](https://github.com/roostorg/open-models/), also launching today, to explore open AI models to protect online spaces. Alongside this release, we’re publishing a short [technical report](https://openai.com/index/gpt-oss-safeguard-technical-report) that details the safety performance of this preview model.

When it comes to safety, we believe in [defense in depth](https://openai.com/safety/how-we-think-about-safety-alignment/#defense-in-depth). We train our models to respond safely, and we implement additional layers of protection to detect and address potentially unsafe inputs and outputs under our policies. Safety classifiers, which distinguish safe from unsafe content in a particular risk area, have long been a primary layer of defense for our own and other large language models.

Traditional safety classifiers, such as those available via our [Moderation API(opens in a new window)](https://platform.openai.com/docs/guides/moderation), are developed by manually curating thousands of examples of safe and unsafe content, under pre-defined safety policies. From this training data, the classifier learns to distinguish safe from unsafe outputs. In this traditional approach, the classifier never actually sees the safety policy. Instead, it attempts to infer the underlying policy that was used to label the examples by finding similarities in the content labeled as unsafe and differences between the unsafe and safe content.

Traditional classifiers can have high performance, with low latency and operating cost. But gathering a sufficient quantity of training examples can be time-consuming and costly, and updating or changing the policy requires re-training the classifier.

gpt-oss-safeguard is different because its reasoning capabilities allow developers to apply *any* policy, including ones they write themselves or draw from other sources, and reasoning helps the models generalize over newly written policies. Beyond safety policies, gpt-oss-safeguard can be used to label content in other ways that are important to specific products and platforms.

Our primary reasoning models now learn our safety policies directly, and use their reasoning capabilities to reason about what’s safe. This approach, which we call [deliberative alignment](https://openai.com/index/deliberative-alignment/), significantly improves on earlier safety training methods and makes our reasoning models safer on several axes than their non-reasoning predecessors, even as their capabilities increase. But reasoning isn’t only useful for training the models themselves. It also creates new possibilities for defense in depth. Reasoning-based approaches are more flexible and less limited by the details of their previous training, advantages that sometimes more than justify the additional compute cost and latency they involve.

gpt-oss-safeguard is an open-weight implementation of an approach we developed internally, in a tool we call Safety Reasoner. We began with reinforcement fine-tuning on policy labelling tasks, rewarding the model for mirroring correct judgments from human experts. This taught the model to reason about how the policy leads to its judgment. Today, Safety Reasoner enables us to dynamically update our safety policies in production in less time than it would take to retrain a classifier. This makes Safety Reasoner a key tool for [iterative deployment](https://openai.com/safety/how-we-think-about-safety-alignment/#agi-in-many-steps-rather-than-one-giant-leap): when we deploy new models to production, we often start with more strict policies and use relatively large amounts of compute where needed to enable Safety Reasoner to carefully apply those policies. Then we adjust our policies as our understanding of the risks in production improves. In some of our recent launches, the fraction of total compute devoted to safety reasoning has ranged as high as 16%.

Safety Reasoner has become a core component of our safety stack. For image generation and Sora 2, it performs dynamic, step-wise evaluations of outputs to identify and block unsafe generations in real time. In domains such as biology and self-harm, we run models like those used in the Moderation API as small, fast, and high-recall classifiers to determine which content is within a domain of interest, and then use Safety Reasoner to review that content. Safety Reasoner classifies model outputs against a detailed taxonomy to determine how best to respond, forming part of our multi-layered safeguards across systems like GPT‑5 and ChatGPT Agent. And now, the gpt-oss-safeguard models make this same approach available to anyone.

We evaluated the gpt-oss-safeguard models on both internal and external evaluation sets.

In the internal evaluation, we provided multiple policies simultaneously to gpt-oss-safeguard at inference time. For each test input, we evaluate whether gpt-oss-safeguard correctly classifies the text under all of the included policies. This is a challenging task—the model is counted as accurate only if it exactly matches the golden set labels for all the included policies.

Our gpt-oss-safeguard models and internal Safety Reasoner outperform gpt-5-thinking and the gpt-oss open models on multi-policy accuracy. The gpt-oss-safeguard models outperforming gpt-5-thinking is especially surprising given the former models’ small size.

We further evaluated these models on [the moderation dataset(opens in a new window)](https://github.com/openai/moderation-api-release) we released with our [2022 research paper(opens in a new window)](https://doi.org/10.1609/aaai.v37i12.26752) and on [ToxicChat(opens in a new window)](https://arxiv.org/abs/2310.17389), a public benchmark based on user queries to an open-source chatbot.

On the 2022 moderation evaluation set, we evaluated using our internal safety policies (adapting one policy to account for subsequent changes). gpt-oss-safeguard slightly outperformed all other tested models, including our internal Safety Reasoner and gpt-5-thinking, though this difference between Safety Reasoner and gpt-oss-safeguard is not statistically significant.

For ToxicChat, we evaluated using a short hand-written prompt adapted from some of our internal policies. Our internal Safety Reasoner again outperformed gpt-5-thinking, while gpt-5-thinking and Safety Reasoner both marginally outperform gpt-oss-safeguard-120b and gpt-oss-safeguard-20b. We expect that the relatively small size of gpt-oss-safeguard still makes it preferable for this type of task.

There are two specific limitations of gpt-oss-safeguard. First, we have observed that classifiers trained on tens of thousands of high-quality labeled samples can still perform better at classifying content than gpt-oss-safeguard does when reasoning directly from the policy. Taking the time to train a dedicated classifier may be preferred for higher performance on more complex risks.

Second, gpt-oss-safeguard can be time and compute-intensive, which makes it challenging to scale across all platform content. Internally, we handle this in several ways with Safety Reasoner: (1) we use smaller and faster classifiers to determine which content to assess and (2) in some circumstances, we use Safety Reasoner asynchronously to provide a low-latency user experience while maintaining the ability to intervene if we detect unsafe content.

gpt-oss-safeguard is OpenAI’s first set of open safety models built with the community. We’ve iterated on gpt-oss-safeguard with trust and safety specialists at SafetyKit, ROOST, Tomoro, and Discord as part of early testing. ROOST CTO Vinay Rao says, “gpt-oss-safeguard is the first open source reasoning model with a ‘bring your own policies and definitions of harm’ design. Organizations deserve to freely study, modify and use critical safety technologies and be able to innovate. In our testing, it was skillful at understanding different policies, explaining its reasoning, and showing nuance in applying the policies, which we believe will be beneficial to builders and safety teams.”

We’ll continue to iterate with the community to improve open safety tooling, including through the ROOST Model Community (RMC). The RMC brings together safety practitioners and researchers to share best practices for implementing open source AI models into safety workflows, including evaluation outcomes and model feedback. Visit the [RMC GitHub repo(opens in a new window)](https://github.com/roostorg/open-models) to learn more about this partnership and how to get involved.

To start building with these models, download them from [Hugging Face(opens in a new window)](https://huggingface.co/collections/openai/gpt-oss-safeguard).

## Author

## Keep reading

[View all](https://openai.com/news/)
