Title: Sparse mixtures of linear transforms

URL Source: https://transformer-circuits.pub/2025/bulk-update

Markdown Content:
This work is a preliminary update describing a method we are still developing. As such, many of the results we present here are incomplete. We present them in the hopes that it inspires follow-up work and improvements from the external research community.

Upon publishing this update, we became aware of contemporaneous work on a highly-related architecture (“Mixture of Decoders,” [Oldfield et al.](https://arxiv.org/pdf/2505.21364)). We have updated the post to describe the similarities / differences between our methods in the Related Work section. We recommend that anyone interested in this work read theirs as well!

In our [recent work](https://transformer-circuits.pub/2025/attribution-graphs/methods.html), we trained [transcoders](https://arxiv.org/abs/2406.11944) – sparse, extra-wide MLPs – as more interpretable replacements for a model’s original MLP layers.  We used the transcoder neurons (“features”) as a basis for understanding model computation. We described computations using “attribution graphs,” which depict the causal interactions between features that give rise to the model’s outputs.

While this approach has proved very useful, we believe that transcoder features have key limitations. Transcoders “shatter” model computation into many extremely granular pieces.  This can lead them to represent computations in a different, less efficient way from the underlying model, and can introduce pathologies like feature [splitting and absorption](https://arxiv.org/html/2409.14507v3).

To take one specific example, we found that transcoders decomposed addition circuits using [“lookup table” features](https://transformer-circuits.pub/2025/attribution-graphs/biology.html#dives-addition) that represented individual, highly specific computations, like “6 + 9 = 5 (mod 10)”.  Other work suggests that transformers embed their representations of numbers in a [geometric structure](https://arxiv.org/pdf/2502.00873) such that simple transformations can be used to compute arithmetic operations. While the lookup table description of addition is accurate in some sense, it misses key structure in the model, and is highly inefficient. This inefficiency is a serious issue in practice – in order to train transcoders that are sufficiently large to capture all the computation in a model, we would likely require an astronomical number of features, using a transcoder with many more parameters than in the underlying model.

In this preliminary update, we provide a description of a method we have been working on that is intended to address some of these concerns. Our approach is to replace MLPs with a sparse mixture of linear transforms (MOLT). Unlike a transcoder, a MOLT does not learn sparsely active features that are embedded along a vector direction in the model’s activation space. Instead, it learns sparsely active transforms, which apply a linear transformation to the residual stream to give their contribution to the MLP output. Unlike transcoder features, which double as both computational and representational units, MOLT transforms are purely computational objects that “bridge” representations between layers. As such, they are not intended to be studied on their own, but in combination with another representation decomposition algorithm like SAEs.

So far, we have found that MOLTs are a more compute-efficient and mechanistically faithful way of replacing MLPs than transcoders are. We find that the conditions in which MOLT transforms are active are similarly interpretable to transcoder features. Our preliminary experiments suggest that these transforms can be used to understand how features in one layer are transformed into features in a subsequent layer. MOLT transforms can be incorporated into an attribution graph by annotating graph edges with the set of transforms that “carried” the edge. We also suspect (but have not yet shown) that compositions of MOLT transforms can be used to understand compositional structure in model representations.

A MOLT is parameterized as follows:

where

A MOLT is trained to mimic the output of a model MLP layer, just like a transcoder is. We apply a sparsity penalty (e.g. L1 or tanh) to the activations of the transforms, scaled by the Frobenius norm of the transform matrix (i.e. we penalize 

The [mixture of decoders](https://arxiv.org/pdf/2505.21364) (MxD) architecture (Oldfield et al.) is quite similar to ours – it replaces MLP layers with a sparse mixture of linear transforms.  There is one key implementation difference between MxDs and MOLTs. Using a sparse mixture of many independently learned, full-rank transforms is intractable because it would require too many parameters. MOLTs get around this issue by making the linear transforms low-rank (with a distribution of different ranks; see below). MxDs allow the transforms to be full-rank, but parameterize them in a way that shares parameters across different transforms. 

Specifically, (translating the MxD paper notation into ours), each 

We also note that our intended use-cases for MOLTs (described later in the post) are somewhat different than the focus of the MxD paper – in particular, we are especially interested in interpreting MOLTs as implementing (potentially compositional) transformations between residual stream features (e.g. from SAEs), and integrating them into attribution graphs, treating MOLT transforms as a kind of MLP analog to attention heads.

MOLTs also bear some resemblance to [skip transcoders](https://arxiv.org/abs/2501.18823), which augment transcoders with a linear transform; however, they differ in several ways:

What’s the intuition for why sparsely active transforms might be a good way to represent computation? We offer a few perspectives.

An important hyperparameter in training a MOLT is the allocation of ranks to transforms. This is a high-dimensional hyperparameter space, and we have not explored it fully. However, when training MOLTs on Claude 3.5 Haiku, we have obtained our best performance (in terms of the MSE/L0 pareto frontier) using a distribution of ranks, varying from 32 to 512. Concretely, we use a collection of N transforms of rank 512, 2N of rank 256, 4N of rank 128, 8N of rank 64, and 16N of rank 32. To increase the scale of runs we vary N, but keep the proportions the same. We have found that using transforms of variable ranks outperforms using transforms of all the same rank, controlling for the total number of parameters.

We trained MOLTs (using the rank allocation given above) and transcoders on the middle layer of Claude 3.5 Haiku, varying the amount of compute used in the run. We scaled the number of training steps proportionally to the number of features, and matched the number of parameters between transcoder and MOLT runs. Thus each 4× increase in FLOPs reflects a 2× increase in both number of parameters and training steps. The largest (“1024x FLOPs”) transcoder runs contain approximately 10 million features.

We find that at a given L0, the reconstruction error (MSE) is significantly lower for MOLTs than transcoders, controlling for the number of parameters. The smallest MOLT runs here Pareto-dominate transcoder runs that use 1024× as many FLOPs. Moreover, transcoder performance appears to be saturating at the higher compute scales (though it is possible this flattening is simply due to poor ML tuning), while we observe no such saturation for bulk runs.

We also evaluated the [mechanistic faithfulness](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#evaluating-model-faithfulness) of MOLTs compared to transcoders – that is, the degree to which the MOLT (or transcoder) responds to input perturbations in the same way as the underlying MLP layer. In the limit of infinitesimal perturbation sizes, faithfulness can be computed by comparing the Jacobians of the replacement layer to the underlying layer on a given datapoint, and averaging over datapoints.  We find that MOLTs have a substantially higher Jacobian correlation (cosine similarity of the flattened Jacobian matrices) than transcoders do, at the same L0; moreover, the faithfulness of transcoders appears to deteriorate with scale, whereas that of MOLTs is more stable.  The greater faithfulness makes sense, given that the Jacobians of transcoders are constrained to be low-rank (rank upper-bounded by the L0), whereas MOLT Jacobians can have rank much higher than their L0. 

Note that the results below are from a different model than those above (the 18-layer model used in our [circuit-tracing paper](https://transformer-circuits.pub/2025/attribution-graphs/methods.html), rather than Claude 3.5 Haiku); we have not yet performed this Jacobian analysis on Haiku. MOLT runs with a given number of “feature-equivalents” have the same number of parameters as a transcoder run with that many features.

Transforms are characterized by two properties:

To understand the first part, we can use the same visualization strategy we use for SAE and transcoder features – highlighting dataset examples that activate the transform. When we do so, we find that transforms appear qualitatively similar to features – we see transforms that select for token-level information in earlier layers, and transforms that select for more abstract contextual information in middle and later layers.

We also observe that higher-rank transforms skew higher-density (i.e. are active more often). However, the higher-density, higher-rank transform conditions still appear comparably interpretable to lower-density transforms (and to transcoder features). For instance, we observed a high-rank transform that activates on period tokens, and another that activates on text written in Spanish.

Interpreting the function of transforms is more difficult. Our initial strategy was to train SAEs on the residual stream prior to and immediately after the MOLT layer, and identify feature-feature pairs that most strongly interact via a given transform ([interference weights](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#global-weights) that makes raw interaction strengths between transcoder features difficult to interpret.

We have had more success interpreting transforms in the context of attribution graphs, as described below.

In our recent paper, we constructed [attribution graphs](https://transformer-circuits.pub/2025/attribution-graphs/methods.html) built on top of transcoder features. Edges between features were computed by determining the influence that a source feature (via its decoder direction) exerted on a target feature’s encoder direction (either directly, via residual connections, or via attention heads).

MOLT transforms have no fixed decoder direction that they write out to, so the same attribution graph strategy cannot be applied to them. However, we have had preliminary success with another attribution graph strategy:

Using such attribution graphs, we have uncovered instances of MOLT transforms performing interpretable computations, such as:

However, we also see transforms playing roles that are less clearly interpretable. For instance, many edges appear to be mediated by transforms that select for key words like “is” or “Assistant.” We also see transforms playing roles that appear redundant with the features they carry, such as a “Paris” feature receiving input from a “France” feature via a “France” transform.

We believe MOLTs are a promising alternative to transcoders and may be able to capture MLP computation in a more parameter-efficient way that more faithfully reflects the computations performed by the underlying model. We suspect that a MOLT-like solution will be necessary to capture all the variance of frontier model MLP layers at a reasonable computational cost. We see signs of life that MOLT transforms can perform interpretable computations, “transforming” input features into output features.

One direction we are excited about, but have not explored yet, is using MOLT transforms to understand compositional representations that are not captured by our SAEs. In particular, the reconstruction error of our SAEs at each layer can be rewritten as a sum of terms corresponding to (feature, transform) pairs from the previous layer. These might correspond to concepts that are too rare to be captured by our finite-size SAEs, but are built out of composing a relatively common feature with a relatively common transform. In an attribution graph, decomposing SAE errors into (feature, transform) pairs in this fashion would manifest as having some graph edges that are mediated by chains of transforms in consecutive layers.

More work needs to be done to conclude that MOLTs are strictly preferable to transcoders. Attribution graphs that include transform information are somewhat more unwieldy than transcoder-based attribution graphs, and not all transform-mediated computations are clearly interpretable. Future work on scaling and improving MOLTs, and the associated attribution graph logic and UI, may address some of these issues.
