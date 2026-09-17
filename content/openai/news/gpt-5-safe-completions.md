Title: From hard refusals to safe-completions: toward output-centric safety training

URL Source: https://openai.com/index/gpt-5-safe-completions

Markdown Content:
# From hard refusals to safe-completions: toward output-centric safety training

Introduced in GPT‑5, safe-completion is a new safety-training approach to maximize model helpfulness within safety constraints. Compared to refusal-based training, safe-completion improves both safety and helpfulness, especially in dual-use domains.

**If a user asks ChatGPT for the minimum energy needed to ignite a firework display, should it give a helpful answer?** The user could be preparing for a July 4th display or a research project for school … or build explosives. As a result, giving a helpful answer could be harmless or harmful depending on the user’s (apparent) intent. This kind of prompt is *dual-use*: a question with unclear intent, where information could be used in benign or malicious ways. Dual-use problems are especially prevalent in risk areas such as biology and cybersecurity.

In the past, production models such as ChatGPT relied on refusal-based safety training: based on the user’s prompt, the model should either fully comply or refuse. While this type of training can work quite well for obviously harmful prompts, it can struggle to handle dual-use prompts like the fireworks example. In that instance, a refusal-trained model makes a binary decision based on how harmful it perceives the prompt to be - and either fully complies (potentially dangerous if the user wants to use the information maliciously), or refuses and says “I’m sorry, I can’t help with that” (unhelpful if the user is truly trying to fix their fireworks display).

For GPT‑5, we introduced a new form of safety-training - safe completions - which teaches the model to give the most helpful answer where possible, while still maintaining safety boundaries. We find this approach to be more nuanced, supporting better navigation of dual-use questions.

## Prompt

## OpenAI o3 (refusal training)

## GPT-5 (safe-completion training)

*OpenAI o3 and GPT‑5 responses to a challenging dual-use prompt asking for instructions on lighting fireworks. o3, which has been refusal-trained, over-rotates on intent: it assesses the prompt as benign and therefore fully complies with the question. In contrast, GPT‑5, which was trained with safe-completions, explains why it cannot fully comply, and then only provides high-level guidance to check appropriate manuals.*

Safe-completion centers safety training on the safety of a model’s *output,* rather than determining a refusal boundary according to the user’s *input.* Concretely this is implemented through two training parameters:

- **Safety constraint** : During post-training, the safe-completion reward penalizes model responses that violate our safety policies (with stronger penalties depending on the severity of the infraction).
- **Helpfulness maximization** : For safe model responses, we reward the model based on its helpfulness: either directly according to the user’s stated objective, or indirectly by providing an informative refusal with helpful and safe alternatives.

We incorporated safe-completions into GPT‑5 (both reasoning and chat models), and found that safe-completion training substantially improves *both* safety and helpfulness compared to refusal-based training. For fair comparison against OpenAI o3, we report the performance of GPT‑5 Thinking versus o3.  In comparisons of both production models and controlled experiments, we find that safe-completions are especially well-suited for dual-use questions. The figure below compares the safety score and average helpfulness score for safe responses.

Safety and helpfulness given safe responses by intent (OpenAI o3 vs. GPT‑5 Thinking, labelled as gpt5-r). GPT‑5 Thinking is both safer and more helpful than OpenAI o3.

By foregoing the comply/refuse binary decision, safe-completion training encourages our models to be more conservative about potentially unsafe content even when they do comply. In our experiments, we find that when safe-completion models *do* make a mistake, their unsafe outputs are lower in severity than the unsafe outputs from refusal-trained models.

Harm severity analysis for unsafe responses (o3 vs GPT‑5 Thinking, labelled as gpt5-r). GPT‑5 Thinking makes less severe mistakes than o3.

It can be easy to trade off helpfulness for safety – a model can be safe if it refuses everything. But we want our models to be both safe *and* helpful. A core research challenge is how to improve both of these goals together. For GPT‑4 we developed [__Rule-Based Rewards__](https://openai.com/index/improving-model-safety-behavior-with-rule-based-rewards/) as a method to trade-off helpfulness and safety. Now, for GPT‑5, safe-completions take another step forward, leveraging the growing capabilities of AI to provide a deeper integration of these two goals. We believe that the focus on the safety of model responses sets a solid foundation to address the growing complexity of safety challenges on the horizon, and we plan to continue this line of research to teach the model to better understand challenging situations and respond with greater nuance and care.

## Authors

## Keep reading

[View all](https://openai.com/news/)
