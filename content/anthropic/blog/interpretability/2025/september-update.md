Title: Circuits Updates - September 2025

URL Source: https://transformer-circuits.pub/2025/september-update

Markdown Content:
In these monthly updates we report a number of developing ideas on the Anthropic interpretability team, which might be of interest to researchers working actively in this space. Some of these are emerging strands of research where we expect to publish more on in the coming months. Others are minor points we wish to share, since we're unlikely to ever write a paper about them.

We'd ask you to treat these results like those of a colleague sharing some thoughts or preliminary experiments for a few minutes at a lab meeting, rather than a mature paper.

New Posts

In [On the Biology of a Large Language Model](https://transformer-circuits.pub/2025/attribution-graphs/biology.html#dives-multilingual), we studied how similar text translated into different languages is represented in language models. We found that:

Recently, we revisited those results and found a curious phenomenon: the similarity in active features (as measured by intersection-over-union, IoU) increases with increasing sample length.

The upward trend with sample length suggests that either:

On priors we suspected explanation (2) because so many of the original findings aligned with our intuitions about models and language. But, just to be sure, we decided to investigate further.

We calculated the IoU score for the first and last sentences in paired English/French paragraphs. If (1) is occurring these should be similar, since the sentence length is not on average varying across the context, whereas if (2) is occurring the last sentence should show a higher IoU score than the first sentence.

The differences between these are shown below. The distribution is skewed strongly to the right, meaning that the final sentence has a higher IoU score than the first. This is what explanation (2) predicts and is contrary to (1). We think this is evidence in favor of the theory that the model just has a richer understanding later in the context.

We also studied a baseline for this experiment where we take unrelated samples from the two languages. That is, we compare unrelated first sentences from English with unrelated first sentences from French, and likewise for last sentences. If (1) were occurring these should again be similar, whereas if (2) is occurring we should expect more overlap between unrelated first sentences than unrelated last sentences.

The distribution, shown below, skews to the left, meaning there's more overlap between unrelated first sentences than between unrelated last sentences. We think this is again consistent with a story where the scaling with context length is about the model developing a richer understanding.
