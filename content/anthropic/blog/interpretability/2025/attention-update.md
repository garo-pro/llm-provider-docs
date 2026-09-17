Title: Progress on Attention

URL Source: https://transformer-circuits.pub/2025/attention-update

Markdown Content:
We report some developing work on the Anthropic interpretability team, which might be of interest to researchers working actively in this space. We'd ask you to treat these results like those of a colleague sharing some thoughts or preliminary experiments for a few minutes at a lab meeting, rather than a mature paper.

Last year, we [announced](https://transformer-circuits.pub/2024/jan-update/index.html#team-update) that studying attention superposition was one of our team’s priorities, motivated in part by a [toy model](https://transformer-circuits.pub/2024/jan-update/index.html#attn-superposition) demonstrating attention features in superposition. Since then, we’ve tried several ideas to study this problem in real language models, and we’d like to give an update on what we’ve tried and where we think the problem is now.

At a high level, we found significant evidence of attention superposition, and additionally of cross-layer attention representations. We looked for and did not find evidence of preferred OV dimensions for attention features

We believe that the core remaining problem is understanding why models attend where they do, which is to say understanding attention pattern formation. We believe we have a promising path forward on this problem, in the form of QK diagonalization, which we report on below.

We want to understand the nature of computation in attention. To get a handle on this problem, we’ve found it useful to decompose it into a number of smaller pieces, each of which would either inform further investigations or else provide a practical tool for analysis if solved. Those pieces are:

We began our study with an ambitious approach that, if successful, would have answered most of these in one go. We then conducted several more targeted studies to solve individual pieces, and we now have preliminary results on several of these.

In the rest of this update we document our approaches to date, explain how each has updated us, and provide a roadmap of next steps.

Our first approach was an “Attention Replacement Layer”. The basic idea was to form a wide (many-head) attention layer and train it to mimic the behavior of an attention layer in the base model of interest (i.e. given the same inputs, train on MSE versus the outputs of the base layer). The wide model would be regularized to be simple, in the sense of the QK and OV circuits being low-rank. In some variants we also regularized to make the attention pattern sparse. With this approach, an attention feature is a single head in the replacement layer.

We made some progress with this approach, including some signs of life (interpretable heads in the replacement layer). At the same time, we hit significant ML difficulties that we were unable to resolve. Our current view is that it is likely possible to get this approach to work, but not easy.

We also made some conceptual progress pursuing this approach. We studied a wide array of different regularization schemes, and found that a regularization penalty of the form [Mazumder+2010](https://www.jmlr.org/papers/volume11/mazumder10a/mazumder10a.pdf) and [Kunin+2019](https://arxiv.org/abs/1901.08168) for more details.

We found some preliminary signs of life with this approach. In particular, we were able to identify heads in the replacement layer implementing specific skip-trigrams and we found heads that appeared to be performing induction. The results were always somewhat messy though, and we spent a fair bit of time trying to understand why.

In the end our most likely hypothesis is that the ML of our setup was not well-optimized. Unfortunately because the replacement layer has a lot of moving parts and was more-different from models we had worked with previously we found it difficult to improve the ML. This was both for engineering reasons (scaling this approach to large numbers of heads efficiently proved challenging) and because there was significant design freedom (multiple hyperparameters for the loss, multiple choices of loss functions, etc.) and no clear quantitative metric for comparing these.

As a result of these challenges, our approach now is focused on getting signal on different aspect of the problem in isolation through a variety of simpler experiments.

To get at questions (1,2,3), we trained Multitoken Transcoders (MTC’s). An MTC takes as input the residual stream over a full context at the input to a layer in the base transformer and predicts either the residual stream after that layer over the full context or the output of the attention part of the layer over the full context. It does this with three components:

An advantage of this architecture is that it can implement a powerful fused Attention+MLP operation (with the identity like “attention head” allowing MLP-like computation to be learned), where information is moved and simultaneously run through an MLP, which allows MTC’s to potentially produce simpler representations of circuits by merging Attention-like operations with MLP-like operations that come either before or after the information is moved[Ameisen+2025](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#building-replacement) has a discussion of how transcoders can be used to study circuits in models. In this context, MTC’s can be seen as extending transcoders to incorporate information about attention.

Additionally, MTC’s form attention patterns in the same way as our [toy model](https://transformer-circuits.pub/2024/jan-update/index.html#attn-superposition) of attention superposition, which makes their performance something of a proxy for the hypothesis of attention superposition.

We train our MTC’s on MSE relative to the objective, plus an L1 penalty on the encoder activations and a different L1 penalty on the negative pre-activations, which we’ve [found](https://transformer-circuits.pub/2025/january-update/index.html) to be an easy and effective way to avoid dead features

With MTC’s we are explicitly not studying QK dimensionality (question 4), cross-layer behavior (question 5), or how attention patterns form (question 6), since we’ve restricted ourselves to a single layer and in some sense inherit the attention patterns from that layer.

We trained MTC’s with 100,000 features on an 18-layer transformer. We found that:

To understand the extent to which attention features exist in superposition (spread across heads), we compared MTC’s against a baseline where each feature is bound to and moved by a single attention head

The fact that restricting MTC features to a single head degrades both performance and interpretability was a moderate update for us towards attention superposition as a hypothesis, especially since MTC’s form attention patterns in precisely the same way as our attention superposition [toy model](https://transformer-circuits.pub/2024/jan-update/index.html#attn-superposition).

We want to understand the extent to which the QK-condition (“where is information moving to/from?”) and the OV-condition (“what information is being moved?”) are naturally coupled.

If these are coupled, we should expect MTC’s to perform better than a baseline shaped like “move SAE features through attention heads”, since the difference between the two is that the MTC’s can jointly learn what information is moving and how to move it, whereas the SAE doesn’t get to see information about the information movement.

We weren’t sure of exactly the right way to make this comparison, so we compared MTC’s against a few different baselines. First, we compared MTC features qualitatively against SAE features run through the single attention head that most-moves that feature in a given context. We made this comparison with the same contexts for both MTC and SAE features. Restricting the SAE features to use a single head favors the MTC in this comparison, but choosing that head dynamically according to the context favors the SAE, since the MTC cannot make that choice dynamically

As a second baseline, and to get some quantitative signal, we trained a model jointly on an SAE target and an MTC target. That is, we trained an encoder and a decoder as an SAE (with no attention operation), and we trained a different decoder as a Top-1 MTC. The SAE & MTC steps see the same dataset examples and are interleaved. The MTC training in this case detaches gradients on the encoder SAE, so the SAE is the sole determinant of feature activations. The idea in this setup is to see if we can pick better features for modelling attention by accounting for attention in the training process, or if SAE features perform comparably-well.

We qualitatively inspected the features that come from this training process and found that they were substantially harder to interpret than the features we see in either regular MTC’s or regular Top-1 MTC’s. The SAE features moved through a single-head MTC decoder were often quite fuzzy (i.e. broadly-attending) or else polysemantic.

We also quantitatively compared this baseline to regular Top-1 MTC’s, and found that the jointly-trained model performed somewhat better in the low-

Overall, these qualitative and quantitative results suggest that the QK and OV conditions are likely coupled in models, and we should choose approaches to attention that allow us to study them jointly.

Above we found semantically-meaningful clusters of features in head-loading space, including clusters of induction features, clusters of features involving delimiters in text, and clusters of features involving code contexts. Since clustering in head-loading space means a shared attention pattern, the differences among these features have to be in what information they move rather than in their patterns.

This raises a question: are these clusters evidence of attention features with many OV dimensions? For instance, an induction feature might have a large OV dimension so it can move a wide range of different pieces of information. Such features would naturally appear as clusters in head-loading space since our MTC only allows one OV dimension per feature.

Since we know that different attention behaviors require different OV dimensions (e.g. one dimension for skip-trigrams, versus many for induction/copying behavior), we might expect to see many small clusters corresponding to skip-trigrams and related behaviors, and a smaller number of large clusters corresponding to induction/copying and related behaviors.

To study this, we clustered the head-loading vectors of our features using 

This suggests that attention behaviors might form more of a continuum in OV-dimension. For instance, it might be that there are broad induction features that move large chunks of the residual stream as well as narrow ones that move smaller chunks. And similarly there might be generalizations of skip-trigram behavior that involve moving more than one dimension at a time.

One potential limitation of this analysis is that we trained each MTC on just one layer. It is conceivable that, if there is a substantial amount of cross-layer attention superposition, our feature clusters might be “blurred” by being limited to a single layer. We address this limitation using multi-layer methods in the next section.

After studying MTC’s, we turned to Attention Value SAE’s, which are SAE’s trained on the concatenated value vectors of multiple attention heads.

We first used Attention Value SAE’s to study cross-layer attention representations. Concretely, we trained three Attention Value SAE’s on an 18-layer model: one on the concatenated value vectors of the heads in layer 4, one on the concatenated value vectors of the heads in layer 10, and one on the concatenation of the value vectors of the heads in both of these layers. We chose layers 4 and 10 because they contained the clearest induction heads, and so seemed interesting to study.

We compared the two-layer SAE to the two one-layer SAE’s by adding the 

In our study of MTC’s we used feature clustering to study the distribution of OV dimensions of attentional features. One concern we had is that our results may have been distorted or “blurred” by the MTC being limited to a single layer.

To address this, we repeated our analysis using an Attention Value SAE trained on two layers. We constructed a head loading vector for each SAE feature using the norm of the decoder vector (per head) as a measure of head loading. We then clustered these vectors as before, using k-means clustering and sweeping over 

The resulting clusters look very similar to those we found with a single-layer MTC: we see a power-law distribution of cluster sizes and no evidence of discrete populations of OV dimensions.

Another strategy we have explored is simply training SAEs on the output of attention layers, as explored in [this prior work](https://www.lesswrong.com/s/FzGeLpkzDgzGhigLm). As in prior work, we observe that these features are broadly interpretable and include induction features and succession features. In addition, we have explored combining them with cross-layer transcoders (CLT’s, which predict MLP outputs) to compute [attribution graphs](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#graphs) that include attention information. This improves on our CLT-only attribution graphs by localizing attention computation to particular layers and context position

Notably, attention output SAE features do not help us address the question of how attention patterns are formed. In addition, compared to MTCs, attention output features provide less information – attention output SAE features have no fixed assignment to attention heads (they receive variable-by-prompt head contributions), and we can only visualize their “query-side” activations (unlike MTC features where we can visualize both key-side and query-side activations). For these reasons, we generally find MTC features to be preferable. However, attention output SAEs are somewhat more convenient to train (in particular, we can train them using only one token position’s activations at a time, unlike MTCs where we need to train on full contexts).

We have also explored cross-layer variants of attention output SAEs that integrate with cross-layer transcoders. One architecture we have found useful for attribution graphs is as follows:

The reason Type 2 features need to be weakly causal is that attention layer outputs are not fully predictable from the preceding residual stream at a single token position. We find this architecture improves MSE/L0 and simplifies attribution graphs somewhat, compared to combining CLTs and plain attention output SAEs. We take this as some evidence for cross-layer representations in attention outputs.

Aside from the Attention Replacement Layer, none of the methods above can tell us, even in principle, about how attention patterns are formed. In many cases this is fine: we can interpret MTC features just by looking at examples they fire on and studying the resulting attention patterns. But in many cases the important question is [why](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#limitations-attention) [the model attended where it did](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#limitations-attention).

For instance, in answering a multiple choice question the model has to choose one of the answers to attend to. All of the interesting logic is involved in this choice! So we really do want a means to understand how attention patterns form.

To this end, we are pursuing an idea we call “QK diagonalization”. The basic premise is to route the query and key vectors from a given attention head in the base model through separate encoders to form query-side and key-side features, respectively. The idea behind “diagonalization” is to then only consider the interaction between the 

Our results so far have shown some signs of life, but using rank-1 QK features it has proved challenging to find a setup that faithfully matches the base attention patterns (i.e., normalized MSE 

This behavior – in the multi-head case in particular – is puzzling because the model is choosing to increase L0 without any improvement in MSE, again, regardless of the total number of features. Unlike when training autoencoders, increasing the total number of features here does not improve MSE (at fixed L0). This points at either an architectural issue – perhaps it is just not possible to faithfully model attention pattern formation using a reasonable number of rank-1 QK features – or an ML issue in our training setup. We suspect it may be the former, and as a result we are now pursuing the high-rank QK feature approach.

That said, the induction features we do find using the rank-1 approach are typically quite interpretable and often significantly more complex than token-based induction. For instance, we have found features that mediate induction on surnames, on syllables within words that start with a C or a K, or on words related to cardinal or spatial directions.

We have only been working on this approach in earnest for a short time, so we expect that there is more to be learned by pushing more in this direction.

For the first time it feels like we have real traction on understanding attention. In particular, we are now more confident that there is attention superposition in models across both heads and layers. We have updated away from thinking that attentional features form discrete families grouped by dimensionality. And we have developed a practical method in the form of MTC’s to answer the questions of “where is the information moving to/from?” and “what information is being moved?”.

Our biggest remaining question is how to understand why models attend where they do. If we had a solution to that, we could combine it with our solutions above to form a complete picture of information movement in models.

Given this, our next steps split into two classes: understanding pattern formation, and further developing our results above to improve our circuit analyses:

We may also return to study attention replacement layers, likely through a lens that looks like a combination of QK diagonalization and MTC’s.
