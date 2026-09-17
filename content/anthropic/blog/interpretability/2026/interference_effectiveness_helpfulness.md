Title: Characterizing interference weights in a tiny language model

URL Source: https://transformer-circuits.pub/2026/interference_effectiveness_helpfulness

Markdown Content:
An ambitious hope for mechanistic interpretability is to read a neural network the way we read a computer program, providing a context-independent (or global) description of its computation 

However, models leverage superposition in maximizing their capacity to represent different circuits 

But even if we found a perfect feature basis for the transformer, those features would still be written and read through low-dimensional projections, and the virtual weights between features would thus be forced into superposition 

We previously identified interference weights in a simple toy model based around a single low-dimensional projection ["definition 2"](https://transformer-circuits.pub/2025/interference-weights/index.html#basic-phenomenology-reproduction-decomposition) in A Toy Model of Interference Weights.  The choice implies that we classify individual weights as interference rather than looking for a latent difference between the model's weights and the "real" ones (["definition 1"](https://transformer-circuits.pub/2025/interference-weights/index.html#basic-phenomenology-reproduction-decomposition)).[Appendix section](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#appendix-interference-weights) of Circuit Tracing 

Here, we train a one-layer transformer, decompose it into the virtual weights between its tokens, positions, features, and logits, and study how those weights combine to produce the model's behavior on the data distribution. This provides a minimal naturalistic setting in which to analyze interference weights, which we approach by scoring weights along two axes. A weight's effectiveness is the magnitude of its effect on the model's outputs. A weight's helpfulness is the magnitude and sign of its effect on the loss. This is the same measurement from the toy model work. Helpfulness is accurate for finding interference weights, but expensive to compute. A weight has to be effective to have helpfulness, but there's no inherent connection between a weight's effectiveness and the sign of its helpfulness.

Our 1L model can complete words like ACETYLCHOLINE`E`" token follows "`IN`".  One term in that prediction is the path from "`IN`" to the logits via the residual stream, whose largest virtual weight votes to complete the word with "`utions`".  This token never follows "`IN`" in the training set, so every time this virtual weight affects the output, it only makes the model's loss worse.  This implies the role of weight superposition as there is no reason that this weight would be large if the model's virtual weights were trained directly.  While prior work has characterized the emergence and identifiability of interference weights in a toy setting, where a neural network is trained to emulate a larger, sparse, ground-truth model, this note is the first place, to our knowledge, that an interference weight like this one has been demonstrated inside a trained transformer by measuring their effect on the training loss 

Expanding upon this result, we then present three major findings. First, helpful and harmful weights are scattered across the entire range of virtual weight magnitudes. Thus, naive attempts to read functional circuitry from raw weights can miss the ones that implement important circuits while highlighting confusing connections that never matter on real data. Second, ineffective virtual weights are abundant and can be easily removed: pruning the least effective 70% only worsens the model's loss by 0.01 nats, and pruning 85% costs 0.1. The most effective weights are overwhelmingly helpful, and they exceed the effectiveness of any harmful ones by an order of magnitude. Third, the number of helpful virtual weights in this basis still appears large. As a crude benchmark, our virtual weight model contains more helpful weights than parameters in the original transformer, even when constrained to the most effective and helpful. The remaining weights may still be more interpretable than the original model, but we suspect that a similar model within a better basis may still provide further sparsity and interpretability.

We can decompose the prediction to complete ACETYLCHOLINE into the separate paths that feed the output and read off what each contributes. When we do so, it's unclear which weights implement circuits for the model that are functional (improving the predictions in some contexts, if not this one), and which act as noise which is merely tolerated.

Our one-layer transformer[Appendix](https://transformer-circuits.pub#app-training-details) for training details.

No path individually ranks "`E`" as the top output.  Each places its largest contribution on a different, incorrect token: the direct path favors "`␣Mrs`", the attention path favors "`ely`", and the MLP path favors "`ATION`".  Instead, "`E`" is the one continuation every path scores somewhat highly.  We display where these votes sit among the other vocabulary effects and how they interact in the [Appendix](https://transformer-circuits.pub#app-pairplot).

What interests us most about these combining paths is that several of these conflicting votes come from confusing-looking connections.  The MLP's strongest vote would complete the word as ACETYLCHOLINATION, which is not a real technical term.  The attention prediction ("`ely`") is lower-case, though every token so far has been upper-case.  And, as we've already highlighted, "`utions`" doesn't seem like it should follow "`IN`" in any context.  There might be reasonable explanations for these connections, but how can we tell?  Do we assume that the model knows something we don't?  Or are these interference weights contributing noise to the model's behavior, existing only because of compromises made in the compression of the residual stream?

In the following sections, we'll show how quantifying each weight's effectiveness and helpfulness helps answer these questions. Ineffective or harmful weights are not valid lessons that the model has learned; they are interference weights that the model carries alongside its actual circuits.

The decomposition above was useful for demonstration, but the attention and MLP paths are too complex as units to permit an interpretable description because they are context-dependent transformations.  We might consider breaking them up into their components, but MLP neurons tend to be polysemantic 

The basis we use consists of tokens, positions, transcoder features, and output logits.  Inputs enter the virtual weight model (the VW model) as a concatenated one-hot encoding of the vocabulary and a one-hot encoding of position into a single [vocabulary, position] axis of size 

These weight families describe the interactions of the basis units between the more complex nonlinearities — five of the families constitute linear maps into logits or features and the QK family is the one bilinear map, producing the attention pattern that conditions the two OV families.  Because our transformer has no normalization, each path is a fixed product of matrices 

Materializing all six families inflates the parameter count from 2.9M to roughly 331M, about 100×, since each pair of endpoints now carries its own explicit weight rather than sharing the residual stream.  Implementing the equivalent forward pass also requires some additional details to manage the input encoding and separated OV paths, and we describe these details in the [Appendix](https://transformer-circuits.pub#app-virtual-weight-model).

Now that we have our set of virtual weights, two questions help us identify the interference weights among them: Does the weight have an appreciable effect within the model? And if it does, does it then lower the loss or raise it? We make each question precise before using them to inspect virtual weights in the next section.

We define effectiveness as the magnitude of a weight's effect on the function the model computes.  We measure it as a second-order estimate of the KL divergence between the model's outputs with and without the weight (using the Fisher metric 

where [Appendix](https://transformer-circuits.pub#app-fisher-math) gives the form the metric takes for each path.  We estimate this value for every virtual weight in the model over ~537M tokens of the training corpus.

Effectiveness measures which of a weight's effects survive interference from every other path.  A virtual weight can be large and still ineffective because its target is never plausible when its source is active and competing paths reliably outvote it.  For ACETYLCHOLINE, "`IN`"'s Tokens→Logits contribution to "`E`" survived the competing votes and contributed probability mass to a reasonably probable target.  Even though "`utions`" carried the larger raw virtual weight, that completion was sufficiently downvoted by other paths that the virtual weight did not contribute much to the output probability distribution.

Effectiveness says how much a weight moves the predictions, not whether it made those predictions more correct. For that we measure helpfulness: the average change in loss when the weight is ablated from the forward pass. If the loss rises when we remove a weight, the weight is helping the model toward the right answer; if the loss falls, it is harmful.

Recovering the sign of a weight's helpfulness with statistical significance takes a lot of data, because a single weight typically helps on some tokens and hurts on others ([Appendix](https://transformer-circuits.pub#app-single-helpfulness)).  Helpfulness can be computed cheaply for weight families that target the logits ([Appendix](https://transformer-circuits.pub#app-helpfulness-logits)), but for the rest it is expensive.  We compute the average helpfulness for the subsets of weights in our worked examples over the entire training set and attach 95% (Gaussian) confidence intervals to each one.  We ran similar measurements for a random sample of the whole population over 1B tokens.[Appendix](https://transformer-circuits.pub#app-mean-helpfulness) for reference.

The expected residual attribution (ERA) and target-weighted (TWERA) measurements in our previous work [counterfactual variant](https://transformer-circuits.pub#app-fisher-math-features) for some weights that helps handle inhibition.[Appendix](https://transformer-circuits.pub#app-fisher-ea)), we tried several related variants, and we do not think that further sharpening of this metric is high-leverage compared to, say, finding better bases.  Later, we'll suggest that the helpful weights are tens of percent of the VW model, and that pruning by Fisher effectiveness reaches a similar density before the loss suffers.  Since helpfulness is the most direct per-weight measure and Fisher already prunes to a similar density, a sharper metric has little room to improve.

Having defined effectiveness and helpfulness, we can now use them to identify interference weights and interpret parts of the VW model.  We work through four examples.  The first returns to ACETYLCHOLINE to confirm that the confusing Tokens→Logit weight from "`IN`" was in fact interference and to show that effectiveness can filter interference weights from the most prominent effects.  The next three demonstrate lessons related to the effect: sometimes the largest magnitude virtual weights are indeed helpful; the most Fisher-effective weights tend to be helpful but are sometimes harmful; and weight families which do not target the logits have lower Fisher-effectiveness than those that do, but these families still show the same qualitative patterns relating weight magnitude, effectiveness, and helpfulness.  We walk through the last example in more detail, as effectiveness proved useful in interpreting the relevant circuitry.

As mentioned before, the largest Tokens→Logits virtual weight from "`IN`" points at "`utions`".  That weight's Fisher effectiveness, however, is around three orders of magnitude below that of the most effective weights from the same token.  It is also harmful; across the model's entire training set, "`utions`" never once follows "`IN`".  Every time this weight changes the output distribution, it pushes the predictions in the wrong direction.

Re-sorting "`IN`"'s weights by Fisher effectiveness not only drops "`utions`", it also lifts the "`E`" from ACETYLCHOLINE to second place.  The effectiveness sort is more interpretable across the top few weights, and, for these weights, it largely agrees with sorting by helpfulness directly.  The most effective weights from "`IN`" are all upper-case continuations consistent with the all-caps contexts where "`IN`" tends to appear, and they are phonetically plausible.`T`", ranked at the top, is indeed more likely to follow "`IN`" than "`E`".

In addition to sorting weights by effectiveness or helpfulness, we can use these measurements to inspect the largest virtual weights after filtering them by a threshold of either value. The virtual weights describe the direct effect of each unit on another globally, so we think of filtering the weights by this kind of threshold as attempting to instantiate a version of the model without interference weights. For this example, filtering by either helpfulness or effectiveness removes the expected interference weights and leaves us with upper-case continuations. We'll study this filtering operation in more detail later when trying to study the VW model as a whole.

Zooming out from the top few weights to the full distribution shows what the effectiveness sort does in aggregate.  We co-vary virtual weight and Fisher effectiveness with helpfulness.  Any weights whose 95% confidence intervals include zero are colored gray as "not significant".[the Appendix](https://transformer-circuits.pub#app-rope-plots) for effect size analysis.

Virtual weight shows a weak relationship to helpfulness (panels 1 and 2); for this set, the virtual weight highlights few of the most important connections.  Fisher effectiveness has a much clearer relationship with helpfulness (panel 3).  Weights with increasing effectiveness bifurcate into helpful and harmful weights, with the most helpful extending to the largest effectiveness values.[Appendix](https://transformer-circuits.pub#app-effectiveness-CDF).

The two measurements have very different scales: virtual weights appear approximately normally distributed, while Fisher effectiveness is roughly lognormally distributed, varying by 10 orders of magnitude. The median magnitude of virtual weight is approximately one third of the magnitude of the largest weight, while the median effectiveness of a virtual weight is 10,000× less than the maximum effectiveness.

This example might suggest that large virtual weights bury the most important weights, but that's not always true.`。`" (the equivalent of a period in East-Asian writing), a cluster of byte tokens such as "`\xe5\xae`", and "`\xe4\xbb`" (the leading bytes of the multi-byte CJK characters the feature predicts).  These high-magnitude weights are quite helpful.  The feature's strongest negative weights suppress Modern Latin-script tokens like "`␣European`" and "`␣fran`", which are intuitive for a Chinese-text feature.  Sorting or filtering barely changes the picture: the top weights are again CJK byte-prefixes ("`\xe5`", "`\xe6`", "`\xe7`").[Towards Monosemanticity](https://transformer-circuits.pub/2023/monosemantic-features#feature-analysis); compared to the neuron basis, weights are generally easier to interpret within the feature basis.

See the virtual weight, effectiveness, and helpfulness plots in the [figure gallery](https://transformer-circuits.pub/figure_gallery/index.html?page=chinese_feature_to_logits)

Fisher effectiveness reliably highlights the most helpful weights, but it does not fully remove interference weights. Instead, the top of the ranking is enriched for the most helpful and harmful weights and the most effective weights are often helpful. We can see this most clearly in a feature that primarily responds to the ends of words in European languages (especially French). Its raw virtual weights again lead with ineffective and harmful connections targeting Arabic-script tokens instead of French.

See more information on these distributions in the [figure gallery](https://transformer-circuits.pub/figure_gallery/index.html?page=french_feature_to_logits)

Sorting or filtering by effectiveness moves these Arabic-script tokens down the ranking, but it does not entirely "clean" the weights. The most harmful weights remain intermingled with the helpful ones across the high-effectiveness range, even though most of the weights on the large-effectiveness end are helpful.

Every example so far has analyzed weights between features and the logits.  Weights that target the model's internals (the QK, Tokens→Features, and Tokens→OV→Features weights) are generally less effective and less helpful, but the same patterns still appear.  We demonstrate how effectiveness can help to interpret feature activations and show an example of QK weights in the [figure gallery](https://transformer-circuits.pub/figure_gallery/index.html?page=QK_from_url_start).

Consider this feature that restricts its strong activations to newline tokens, and shows much weaker activations on some whitespace tokens. Its top-50 activations all occur within repeated declaration blocks (common in Java, TypeScript and C# member lists), and its Features→Logits virtual weights predict newline and other whitespace tokens.

The largest positive Tokens→Features virtual weights to this feature originate from positions. Despite the feature firing on newline tokens and weakly on specific whitespace tokens, the newline token weight is buried deep in the distribution of weak positive weights and the whitespace tokens carry its strongest negative weights! How does this feature know when to fire?

Seeing the virtual weights, we might try to understand the activation pattern using the position weights.  This turns out to be misleading.  Position input to this feature comes both from the Tokens→Features path and the OV path (Tokens→OV→Features), and it turns out that these two sum to a roughly constant bias.  We can see this clearly by covarying the two (plotted in the [Appendix](https://transformer-circuits.pub#app-position-input))

We get better direction by inspecting which weights are most effective and helpful. The most helpful or effective weights are negative weight whitespace tokens and the newline (see them by filtering to the top few above!).

We've already accounted for three inputs to this feature: vocabulary input from Tokens→Features, position input from Tokens→Features, and position input from Tokens→OV→Features.  The only other input comes through attention from vocabulary tokens.  The role of the Tokens→Features path must be to distinguish the current token in contexts containing relevant tokens from code.`public`", "`);`", and others.

While the whitespace and newline tokens recruit similar amounts of attention input (y-axis), the activation threshold for the whitespace tokens is much larger due to the negative direct path weights. This means that the feature relies on the attention path to signal when it should activate and negates that signal for over tokens other than newline. Effectiveness highlights where the role of inhibition is greater than that of excitation.

More generally, we've seen that effectiveness highlights the most helpful weights across each of the examples above. Sorting by effectiveness is not always necessary to interpret parts of this model, but a large fraction of the most effective weights are helpful.

We can arrange our understanding of each transcoder feature above into a simple circuit diagram, giving an intuitive sense for the relative merits of sorting by virtual weights, effectiveness, and helpfulness.  We also collected the features we've described here and 18 others in [a separate page](https://transformer-circuits.pub/feature_vis/index.html) to give a loose sense for how well filtering works.

The examples above gave an impression of how effectiveness and helpfulness relate in a single node of the VW model at a time. How does the model allocate effectiveness between helpful and harmful directions as a whole?

We first sample 1,111 weights from the six virtual weight families and estimate each one's mean helpfulness.[the fraction of weights that are non-null at several effect sizes](https://transformer-circuits.pub#app-rope-plots).

This display sorts the sample into three regimes for each weight family.  The ineffective weights unsurprisingly have near-zero helpfulness.  Where effectiveness is moderate, helpful and harmful weights are mixed, and effectiveness alone does not separate them.  Only helpful weights remain in the highest effectiveness regime.  The most effective helpful weight out-measures any harmful weight by an order of magnitude or more within each weight family.  This gap is much larger and more consistent than within the same plot for virtual weights ([Appendix](https://transformer-circuits.pub#app-vw-helpfulness)).

The data gives a clear understanding of the model's priorities. Its most effective contributions to the output are overwhelmingly the helpful ones, and the weights that actively hurt the loss are confined to a lower range of effectiveness. This is intuitive; a model trained to minimize loss has every reason to get its most consequential weights pointing in a good direction, and relatively little reason to police weights whose effects barely reach the output.

Our deeper motivation to study interference weights is the hope that, once we identify them accurately, we can then remove them to extract a sparse model whose global circuitry we read directly 

First, we remove weights from the VW model in order from least to most effective (moving from right to left above) and measure the pruned model's loss on a held-out test set.  This costs approximately 0.01 nats at 70% sparsity (30% density) and under 0.1 nats at 85% sparsity (15% density).  This performs far better than raw virtual weight magnitude for every density and individual weight family (except for negative Features→Logits weights, see [Appendix](https://transformer-circuits.pub#app-threshold-eval)), similar to previous pruning literature 

It is natural to consider using helpfulness for this thresholding experiment instead as it directly measures whether removing a weight raises the loss.

Across the sample, roughly half (47.6%) of all weights have positive mean helpfulness (with 12.7% dead), and even counting only those whose confidence interval excludes zero leaves tens of percent (see more details in the [Appendix](https://transformer-circuits.pub#app-full-helpfulness-table)).  Since our VW model increases the overall number of parameters by two orders of magnitude, we're still left with tens of millions of weights to interpret for a single-layer model.  

Not allowing ourselves to remove any helpful weight is much stricter than many applications of circuit pruning today [Appendix](https://transformer-circuits.pub#app-helpfulness-mass)), but this fraction quickly climbs to a similar regime (13.6%) when accounting for 99% of the helpfulness mass.  It is difficult to say which level of helpfulness mass is a better guide, particularly when we also expect nonlinear effects from removing multiple weights together.  We note that keeping 2% of the virtual weights would roughly double the number of parameters in the filtered, expanded VW model relative to the original transformer; if it were the case that all of the resulting weights were quite interpretable, we would be closer to an "upstairs" lift than previously observed.  Computing helpfulness for all weights is expensive, even in a tiny model like this, but the theoretical possibility remains intriguing.

We identified interference weights via two properties: whether a virtual weight does anything to the model's outputs, and whether it lowers or raises the loss on the training data. We used these measurements to make the theoretical implications of superposition concrete within a trained transformer model, demonstrated how identifying and filtering out interference weights helps to interpret model components, bounded the extent to which they affect the model's predictions, and measured our progress towards extracting a sparse interpretable model.

For practitioners reading circuits off virtual weights, we find that large virtual weights are not guaranteed to be helpful or even effective.  This does not make them inherently misleading — they reflect a real part of the model's forward pass — but it is cause for some caution in interpreting them directly.  Sometimes the largest weights are the functional ones, as for the Chinese feature whose top weights predicts intuitive byte-prefixes, and sometimes the largest weight is interference that never influences the output, as for the direct path Tokens→Logits weight from "`IN`"→"`utions`".  Sorting by effectiveness can help to highlight important weights and interpret the model's functional circuitry, even if it does not cleanly separate all helpful weights from the harmful ones.

We find that the model puts its most effective weights in helpful directions and confines its harmful ones to a lower-effectiveness range. Helpfulness is close to a ground-truth measure of a weight's functional importance since it reads the change in loss directly, and we find that the cheaper effectiveness measurement largely tracks it at the largest values. Between these helpful weights and ineffective ones, the model contains a middle regime of helpful and harmful weights with marginal effectiveness. Harmful weights are a sign of the inherent tradeoff between implementing more, or more complex circuits, and the interference caused between them.

This work focuses on a small transformer to leverage the most accurate and expensive tools we have to identify interference weights.  Fisher effectiveness and helpfulness may be useful for interpreting [logit effects](https://transformer-circuits.pub#app-helpfulness-logits) in other settings, but they do not scale nicely to frontier models across other families, so further development of scalable proxies like ERA and TWERA 

We were unable to reach a highly sparse and interpretable model using Fisher effectiveness as the filtering criterion, and we suspect, based on our helpfulness computations, that no saliency scheme will perform much better.  We think this says less about the potential for a sparse interpretation of this model than about the basis in which we study it.  We expressed our virtual weight model using the coordinates of tokens and features of this transformer, and the same model can be dense in one basis but sparse in another 

Avoiding interference weights was the original reason to study per-prompt attribution graphs rather than the global weights, and identifying them turns out to be necessary but not sufficient for reading global circuits.  We see the metrics here as instruments that might let such a basis be recognized once found: a better decomposition, if it exists, is one in which the vast majority of weights can be discarded as ineffective or harmful, and we are left with a sparse helpful set to interpret.

Interference weights are one of the stranger implications of trained models approximating high-dimensional circuits through low-dimensional bottlenecks; despite significant optimization pressure, large, confident connections can actively harm the model's performance. Concretely identifying these weights helps us see these circuits (and their required compromises) in action, showing that these superposed circuits play a significant role in model computation.

Superposition [related work](https://transformer-circuits.pub/2022/toy_model/index.html#related) section of [Toy Models of Superposition](https://transformer-circuits.pub/2022/toy_model/index.html)).  Toy Models of Superposition [A Toy Model of Interference Weights](https://transformer-circuits.pub/2025/interference-weights/index.html) 

The difficulty interference poses for reading circuits off weights is the subject of the [global-weights analysis](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#global-weights) in our circuit-tracing methods 

Removing weights by their lack of importance or salience has a long precedent in the pruning literature.  Magnitude pruning with retraining removes roughly ninety percent of the parameters of vision models without hurting accuracy 

A complementary line of work incentivizes weight sparsity during training rather than scoring it afterward. Gao et al. 

The units of an interpretable weight description need not be the original matrices of a given model parametrization.  Attribution-based Parameter Decomposition 

A related line of work, developed under the headings of singular learning theory and developmental interpretability, likewise relates parameters to their effect on the loss, but through the geometry of the loss landscape rather than through individual virtual weights.  The local learning coefficient (LLC) of Lau et al. 

We thank all the members of the Anthropic interpretability team for providing feedback on the work. We are grateful to Thomas Conerly and Chris Olah for assistance reviewing the paper, as well as Kelley Rivoire and Chris Olah for their organizational leadership.

For attribution in academic contexts, please cite this work as

Turner, et al., "Characterizing interference weights in a tiny language model", Transformer Circuits, 2026.

BibTeX citation

```
@article{turner2026interference,
  author={Turner, Nicholas L. and Wu, Jeffrey and Batson, Joshua},
  title={Characterizing interference weights in a tiny language model},
  journal={Transformer Circuits Thread},
  year={2026},
  url={https://transformer-circuits.pub/2026/workspace/index.html}
}
```
The transformer we study is a one-layer, decoder-only transformer with residual width 

Attention is causal with standard softmax; scores are scaled by 

The tokenizer is a 4,096-token BPE vocabulary obtained by truncating the tokenizer of the publicly released Pleias-1.2B model to its first 4,096 token ids 

Training text is drawn from the openly licensed Common Corpus (PleIAs) 

Training used Adam 

The model trained on ≈9.8×10⁷ unique tokens (95,464 unique sequences) over its 11,933 steps in a single pass with no data repetition. Training loss fell from 8.93 (≈ln 4096 at initialization) to ≈3.33 at the final step and was still decreasing when the step budget was exhausted.

A single-layer transcoder (SLT) was trained on the transformer's activations: it reads the pre-MLP residual-stream activation and is trained to predict the MLP output. It has 4,096 features over the 256-dimensional input, with a JumpReLU activation 

The loss is a reconstruction term (squared error summed over output dimensions, averaged over the batch) plus a sparsity penalty (the L1 norm of feature activations weighted by the corresponding decoder-column norms) 

Activations were centered and normalized for training per input dimension using statistics estimated from 

In the [Motivating Example section](https://transformer-circuits.pub#example), we visualized the effects of three paths through the transformer when predicting ACETYLCHOLINE. Here we put those effects in context with all of the other logit effects, and compare them to one-another.

The [Virtual Weight Model](https://transformer-circuits.pub#virtual-weight-model) section introduced the six virtual weight families as products of matrices that contract away the residual dimension. This section records the details needed to run the virtual weight model as an exact replacement for the transformer's forward pass.

Inputs enter the VW model as a concatenated one-hot encoding of the vocabulary and a one-hot encoding of position, so each input to the VW model is a vector 

We materialize equivalent matrices to read this concatenated encoding by using the projection of each token embedding and position embedding onto the original transformer matrix. We can express this using a combined matrix that concatenates both embeddings from the original transformer 

The OV weight families (Tokens→OV→Logits and Tokens→OV→Features) act on the tokens that attention "moves" through the attention pattern. To express them as fixed weights we introduce a moved-token representation: applying the attention pattern to the stacked [token, position] one-hot vectors yields, at the query position:

where 

The single-layer transcoder predicts the MLP output but does not reconstruct it perfectly; there is a residual on each token 

Combining all of the details above, we give a schematic diagram of the VW model we analyze, along with its connections to the trained transformer.

The [Effectiveness and Helpfulness](https://transformer-circuits.pub#effectiveness-helpfulness) section gave an expression for Fisher effectiveness:

Three families target the logits directly: Tokens→Logits, Features→Logits, and Tokens→OV→Logits. For any weight in these families, you can represent its effect on the logits using an attribution vector 

Every other entry of the attribution vector is zero, so the general quadratic form simplifies to an expression of 

Let the attribution of a Tokens→Features or Tokens→OV→Features weight to a feature's activation be 

An important choice is how 

We derive a closed-form for the attribution to the attention pattern from modifying a single attention score. We then use that expression to find another for the imposed attribution vector 

The same token can appear multiple times in a context and use the same QK weight to modify the attention scores. In the derivations below we take these uses as "independent" and our expressions are not exact when the same token appears multiple times.

Fix a query position, 

defining 

Note that we can rewrite this quotient as

This implies that 

Shifting the attention score at a single position also shifts the pattern value at 

Consider the shift in each attention pattern value 

For 

The expressions for 

where we define 

For each context position 

The direct attribution to the logits follows by summing the pattern shift against these per-position vectors. Ablating the weight shifts the head's output at query 

where we use 

All of this in place, we can write an expression for the attribution of one use of a QK weight to the logits as:

Overall, this gives the following expression for our Fisher effectiveness equation over the QK weights:

For tractability the indirect route through the transcoder treats features as independent, using the full variance for each feature but dropping cross-feature covariances. The covariance between the direct and indirect routes is also dropped.

All expectations are estimated by averaging over 

The scores are designed to be computable from forward passes alone in a single streaming pass over the corpus for every weight simultaneously. Two structural facts make this cheap. First, the quadratic-form scores factor into a weight-dependent part and an activation statistic. For example, the Fisher effectiveness computation for weights to the logits 

We defined [helpfulness](https://transformer-circuits.pub#effectiveness-helpfulness) as the average change in loss when a weight is ablated from the forward pass, and noted that for weights targeting the logits the average can be read off cheaply. Here we derive the closed form to implement that computation. As in the effectiveness computation for these paths, a weight in the Tokens→Logits, Features→Logits, or Tokens→OV→Logits family contributes 

Fix a position and let 

with primes marking post-ablation values.

Consider the change in loss under ablation, 

When 

The two cases condense with an indicator:

The weight’s helpfulness is the expectation of this per-position change over the data distribution,

We noted in [Effectiveness and Helpfulness](https://transformer-circuits.pub#effectiveness-helpfulness) that recovering the sign of a weight's helpfulness takes a large amount of data because a single weight typically helps on some tokens and hurts on others. We show more granular data for 10 sampled weights from each decile of the helpfulness distribution. We plot a histogram of the per-batch change in loss under ablation for each weight, rather than its overall average. Each batch consists of 16,384 tokens. You can show more or fewer weights or deciles using the sliders. The `linthresh` parameter was only chosen to separate zeros from nonzero data.

The distributions are typically broad and centered close to zero, with mass on both sides even for weights whose mean is confidently positive.

We described the helpfulness metric in [Effectiveness and Helpfulness](https://transformer-circuits.pub#effectiveness-helpfulness) and showed a granular version of the data [above](https://transformer-circuits.pub#app-single-helpfulness). Here we plot the distribution of overall mean helpfulness across our sample of 7,765 weights computed over 1B tokens for reference, split by weight family. As above, the `linthresh` parameter was only chosen to separate zeros from nonzero data.

The distributions are centered near zero and are roughly symmetric, with the helpful side slightly heavier.

For reference, we plot the CDF of the distribution of overall mean helpfulness values. The shaded regions indicate where the mean helpfulness value typically cannot be distinguished from 0. We compute that region using a rolling median of the standard error estimate computed over the nearest 1% of the data at each point.

In the [pruning section](https://transformer-circuits.pub#pruning), we discussed a simple model of pruning performance that relaxes the constraint of keeping all (statistically significant) helpful weights and instead preserving the sum (or mass) of the helpful weights. This shows the distribution of helpfulness mass across the helpful, harmful, and nonzero weights within the random sample of 7765 weights (6750 of which have nonzero helpfulness). Each mass (y-axis value) is computed separately for each population (between positive, negative, and nonzero), but the percentage annotation refers to the fraction of the entire sample.

The sum of positive helpfulness values in this sample is 1.44×10-4. The sum of negative helpfulness magnitudes is 1.12×10-5.

For reference, we plot the CDF of the distribution of Fisher effectiveness values. As [above](https://transformer-circuits.pub#app-sample-help-cdf), the shaded regions indicate where the effectiveness value typically cannot be distinguished from 0. We compute this region using a rolling median of the standard error estimate computed over the nearest 1% of the data at each point.

For reference, we plot the mass distribution of Fisher effectiveness values. Unlike the [helpfulness plots above](https://transformer-circuits.pub#app-helpfulness-mass), this includes every weight in the VW model. The mass is computed across each population (positive, negative, and nonzero), but the percentage annotation refers to the fraction of all weights.

331,350,016 weights, 315,487,613 with nonzero effectiveness

The sum of effectiveness across all positive weights is 3.62. The analogous sum across negative weights is 1.59.

[Above](https://transformer-circuits.pub#effectiveness-helpfulness) we asserted that Fisher effectiveness is quantitatively similar to previous metrics we've used as proxies for effectiveness. Here we compare a version of expected attribution to a matching version of Fisher effectiveness. The version of expected attribution includes some changes from the ERA equations from Ameisen et al 

In the [main text](https://transformer-circuits.pub#examples), we described a feature that activates on newline tokens (primarily within declaration blocks). Here we plot its input from position embeddings, across the Token→Feature path and the Token→OV→Feature path. These two roughly cancel and add a small amount relative to the model's effective activation threshold.

In the main text, we showed that the tail of effective weights is also helpful. Here we plot the same relationship for virtual weights, finding that the gap between the weight of the most helpful weights and harmful ones is present for some paths, but not consistently so.

In the [pruning section](https://transformer-circuits.pub#pruning), we showed how the VW model performs after removing weights in increasing order of effectiveness. We perform the same experiment here for each weight family separately, and split it further into studying positive and negative weights.

In the [main text](https://transformer-circuits.pub#aggregate), we attached 95% (Gaussian) confidence intervals to the mean helpfulness of each of the 7,765 sampled weights, and in the aggregate and distribution figures, any weight whose interval includes zero lands in the gray “not significant” bin. That analysis ignores effect size, and with 1B tokens per estimate a weight can be deemed statistically significant while having a negligible effect on the loss.

Here we classify each sampled weight against a region of practical equivalence (ROPE)

Which effect sizes should we care about? A naive yardstick is the VW model’s loss budget: predicting uniformly over the 4,096-token vocabulary costs 

Here we expand the [table from the main text](https://transformer-circuits.pub#pruning) describing the proportion of weights that are deemed helpful and harmful by our simple significance testing.
