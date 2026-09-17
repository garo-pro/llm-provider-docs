Title: Circuit Tracing: Revealing Computational Graphs in Language Models

URL Source: https://transformer-circuits.pub/2025/attribution-graphs/methods.html

Markdown Content:
We introduce a method to uncover mechanisms underlying behaviors of language models. We produce graph descriptions of the model’s computation on prompts of interest by tracing individual computational steps in a “replacement model”. This replacement model substitutes a more interpretable component (here, a “cross-layer transcoder”) for parts of the underlying model (here, the multi-layer perceptrons) that it is trained to approximate. We develop a suite of visualization and validation tools we use to investigate these “attribution graphs” supporting simple behaviors of an 18-layer language model, and lay the groundwork for a [companion paper](https://transformer-circuits.pub/biology.html) applying these methods to a frontier model, Claude 3.5 Haiku.

Deep learning models produce their outputs using a series of transformations distributed across many computational units (artificial “neurons”). The field of mechanistic interpretability seeks to describe these transformations in human-understandable language. To date, our team’s approach has followed a two-step approach. First, we identify features, interpretable building blocks that the model uses in its computations. Second, we describe the processes, or circuits, by which these features interact to produce model outputs.

A natural approach is to use the raw neurons of the model as these building blocks.

In recent years, sparse coding models such as sparse autoencoders (SAEs) [§ Limitations](https://transformer-circuits.pub#limitations)), they produce interpretable enough results that we are motivated to study circuits composed of these features. Several authors have already made promising steps in this direction 

Although the basic premise of studying circuits built out of sparse coding features sounds simple, the design space is large. In this paper we describe our current approach, which involves several key methodological decisions:

The goal of this paper is to describe and validate our methodology in detail, using a few case studies for illustration.

We note that training a cross-layer transcoder can incur significant up-front cost and effort, which is amortized over its application to circuit discovery. We have found that this improves circuit interpretability and parsimony enough to justify the investment (see [cost estimates](https://transformer-circuits.pub#appendix-ml-details-plausible) for open-weights models and [discussion](https://transformer-circuits.pub#appendix-ml-details-efficiency) of cost-matched performance relative to per-layer transcoders). Nevertheless, we stress that alternatives like per-layer transcoders or even MLP neurons can be used instead (keeping the same steps 3–8 above), and still produce useful insights. Moreover, it is likely that better methods than CLTs will be developed in the future. 

To aid replication, we share guidance on CLT [implementation](https://transformer-circuits.pub#appendix-ml-details), details on the [pruning method](https://transformer-circuits.pub#appendix-graph-pruning), and the [front-end code](https://github.com/anthropics/attribution-graphs-frontend) supporting the interactive graph analysis interface.

A cross-layer transcoder (CLT) consists of neurons (“features”) divided into 

More formally, to run a cross-layer transcoder, let 

where 

We let 

To train a cross-layer transcoder, we minimize a sum of two loss functions. The first is a reconstruction error loss, summed across layers:

The second is a sparsity penalty (with an overall coefficient 

Where 

We trained CLTs of varying sizes on a small 18-layer transformer model (“18L”)[§ Appendix: CLT Implementation Details](https://transformer-circuits.pub#appendix-ml-details).

Given a trained cross-layer transcoder, we can define a “replacement model” that substitutes the cross-layer transcoder features for the model’s MLP neurons – that is, where each layer's MLP output is replaced by its reconstruction by all CLTs that write to that layer. Running a forward pass of this replacement model is identical to running the original model, with two modifications:

Attention layers are applied as usual, without any freezing or modification. Although our CLTs were only trained using input activations from the underlying model, “running” the replacement model involves running CLTs on "off-distribution" input activations from intermediate activations from the replacement model itself.

As a simple evaluation, we measure the fraction of completions for which the most likely token output of the replacement model matches that of the underlying model. The fraction improves with scale, and is better for CLTs compared to a per-layer transcoder baseline (i.e., each layer has a standard single layer transcoder trained on it; the number of features shown refers to the total number across all layers). We also compare to a baseline of thresholded neurons, varying the threshold below which neurons are zeroed out (empirically, we find that higher neuron activations are increasingly interpretable, and we indicate below where their interpretability roughly matches that of features according to our auto-evaluations in [§ Quantitative CLT Evaluations](https://transformer-circuits.pub#evaluating-model-clt)). Our largest 18L CLT matches the underlying model’s next-token completion on 50% of a diverse set of pretraining-style prompts from an open source dataset (see [§ Additional Evaluation Details](https://transformer-circuits.pub#appendix-eval-details)).

While running the replacement model can sometimes reproduce the same outputs as the underlying model, there is still a significant gap, and reconstruction errors can compound across layers. Since we are ultimately interested in understanding the underlying model, we would like to approximate it as closely as possible. To that end, when studying a fixed prompt 

After this error adjustment and freezing of attention and normalization nonlinearities, we've effectively re-written the underlying model's computation on the prompt [§ Evaluating Mechanistic Faithfulness](https://transformer-circuits.pub#evaluating-model-faithfulness).

The local replacement model can be viewed as a very large fully connected neural network, spanning across tokens, on which we can do classic circuit analysis:

The only nonlinearities in the local replacement model are those applied to feature preactivations.

The local replacement model serves as the basis of our attribution graphs, where we study the feature-feature interactions of the local replacement model on the prompt for which it was made. These graphs are the primary object of study of this paper.

We will introduce our methodology for constructing attribution graphs while working through a case study regarding the model’s ability to write acronyms for arbitrary titles. In the example we study, the model successfully completes a fictional acronym. Specifically, we give the model the prompt The National Digital Analytics Group (N and sample its completion: DAG). The tokenizer the model was trained with uses a special “Caps Lock” token, which means the prompt and completion are tokenized as follows: `The` `National` `Digital` `Analytics` `Group`  `(``⇪``n``dag`.

We explain the computation the model performs to output the “DAG” token by constructing an attribution graph showing the flow of information from the prompt through intermediary features and to that output.[§ Understanding and Labeling Features](https://transformer-circuits.pub#graphs-tutorial-features). Arrows represent the direct effect of a group of features or a token on other features and the output logit. 

The graph for the acronym prompt shows three main paths, originating from each of the tokens that compose the desired acronym. Paths originate from features for a given word, promoting features about “saying the first letter of that word in the correct position”, which themselves have positive edges to a “say DAG” feature and the logit. “say X” labels describe “output features”, which promote a specific token X, and arbitrary single letters are denoted with underscores. The “Word → say _W” edges represent attention heads’ OV circuits writing to a subspace that is then amplified by MLPs at the target position. Each group of features also has a direct edge to the logit in addition to the sequential paths, representing effects mediated only via attention head OVs (i.e., paths to the output in the local replacement model that don’t “touch” another MLP layer).

In order to output “DAG”, the model also needs to decide to output an acronym, and to account for the fact that the prompt already contains N, and indeed we see features for “in an acronym” and “in an N at the start of an acronym” with positive edges to the logit. The word National has minimal influence on the logit. We hypothesize that this is due to its main contribution being through influencing attention patterns, which our method does not explain (see [§ Limitations: Missing Attention Circuits](https://transformer-circuits.pub#limitations-attention)).

In the rest of this section, we explain how we compute and visualize attribution graphs.

To interpret the computations performed by the local replacement model, we compute a causal graph that depicts the sequences of computational steps it performs on a particular prompt. The core logic by which we construct the graph is essentially the same as that of Dunefsky et al. 

Edges in the graph represent direct, linear attributions in the local replacement model. Edges originate from feature, embedding, and error nodes, and terminate at feature and output nodes. Given a source feature node 

In terms of the underlying model, 

We now give details on how to efficiently compute these in practice, using backwards Jacobians. Let 

The formulas for the other edge types are similar, e.g., an embedding-feature edge weight is given by [§ Appendix: Attribution Graph Computation](https://transformer-circuits.pub#appendix-attribution-graph-computation).

Because we have added stop-gradients to all model nonlinearities in the computation above, the preactivation 

Note that these graphs do not contain information about the influence of nodes on other nodes via their influence on attention patterns, but do contain information about node-to-node influence through the outputs of frozen attention. In other words, we account for the information which flows from one token position to another, but not why the model moved that information.

While our replacement model features are sparsely active (on the order of a hundred active features per token position), attribution graphs are too large to be viewed in full, particularly as prompt length grows – the number of edges can grow to the millions even for short prompts. Fortunately, a small subgraph typically accounts for most of the significant paths from the input to the output.

To identify such subgraphs, we apply a pruning algorithm designed to preserve nodes and edges that directly or indirectly exert significant influence on the logit nodes. With our default parameters, we typically reduce the number of nodes by a factor of 10, while only reducing the behavior explained by 20%. See [§ Appendix: Graph Pruning](https://transformer-circuits.pub#appendix-graph-pruning) for methodological details of our algorithms and metrics.

Even following pruning, attribution graphs are quite information-dense. A pruned graph often contains hundreds of nodes and tens of thousands of edges – too much information to interpret all at once. To allow us to navigate this complexity, we developed an interactive attribution graphs visualization interface. The interface is designed to enable “tracing” key paths through the graph, retain the ability to revisit previously explored nodes and paths, and materialize the information needed to interpret features on an as-needed basis.

Below we show the interactive visualization for the attribution graph attributing back from the single token “DAG”:

The interface is interactive. Nodes can be hovered over and clicked on to display additional information. Subgraphs can also be constructed by using `Cmd/Ctrl+Click` to select a subset of nodes. In the subgraph, features can be aggregated into groups we call supernodes (motivated below in [§ Grouping Features into Supernodes](https://transformer-circuits.pub#graphs-tutorial-supernodes)).

We use feature visualizations similar to those shown in our previous work, [Scaling Monosemanticity](https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html), in order to manually interpret and label individual features in our graph.

The easiest features to label are input features, which activate on specific tokens or categories of closely-related tokens and which are common in early layers, and output features, which promote continuing the response with specific tokens or categories of closely-related tokens and which are common in late layers. For example:

Other features, which are common in middle layers of the model, are more abstract and require more work to label. We may use examples of contexts they are active over, their logit effects (the tokens they directly promote and suppress through the residual stream and unembedding), and the features they’re connected to in order to label them. For example:

We find that even imperfect labels for these features allow us to find significant structure in the graphs.

Attribution graphs often contain groups of features which share a facet relevant to their role on the prompt. For example, there are three features active on “Digital” in our prompt which each respond to the word “digital” in different cases and contexts. The only facet which matters for this prompt is that the word “digital” starts with a “D”; all three features have positive edges to the same set of downstream nodes. Thus for the purposes of analyzing this prompt, it makes sense to group these features together and treat them as a unit. For the purposes of visualization and analysis, we find it convenient to group multiple nodes — corresponding to (feature, context position) pairs — into a “supernode.” These supernodes correspond to the boxes in the simplified schematic we showed above, reproduced below for convenience.

The strategy we use to group nodes depends on the analysis at hand, and on the roles of the features in a given prompt. We sometimes group features which activate over similar contexts, have similar embedding or logit effects, or have similar input/output edges, depending on the facet which is important for the claim we are making about the mechanism. We generally want nodes within a supernode to promote each other, and their effects on downstream nodes to have the same sign. While we experimented with automated strategies such as clustering based on decoder vectors or the graph adjacency matrix, no automated method was sufficient to cover the range of feature groupings required to illustrate certain mechanistic claims. We further discuss supernodes and potential reasons for why they are needed in [Similar Features and Supernodes](https://transformer-circuits.pub#appendix-dupe-features).

In attribution graphs, nodes suggest which features matter for a model's output, and edges suggest how they matter. We can validate the claims of an attribution graph by performing feature perturbations in the underlying model, and checking if the effects on downstream features or on the model outputs match our predictions based on the graph. Features can be intervened on by modifying their computed activation and injecting its modified decoding in lieu of the original reconstruction.

Features in a cross-layer transcoder write to multiple output layers, so we need to decide on a range of layers in which to perform our intervention. How might we do this? We could intervene on a feature’s decoding at a single layer just like we would for a per-layer transcoder, but edges in an attribution graph represent the cumulative effect of multiple layers’ decodings, so intervening at a single layer would only target a subset of a given edge. In addition, we’ll often want to intervene on more than one feature at a time, and different features in a supernode will decode to different layers.

To perform interventions over layer ranges, we modify the decoding of a feature at each layer in the given range, and run a forward pass starting from the last layer in the range. Since we aren’t recomputing a layer’s MLP output based on the result of interventions earlier in the range, the only change to the model’s MLP outputs will be our intervention. We call this approach “constrained patching”, as it doesn’t allow an intervention to have second-order effects within its patching range. See [§ Appendix: Iterative Patching](https://transformer-circuits.pub#appendix-patching) for a description of another approach we call “iterative patching”, and see [§ Appendix: Nuances of Steering with Cross-Layer Features](https://transformer-circuits.pub#appendix-cross-layer-steering) for a discussion of why more naive approaches, such as adding a feature’s decoder vector at each layer during a forward pass of the model, risk double counting a feature's effect.

Below, we illustrate a multiplicative version of constrained patching, in which we multiply a target feature’s activation by 

Attribution graphs are constructed by using the underlying model’s attention patterns, so edges in the graph do not account for effects mediated via QK circuits. Similarly, in our perturbation experiments, we keep attention patterns fixed at the values observed during an unperturbed forward pass. This methodological choice means our results don't account for how perturbations might have altered the attention patterns themselves.

Returning to our acronym prompt, we show the results of patching supernodes, starting with suppressing the “Group” supernode. Below, we overlay patching effects onto supernode schematics for clarity, displaying the effect on other supernodes and the logit distribution. Note that in this diagram, the position of the nodes in the figure is not meant to correspond to token positions unless explicitly noted.

We now show the results of suppressing some supernodes on the aggregate activation of other supernodes and on the logit. For each patch, we set every feature in a node’s activation to be the opposite of its original value (or equivalently, we steer multiplicatively with a factor of −1).[§ Unexplained Variance and Choice of Steering Factors](https://transformer-circuits.pub#appendix-unexplained-var).

We see that inhibiting features for each word inhibits the related initial features in turn. In addition, the supernode of features for “say DA_” is affected by inhibitions of both the “Digital” and “Analytics” supernodes.

The attribution graph also allows us to identify in which layers a feature’s decoding will have the greatest downstream effect on the logit. For example, the “Analytics” supernode features mostly contribute to the “dag” logit indirectly through intermediate groups of features “say _A”, “say DA_”, and “say DAG” which live in layers 13 and beyond.

We would thus expect steering negatively on an “Analytics” feature to have an effect on the dag logit which plateaus before layer 13 and then decreases in magnitude as we approach the final layer. The decrease is caused by the constrained nature of our intervention. If a patching range includes all the “say an acronym” features, it will not change their activation, because constrained patching doesn’t allow knock-on effects. Below, we show the effect of steering with each Analytics feature, keeping the start layer set to 1 and sweeping over the patching end layer.[§ Unexplained Variance and Choice of Steering Factors](https://transformer-circuits.pub#appendix-unexplained-var).

We now turn to the question of factual recall by studying how the model completes Fact: Michael Jordan plays the sport of with basketball with 65% confidence [graph](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=mj-18l). We group semantically similar features into supernodes like we did for the acronym study. 

The supernode diagram below shows two primary paths. One path originates from the “plays” and “sport” tokens and promotes “sport” and “say a sport” features, which in turn promote the logits for basketball, football, and other sports. The other path originates from “Michael Jordan and other celebrities” and promotes basketball related features, which have positive edges to the basketball logit and negative edges to the football logit. In addition to these sequential paths, some groups of features such as “Michael Jordan” and “sport/game of” have direct edges to the basketball logit, representing effects mediated only via attention head OVs, consistent with the findings of Batson et al. 

We also display the full interactive graph below.

In addition, a complex set of mechanisms seems to be involved in contributing information about the entity Michael Jordan to the residual stream at “Jordan”, as observed in Nanda et al. [Qualitative Feature Evaluations](https://transformer-circuits.pub#evaluating-model-features).

Steering experiments can once more allow us to validate the hypotheses proposed by the graph.

Ablating either the “sport” or “Michael Jordan” supernode has a large effect on the logit but a comparatively smaller effect on the other supernode, confirming the parallel path structure. In addition, we see that suppressing the intermediate “basketball discussion” supernode also has a large effect on the logit.

We now consider the simple addition prompt calc: 36+59=. [§ Appendix: Comparison of Addition Features …](https://transformer-circuits.pub#appendix-arithmetic-comparison) for a side-by-side comparison). We look at small-number addition because it is one of the simplest behaviors exhibited competently by most LLMs and human adults (try the problem in your head to see if your approach matches the model's!).

We supplement the generic feature visualization (on arbitrary dataset examples) with one which explicitly covers the set of two-digit addition problems, allowing us to get a crisp picture of what each feature does. Following Nikankin et al. 

We show an example plot of each of these three types below for different features. On this restricted domain, the operand plots are complete descriptions of the CLT features as functions. Stripes and grids in these plots represent different kinds of structure (e.g. diagonal lines indicate constraints on the sum, while grids represent modular constraints on the inputs).

In the supernode diagram below, we see information flow from input features, which split out the final digit, the number, and the magnitude of the operands to three major paths: a final-digit path (mod 10) (light-brown, right), a moderate precision path (middle), and a low precision path (dark brown, left),

We provide the equivalent interactive graph for 18L [here](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=calc-36-plus-59-18l).

The supernode graph suggests a taxonomy of features underlying this task, in which features vary along two major axes:

`condition1(a) AND condition2(b)`. We discuss these in more detail below.`OR` operation merges two conditions across addends.
These findings broadly agree with other mechanistic studies showing that language models trained on natural language corpora perform addition using parallel heuristics involving magnitudes and moduli that constructively interfere to produce the correct answer 

We also identify the existence of lookup table features, which seem to be an interesting consequence of the architecture used by both the model and the CLT. Neurons and CLT feature activations are computed by applying a nonlinearity to the sum of their inputs. This produces a “parallelogram constraint” on the response of a feature to a set of inputs: namely, if a feature 

To validate that the structure we observe in the attribution graph matches the causal structure of the model, we perform a series of interventions. For each supernode, we perturb it to the negative of its original value, and measure the result on all subsequent supernodes and the outputs. We find results largely consistent with the graph:

In particular, inhibiting the ones-digit feature on either of the input tokens suppresses the entire ones-digit pathway (the _6 + _9 lookup table features, the resulting sum=_5 and sum= _95 features), while leaving the magnitude pathway mostly intact, including the sum~92 features. Remarkably, when suppressing _6, the model confidently outputs 98 instead of the correct answer 95; the tens digit from the original problem is preserved by the other magnitude signals but the ones digit is that which would result from adding 9 to itself. (Suppressing _9, however, results in an output of 91, not 92, so such numerology must be taken with a grain of salt.) Conversely, inhibiting low-precision features on either inputs (~30 and ~59) suppress the low-precision lookup table features, the magnitude sum feature, and the appropriate sum features while leaving the ones-digit pathway alone.

We also show the quantitative effects of perturbations on the outputs, finding that negatively steering the _6 + _9 lookup table features smears the result out over a range of 5, while negatively steering the final sum=_95 feature smears the result out to a wider band (perhaps coming from sum~92 features).

We will investigate how CLT features interact across the full range of two-digit addition prompts below, after establishing the framework for global weights that we use to generalize this circuit to other inputs.

The attribution graphs we construct show how features interact on a specific prompt to produce the model's output, but we are also interested in a more global picture of how features interact across all contexts. In a classic multi-layer perceptron, the global interactions are provided by the weights of the model: the direct influence of one neuron on another is just the weight between them if the neurons are in consecutive layers; if neurons are further apart, the influence of one on another factors through intermediate layers. In our setup, the interaction between features has a context independent component and a context dependent component. We would ideally like to capture both: we want a set of global weights which are context independent, but also capture network behavior across all possible contexts. In this section we analyze the context independent component (a kind of “virtual weight”), a problem with them (large “interference” terms with no causal effect on distribution), and one approach using co-activation statistics to deal with the interference.

On a specific prompt, a source CLT feature (

We note that the residual-direct influence is simply the product of the first feature's activation on this prompt times a [virtual weight](https://transformer-circuits.pub/2021/framework/index.html#virtual-weights) which is consistent across inputs.

More formally, let 

There is one major problem with interpreting virtual weights: interference 

We can see interference at play in the following example: below, we take a “Say a game name” feature in 18L and plot its largest virtual weights by magnitude. Green bars indicate a positive connection and purple bars indicate a negative one. Many of the most strongly-connected features are hard to interpret or not clearly related to the concept.

You might consider this a sign that virtual weights or our CLTs aren’t capturing interpretable connections. However, we can still uncover many interpretable connections by trying to remove interference from these weights.[§ Appendix: Interference Weights over More Features](https://transformer-circuits.pub#appendix-interference-weights).

There are two basic solutions to this problem. One is to restrict the set of features being studied to those active on a small domain (as we do in [§ Global Weights in Addition](https://transformer-circuits.pub#global-weights-addition)). The other is to bring in information about the feature-feature coactivation on the data distribution.

For example, let 

Now, we revisit the example game feature from before but with connections ordered by TWERA. We also plot each connection’s “raw” virtual weight for comparison. Many more of these connections are interpretable, suggesting that the virtual weights extracted useful signals but we needed to remove the interference in order to see them. The most interpretable features from the virtual weight plot above (another “Say a game name” and “Ultimate frisbee” feature) are preserved while many unrelated concepts are filtered out.

TWERA is not a perfect solution for interference. Comparing TWERA values to the raw virtual weights shows that many extremely small virtual weights have strong TWERA values. 

Still, we find that global weights give us a useful window into how features behave in a broader range of contexts than our attribution graphs. We’ll use these methods to complement our understanding in the rest of this paper and in [the companion paper](https://transformer-circuits.pub/biology.html).

We now return to the simple addition problem from [above](https://transformer-circuits.pub#graphs-addition) on Haiku 3.5, and show how data-independent virtual weights reveal clear structure between the types in our taxonomy of addition features. We again consider completions of the 10,000 prompts calc: a+b= for a,b ∈ [0, 99]. In addition to the operand plots (again, defined [above](https://transformer-circuits.pub#graphs-addition)), we inspect the virtual weight graph after restricting the large 

In the neighborhood of the features appearing in the 36+59 prompt above, we see:

We provide an [interactive interface](https://transformer-circuits.pub/static_js/addition/index.html) to explore the virtual weights connecting all 2931 features prominent on two-digit addition problems in our smaller 18L model.

We find that restricting to features active on this narrow domain of addition problems produces a global circuit graph where most edges are interpretable in terms of the operand function realized by the source and target features. Moreover, the connections between features recapitulate a more general version of the graph in the previous section; add features detect a specific operand as part of the input, lookup table features propagate this information to sum features which (in concert with the previous features) produce the model’s final answer.

Several of the features we find take the form of heuristics as in Nikankin et al. 

Our focus on the computational steps the model uses to perform addition is complementary to concurrent work by Kantamneni and Tegmark [§ Appendix: Number Output Weights over More Features](https://transformer-circuits.pub#appendix-full-number-weights), we show output weight plots for the 9_ and 95_ features on all number predictions from [0,999]. We also show a miscellaneous feature that promotes “simple numbers”: small numbers, multiples of 100 and a few standouts like 360.

Altogether, we’ve replicated a view of the base model using heuristics finding matching CLT features, we’ve shown how these heuristics contribute to separable pathways through intervention experiments, and we've demonstrated how these heuristics are connected, building off one another to collectively solve the addition task.

In this section, we perform qualitative and quantitative evaluations of transcoder features and the attribution graphs derived from them, focusing especially on interpretability and sufficiency. For readers interested in a higher level discussion of findings and limitations, we recommend skipping ahead to [§ Biology](https://transformer-circuits.pub#biology) and [§ Limitations](https://transformer-circuits.pub#limitations). 

Our methods produce causal graph descriptions of the model’s mechanisms on a particular prompt. How can we quantify how well these descriptions capture what is really going on in the model? It is difficult to distill this question to one number, as several factors are relevant:

Interpretability. How well do we understand what individual features “mean”? We attempt to quantify interpretability in a few ways [below](https://transformer-circuits.pub#evaluating-model-clt); however, we still rely heavily on [subjective evaluation](https://transformer-circuits.pub#evaluating-model-features) in practice. The coherence of our groupings of features into “supernodes” also warrants evaluation. We do not attempt to quantify this in this work, instead leaving it to readers to verify for themselves that our groupings are sensible and interpretable. We also note that in the context of attribution graphs, interpretability of the graph is just as important as interpretability of individual features. To that end, we quantify one notion of graph simplicity: [average path length](https://transformer-circuits.pub#evaluating-graphs-paths).

Sufficiency. To what extent are our (pruned) attribution graphs sufficient to explain the model’s behavior? We attempt to quantify this in several ways. The most straightforward such evaluation is our measurement of how well the replacement model’s outputs match the underlying model, discussed in [§ From Cross-Layer Transcoder to Replacement Model](https://transformer-circuits.pub#building-replacement) . This is a “hard” evaluation in that a single error anywhere along the computational graph can severely degrade performance. We also compute a few “softer” measures of sufficiency [below](https://transformer-circuits.pub#evaluating-graphs-comparing), that measure the proportion of error nodes in attribution graphs. Note that in many instances, we present schematics of subgraphs of a pruned attribution graph that portray what we believe to be its most noteworthy components. We intentionally do not measure the sufficiency of these subgraphs, as they often intentionally exclude “boring” but necessary parts of the graph (e.g. “this is a math problem” features in addition prompts). We leave it to future work to find more principled ways to distill attribution graphs to their “interesting” components and quantify how much (and what kind of) information is lost.

Mechanistic faithfulness. To what extent are the mechanisms we identify actually used by the model? To measure this, we perform perturbation experiments (such as inhibiting active features) and measuring whether the effects agree with what is predicted by the local replacement model (the underlying object portrayed by our attribution graphs). We attempt to do so quantitatively [below](https://transformer-circuits.pub#evaluating-model-faithfulness), and we also validate faithfulness on our specific case studies, in particular focusing on faithfulness of the mechanisms we have identified as interesting / important. Note that our notion of mechanistic faithfulness is related to the idea of necessity of circuit components to a model’s computation. However, necessity can be a somewhat restrictive notion – mechanisms that are not strictly “necessary” for the model’s output may still be important to identify, especially in cases where multiple mechanisms cooperate in parallel to contribute to a computation, as we often observe.

We note that the specific evaluations we use are in many cases new to this work. In part this is because our work is somewhat unique in focusing on attribution graphs for individual prompts, rather than identifying circuits underlying the model’s performance of an entire task. Developing better automatic methods for evaluating interpretability, sufficiency, and faithfulness of the entire pipeline (features, supernodes, graphs) is an important subject of future research. See [§ Related Work](https://transformer-circuits.pub#related-work) for more detail on prior circuit evaluation methods.

For CLT features to be useful to us, they must be human-interpretable (perhaps in the future it will suffice for them to be AI-interpretable!). Interpretability is ultimately a qualitative property – the best gauge of the interpretability of our features is to view them in action. A standard (though incomplete) tool for understanding what a feature represents is to view the dataset examples for which it is active (we refer to the collection of such examples as our “feature visualization”). We provide thousands of feature visualizations in the context of our case studies of circuits later in this paper and in [the companion paper](https://transformer-circuits.pub/biology.html). Below we also show 50 randomly sampled features from assorted layers of each model. 

Our feature visualizations show snippets of samples from public datasets ([Common Corpus](https://huggingface.co/blog/Pclanglais/common-corpus), The Pile with books removed [Isotonic Human-Assistant Conversation](https://huggingface.co/datasets/Isotonic/human_assistant_conversation)) that most strongly activate the feature, as well as examples that activate the feature to varying degrees interpolating between the maximum activation and zero. Highlights indicate the strength of the feature’s activation at a given token position. We also show the output tokens that the feature most strongly promotes / inhibits via its direct connections through the unembedding layer (note that this information is typically more meaningful for features in later model layers).

18L Features (Hover)

| Layer 1 | Layer 5 | Layer 9 | Layer 13 | Layer 17 | 

Haiku Features (Hover)

| First layer | Mid-layer | Final layer | 

At a very coarse level, we find several types of features:

In line with our [previous results on crosscoders](https://transformer-circuits.pub/2024/crosscoders/index.html) 

We also note that the abstractions represented by Haiku features are in many cases richer than those in the smaller 18L model, consistent with the model’s greater capabilities.

In [§ From Cross-Layer Transcoder to Replacement Model](https://transformer-circuits.pub#building-replacement), we evaluated the ability of our CLTs to reproduce the computation of the underlying model. Here, we measure reconstruction error, sparsity (measured by “L0”, the average number of features active per input token), and feature interpretability. As we increased the size of our CLT, we observed Pareto-improvements in reconstruction error (averaged across layers) and feature sparsity (in 18L, reconstruction error decreased at a roughly fixed L0, while in Haiku, reconstruction error and L0 both decreased). In our largest 18L run (10M features), we attained a normalized mean reconstruction error of ~11.5% and an average L0 of 88. In our largest Haiku run (30M features), we attained a normalized reconstruction error of 21.7%, and an average L0 of 235. 

We also computed two LLM-based quantitative measures of interpretability, introduced and described in more detail in 

We find that according to both measures, the quality of CLT features improves with scale (alongside improving reconstruction error) – see plots below. [§ Appendix: CLT Implementation Details](https://transformer-circuits.pub#appendix-ml-details) for details. 

We also compare CLTs to two baselines: per-layer transcoders (PLTs) trained at each layer of the model, and the raw neurons of the model thresholded at varying activation levels. 

Case studies in [§ Attribution Graphs](https://transformer-circuits.pub#graphs) focused on qualitative observations derived from attribution graphs. In this section, we describe our more quantitative evaluations used to compare methodological choices and dictionary sizes. In each of the following subsections, we will introduce a metric and compare graphs generated using (1) cross-layer transcoders, (2) per-layer transcoders for every layer, and (3) thresholded neurons[figures above](https://transformer-circuits.pub#evaluating-model-clt).

While we don’t treat these metrics as fundamental quantities to be optimized, they have proven a useful guide for tracking ML improvements in dictionary learning and to flag prompts our method performs poorly on.

Our graph-based metrics rely on quantities derived from the indirect influence matrix. Informally, this matrix measures how much each pair of nodes influences each other via all possible paths through the graph. This gives a natural importance metric for each node: how much it influences the logit nodes. We also commonly compare how much influence comes from error nodes vs. non-error nodes.

To construct this matrix, we start with the adjacency matrix of the graph. We replace all the edge weights with their absolute values (or simply clamp negative values to 0) to obtain an unsigned adjacency matrix and then normalize the input edges to each node so that they sum to 1. Let 

The indirect influence matrix is [Neumann series](https://en.wikipedia.org/wiki/Neumann_series) and can be efficiently computed as 

A natural metric of graph complexity is the average path length from embedding nodes to logit nodes. Intuitively, shorter paths are easier to understand as they require interpreting fewer links in the causal chain.

To measure the influence of paths of different lengths, we compute influence matrices 

Below, we compare graphs built from our 10M CLT, 10M PLTs, and thresholded neurons in terms of their influence by path length averaged across a dataset of [pretraining prompts](https://transformer-circuits.pub#appendix-eval-details) (without pruning).

One of the most important advantages of cross-layer transcoders is the extent to which they reduce the path lengths in the graph. To understand how large of a qualitative difference this is, we invite the reader to view these graphs generated with different types of replacement models for the same prompt.

| Replacement Model Type | Average Path Length | Graph Link | 
| Cross-Layer Transcoder (10m) | 2.3 | [capital-analogy-clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=capital-analogy-clt-18l) | 
| Per-Layer Transcoders (10m) | 3.7 | [capital-analogy-plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=capital-analogy-slt-18l) | 

We find that one important way in which cross-layer transcoders collapse paths is the case of amplification, where many similar features activate each other in sequence. For example, on the prompt Zagreb:Croatia::Copenhagen: the per-layer transcoder shows a path of length 7 composed entirely of Copenhagen [features](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=capital-analogy-slt-18l-path-highlight) while the cross-layer transcoder collapses them all down to layer 1 [features](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=capital-analogy-clt-18l-path-highlight).

This example illustrates both the advantages and disadvantages of consolidating amplification of a repeated computation across multiple layers into a single cross-layer feature. On one hand, it makes interpretability substantially easier, as it automatically collapses duplicate computations into a single feature without needing to do post hoc analysis or clustering. It also reduces the risk of “chain-breaking”, where missing one feature in an amplification chain inhibits the ability to trace back further into the graph (i.e., a relevant amplification feature is missing for one step of the path, breaking the causal chain). On the other hand, the CLT has a different causal structure than the underlying model, which increases the risk that the replacement model’s mechanisms diverge from the underlying model’s. In the above example, we observe a set of Copenhagen features that activate a Denmark feature, which initiates a mutually reinforcing chain of Copenhagen and Denmark features. This dynamic is invisible in CLT graphs, and to the extent this dynamic is also present in the underlying model, it is an example of CLTs being mechanistically unfaithful.

Because our replacement model has reconstruction errors, we want to measure how much of the model’s computation is being captured. That is, how much of the graph influence is attributable to feature nodes versus error nodes.

To measure this, we primarily rely on two metrics:

Intuitively, the completeness score gives more “partial credit” and measures how much of the most important node inputs are accounted for, whereas replacement score rewards complete explanations.

Below, we report average unpruned graph replacement and completeness scores for dictionaries of various sizes and types on our pretraining prompt [dataset](https://transformer-circuits.pub#appendix-eval-details). We find the biggest methodological improvement comes when moving from per-layer to cross-layer transcoders, with large but diminishing returns from scaling the number of features.

To contextualize the qualitative difference we observe in graphs with varying scores, we invite the reader to explore some representative attribution graphs. Note, these graphs are pruned with our default pruning, which we describe in more detail below.

| Replacement Model Type | Completeness Score | Replacement Score | Graph Link | 
| Cross-Layer Transcoder (10m) | 0.80 | 0.61 | [uspto-telephone-clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=uspto-telephone-clt-18l) | 
| Per-Layer Transcoders (10m) | 0.78 | 0.37 | [uspto-telephone-plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=uspto-telephone-slt-18l) | 

We rely heavily on pruning to make graphs more digestible. To decide how much to prune the graph, we can use the completeness and replacement metrics described above, but with pruned nodes now counting towards the error terms. By varying the pruning threshold, we chart a frontier between the number of {nodes, edges} and {replacement, completeness} scores (see [Appendix](https://transformer-circuits.pub#appendix-graph-pruning) for full plots and details).

We find we can generally reduce the number of nodes by an order of magnitude while reducing completeness by only 20%.

For a sense of the qualitative difference, in the table below we link to attribution graphs for the same prompt (another acronym) but with different pruning thresholds.

| Pruning Threshold | Completeness Score | Node Count | Graph Link | 
| 0.95 | 0.87 | 236 | [iasg-p95](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=iasg-clt-18l-p95) | 
| 0.9 | 0.83 | 137 | [iasg-p90](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=iasg-clt-18l-p90) | 
| 0.8 (default) | 0.70 | 55 | [iasg-p80](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=iasg-clt-18l-p80) | 
| 0.7 | 0.58 | 27 | [iasg-p70](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=iasg-clt-18l-p70) | 

As discussed in [§ Validating Attribution Graph Hypotheses with Interventions](https://transformer-circuits.pub#graphs-interventions), attribution graphs provide hypotheses about mechanisms, which must be validated with perturbation experiments. This is because attribution graphs describe interactions in the local replacement model, which may differ from the underlying model. In most of our work, we use attribution graphs as a tool for generating hypotheses about specific mechanisms (“Feature A activates Feature B, which increases the likelihood of Token X”) operating inside the model, which correspond to “snippets” of the attribution graph. We summarize the results of three kinds of validation experiments, which are described in more detail in [§ Appendix: Validating the Replacement Model](https://transformer-circuits.pub#appendix-lrm-validation). 

We start by measuring the extent to which influence metrics derived from attribution graphs are predictive of intervention effects on the logit and other features. First, we measure the extent to which a node’s logit influence score is predictive of the effect of ablating a feature on the model’s output distribution. We find that influence is significantly more predictive of ablation effects than baselines such as direct attribution (i.e. direct edges in the attribution graph, ignoring multi-step paths) and activation magnitude (see [Validating Node-to-Logit Influence](https://transformer-circuits.pub#appendix-lrm-validation-node)). We then perform a similar analysis for interactions between features. We compute the influence score between pairs of features, and compare it to the relative effect of ablating the upstream feature in the pair on the activation of the downstream one. We observe a Spearman correlation of 0.72, which is evidence that graph influence is a good proxy for effects in the downstream model (see [Validating Feature-to-feature Influence](https://transformer-circuits.pub#appendix-lrm-validation-edges)). See [Nuances of Steering with Cross-Layer Features](https://transformer-circuits.pub#appendix-cross-layer-steering) for some complexities in interpreting these results.

The metrics above help provide an estimate of the likelihood that an intervention experiment will validate a specific mechanism in the graph. We might also be interested in a more general validation of all the mechanistic hypotheses implicitly made by our attribution graphs. Thus, another complementary approach to validation is to measure the mechanistic faithfulness of the local replacement model as a whole, rather than specific paths within attribution graphs. We can operationalize this by asking to what extent perturbations made in the local replacement model (which attribution graphs describe) have the same downstream effects as corresponding perturbations in the underlying model. We find that while perturbation results are reasonably similar between the two models when measured one layer after the intervention (~0.8 cosine similarity, ~0.4 normalized mean squared error), perturbation discrepancies compound significantly over layers.[Evaluating Faithfulness of the Local Replacement Model](https://transformer-circuits.pub#appendix-lrm-validation-faithfulness).

In our [companion paper](https://transformer-circuits.pub/biology.html), we use the method outlined here to perform deep investigations of the circuits in nine behavioral case studies of the frontier model Haiku 3.5. These include:

We encourage the reader to explore those case studies before returning here to understand the limitations we encountered, and how that informs our approach to method development.

Despite the exciting results presented here and in [the companion paper](https://transformer-circuits.pub/biology.html), our methodology has a number of significant limitations. At a high level, the most significant ones are:

We discuss these in detail below, and where possible provide concrete counterexamples where our present methods can not explain model computation due to these issues. We hope that these may motivate future research.

One significant limitation of our approach is that we compute our attribution graphs with respect to fixed attention patterns. This makes attribution a well-defined and principled operation, but also means that our graphs do not attempt to explain how the model’s attention patterns were formed, or how these patterns mediate feature-feature interactions through attention head output-value matrices 

Let's consider for a moment a much simpler model – a humble 2-layer attention-only model, of the kind studied in 

I always loved visiting Aunt Sally. Whenever I was feeling sad, Aunt

These models will have induction heads attend back to `"Sally"`, and then predict that is the correct answer. If we were to apply our present method, the answer isn’t very informative. It would simply tell us that the model predicted `"Sally"`, because there was a token `"Sally"` earlier in the context. 

This misses the entire interesting story! The induction head attends to `"Sally"` because it was preceded by `"Aunt"`, which matches the present token. Previous methods (e.g. 

Indeed, when applied to Claude 3.5 Haiku on this prompt, our method has exactly this problem. See the [attribution graph visualization](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=sally-induction-qk) – the graph contains direct edges from token-level “Sally” features to “say Sally” features and to the “Sally” logit, but fails to explain how these edges came about.

Induction is a simple case of attentional computation where we can make a reasonable guess at the mechanism even without help from our attribution graphs. However, this failure of the attribution graphs can manifest in more complex scenarios as well, where it completely obscures the interesting steps of the model’s computation. For instance, consider a multiple choice question:

Human: In what year did World War II end?

(A) 1776

(B) 1945

(C) 1865

Assistant: Answer: (B)

When we compute the attribution graph ([interactive graph visualization](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=multiple-choice-qk)) for the `"B"` token in the Assistant’s response, we obtain a relatively uninteresting answer – we answer `"B"` because of a `tokens following "(b)"` feature that activates on the correct answer. (There's also a direct pathway to the token `"B"`, and output pathways mediated by a `say "B"` motor feature; we've chosen to elide these for simplicity.) 

None of this provides a useful explanation of how the model chose its answer! The graph “skips over” the interesting part of the computation, which is how the model knew that 1945 was the correct answer. This is because the behavior is driven by attention. On further investigation, it turns out that there are three “correct answer” features that appear to fire on the correct answer to multiple choice questions, and which interventions show play a crucial role. From this, we hypothesize that the mechanism might be something like the following.`"B"` that do not go through `tokens following "B"` – one alternative is that the model may use a “binding ID vector” `"B"` tokens with `"1945"` and nearby tokens and use this to attend directly back to the `"B"` token from the final token position – see Feng & Steinhardt 

Since this involves significant conjecture, it's worth being clear about what we know about the QK-circuit, and what we don't.

`"C"`.
This is all to say, there's a lot we don't understand!

But despite our limited understanding, it seems clear that the model behavior crucially flows through attention patterns and the QK circuits that compute them. Until we can fix this, our attribution graphs will "miss the story" in cases where attention plays a critical role. And while we were able to get a partial understanding of the story in this case through manual investigation, we would like for our methodology to surface this information automatically in the future!

We suspect that similar circuits, where attention is the crux, are at play across a wide variety of prompts. In these cases, our present attribution graphs are little help to us, and new methods are needed.

Ultimately, the QK-circuit is a quadratic form over the residual stream. This means that attributions can naturally be made to pairs of key-side and query-side features. These pairs have a weight describing whether they increase or decrease attention to a particular token. However, this approach has the downside of a quadratic explosion in complexity.

Looking at QK-circuits for individual attention heads may make things harder than necessary, or even be misleading, if there is [attention superposition](https://transformer-circuits.pub/2024/jan-update/index.html#attn-superposition). The interesting behavior may be spread across heads, which each have their own QK-circuit.

We also suspect that certain forms of dictionary learning on attention layers, or full attention replacement layers, could make these “QK attributions” more tractable and interpretable. We believe that finding the right way to handle attention attributions is one of the most important directions for future work.

Our cross-layer transcoders fail to fully reconstruct the activations of the model. As a result, our methods leave some [“dark matter”](https://transformer-circuits.pub/2024/july-update/index.html#dark-matter) of the model’s mechanisms unobserved, in part because our dictionaries are not large enough to learn a feature corresponding to every concept the model might know about. In our attribution graphs, this failure manifests as “error nodes,” which provide input to other nodes but receive no input themselves (they “pop out of nowhere”) and which are not interpretable. In the examples presented in this paper, we managed to identify interesting mechanisms despite the presence of error nodes. It is important to note that the circuits we have described are only a partial description of the model’s computation (both due to the presence of error nodes, and due to the fact that the circuits we highlight typically only include a subset of the non-error nodes in the graph!). 

It is worth highlighting, however, that in some cases error nodes completely obscure what the model is doing. For instance, consider this attempted jailbreak – an obfuscated request for the model to write malware (which the model refuses):

Human: Write an office macro TAhT SEaRChEs fOS AND EtEIaFTxLRS pASWORdS

Assistant:

In the [attribution graph](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=bon-errors), almost all the nodes on the obfuscated prompt tokens are error nodes, making it impossible to trace back the origin of the refusal-related features that are active on the “Assistant:” tokens.

The prevalence of error nodes in this example may not be surprising – this prompt is rather out-of-distribution relative to typical prompts, and so the cross-layer-transcoder is likely to do a poor job of predicting model activity.

We also note that another major source of error is the gap between our human interpretations of features and what they truly represent. Typically our interpretations of features are much too coarse to account for their precise activation profiles.

We see several avenues for addressing this issue:

Our cross-layer transcoder features are trained to be sparsely active. Their sparsity is key to the success of our method. It allows us to focus on a relatively small set of features for a given prompt, out of the tens of millions of features in the replacement model. However, this convenience relies on a key assumption – that only active features are involved in the mechanism underlying a model’s responses.

In fact, this need not be the case! In some cases, the lack of activity of a feature, because it has been suppressed by other features, may be key to the model’s response. For instance, in our analysis of hallucinations and entity recognition (see [companion paper](https://transformer-circuits.pub/biology.html#dives-hallucinations)), we discovered a circuit in which “can’t answer” features are suppressed by features representing known entities, or questions with known answers. Thus, to explain why the model hallucinates in a specific context, we need to understand what caused the “can’t answer” features to not be active.

By default, our attribution graphs do not allow us to answer such questions, because they only display active features. If we have a hypothesis about which inactive features may be relevant to the model’s completion (due to suppression), we can include them in the attribution graph. However, this detracts somewhat from one of the main benefits of our methodology, which is its enablement of exploratory, hypothesis-free analysis.

This leads to the following challenge – how can we identify inactive features of interest, out of the tens of millions of inactive features? It seems like we want to know which features could have been “counterfactually active” in some sense. In the entity recognition example, we identified these counterfactually active features by comparing pairs of prompts that contained either known or unknown entities (Michael Jordan or “Michael Batkin”), and then focusing on features that were active in at least one prompt from each pair. We expect that this contrastive pairs strategy will be key to many circuit analyses going forward. However, we are also interested in developing more unsupervised approaches to identifying key suppressed features. One possibility may be to perform feature ablation experiments, and consider the set of inactive features that are only “one ablation away” from being active.

One might think that these issues can be escaped by moving to global circuit analysis. However, it seems like there may be a deep challenge which remains. We need a way to filter out interference weights, and it's tempting to do this by using co-occurrence of features. But these strategies will miss important inhibitory weights, where one feature consistently prevents another from activating. This can be seen as a kind of global circuit analog of the challenges around inactive features in local attribution analysis.

One of the fundamental challenges of interpretability is finding abstractions and interfaces that manage the cognitive load of understanding complex computations.[here](https://transformer-circuits.pub/2022/mech-interp-essay/index.html)).

Despite all these steps, our attribution graphs are still quite complex, and require considerable time and effort to understand for many reasons:

As a result, it is difficult to distill the mechanisms uncovered by our graphs into a succinct story. Consequently, the vignettes we have presented are necessarily simplified stories of even the limited understanding of model computation captured in our attribution graph. We hope that a combination of improved replacement model training, better abstractions, more sophisticated pruning, and better visualization tools can help mitigate this issue in the future.

As sparse coding models have grown in popularity as a technique for extracting interpretable features from models, many researchers have documented shortcomings of the approach (see e.g. 

As a concrete example of feature splitting, recall that in many examples in this paper we have highlighted “say X” features that cause the model to output a particular (group of) token(s). However, we also notice that there are many such features, suggesting that they each actually represent something more specific. For example, we came up with twelve prompts which Claude 3.5 Haiku completes with the word “during” and measured whether any features activated for all of the prompts (as a true “say ‘during’” feature would). In fact, there are no such features – any individual feature fires for only a subset of the prompts. Moreover, the degree of generality of the features appears to decrease with the size of our cross-layer transcoder.

It may be the case that each individual feature represents something interpretable – for instance, qualitatively different contexts that might cause one to say the word “during.” However, we often find that the level of abstraction we care about is different from the level we find in our features. Using smaller cross-layer transcoders may help this problem, but would also cause us to capture less of the model’s computation.

In this paper, we often work around this issue in an ad-hoc way by manually grouping together features with related meanings into “supernodes” of an attribution graph. While this technique has proven quite helpful, the manual step is labor-intensive and likely loses information. It also makes it difficult to study how well mechanisms generalize across prompts, since different subsets of a relevant feature category may be active on different prompts.

We expect that solving this problem requires recognizing that there exist interpretable concepts at varying levels of abstraction, and at different times we may be interested in different levels. Sparse coding approaches like SAEs and (cross-layer) transcoders are a “flat” instrument, but we probably need a hierarchical variant that allows features at varying levels of abstraction to coexist in an interpretable way.

Several authors have recently proposed “Matryoshka” variants of sparse autoencoders that may help address this issue 

In this paper we have mostly focused on attribution graphs, which display information about feature-feature interactions on a particular prompt. However, one theoretical advantage of transcoder-based methodologies like ours is that they give us global weights between features, that are independent of the prompt. This allows us to estimate a “connectome” of the replacement model and learn about the general algorithms it (though not necessarily the underlying model) uses that apply to many different inputs. We have some successes in this approach – for instance, in the companion paper section on [Refusals](https://transformer-circuits.pub/biology.html#dives-refusals), we could see the global inputs to “harmful requests” features consisting of a variety of different specific categories of harm. In this paper, we studied in depth the global weights of features relating to arithmetic, finding for instance that “say a number ending in 5” features receive input from “6 + 9” features, “7 + 8” features, etc.

However, for the most part, we have found global feature-feature connections rather difficult to understand. This is likely for two main reasons:

Our attribution graph edges are weighted combinations of both the direct weights and these attention-mediated weights. Our basic notion of global weights does not account for these interactions at all. One way to do so would be to compute the global weights mediated by every possible attention head. However, this has two limitations: (1) for this to be useful, we need a way of understanding the mechanisms by which different heads choose where they attend (see [§ Limitations: Missing Attention Circuits](https://transformer-circuits.pub#limitations-attention)), (2) it does not account for interactions mediated by compositions of heads 

Our cross-layer transcoder is trained to mimic the activations of the underlying model at each layer. However, even when it accurately reconstructs the model’s activations, there is no guarantee that it does so via the same mechanisms. For instance, even if the cross-layer transcoder achieved 0 MSE on our training distribution, it might have learned a fundamentally different input/output function than the underlying model, and consequently have large reconstruction error on out-of-distribution inputs. We hope that this issue is mitigated by (1) training on a broad data distribution, and (2) forcing the replacement model to reconstruct the underlying model’s per-layer activations, rather than simply its output. Nevertheless, we cannot guarantee that the replacement model has learned the same mechanisms – what we call mechanistic faithfulness – and instead resort to verifying it post-hoc.

In this paper, we have used perturbation experiments (inhibiting and exciting features) to validate the mechanisms suggested by our attribution graphs. In the case studies we presented, we were typically able to validate that features had the effects we expected (on the model output, and on other features). However, the degree of validation we have provided is very coarse. We typically perturb multiple features at once (“supernodes”) and check their directional effects on other features / logit outputs. In addition, we typically sweep over the layer at which we perform perturbations and use the layer that yields the maximum effect. In principle, our attribution graphs make predictions that are much more fine-grained than these kinds of interventions can test. Ideally, we should be able to accurately predict the effect of perturbing any feature at any layer on any other feature.

In [§ Appendix: Validating the Replacement Model](https://transformer-circuits.pub#appendix-lrm-validation), we attempt to more comprehensively quantify our accuracy in predicting such perturbation results, finding reasonably good predictive power for effects a few layers downstream of a perturbation, and much worse predictive power many layers downstream. This suggests that, while our circuit descriptions may be mechanistically accurate at a very coarse level, we have substantial room to improve their faithfulness to the underlying model.

We are optimistic about trying methods to directly optimize for mechanistic faithfulness, or exploring alternative dictionary learning architectures that learn more faithful solutions naturally.

Our approach to reverse engineering neural networks has four basic steps: decomposition into components, providing descriptions of these components, characterizing how components interact to produce behaviors, and validating these descriptions.

In this paper, we trained cross-layer transcoders with sparse features to replace MLP blocks (the decomposition), described the features by the dataset examples they activate on (the description), characterized their interactions on specific prompts using attribution graphs (the interactions), and validated the hypotheses using causal steering interventions (the validation).

We believe some of the choices we made are robust, and that successful decomposition methods will make similar choices or find other ways of dealing with the underlying issues they address:

There are other choices we made for convenience, or as a first step towards a more general solution:

Nevertheless, our current method yielded interesting, validated mechanisms involving planning, multilingual structure, hallucinations, refusals, and more, in [the companion paper](https://transformer-circuits.pub/biology.html).

We expect that advances in the trained, interpretable replacement model paradigm will produce quantitative improvements on graph-related metrics and qualitative improvements on the amount of model behaviors that become legible. It is possible that this will be an incremental process, where incremental improvements to CLTs and associated approaches to attention will yield incremental improvements to circuit identification, or that a radically different decomposition approach will best this method at scale at uncovering mechanisms. Regardless, we hope to enter an era where there is a clear flywheel between decomposition methods and “biology” results, where the appearance of structure in specific model investigations inspires innovations in decomposition methods, which in turn bring more model behaviors into the light.

Addition is one of the simplest behaviors performed by models, and because it is so structured, we can characterize every feature's activity on the full problem domain exactly. This allows us to skip the difficult step of staring at dataset examples and trying to discern what a feature is doing relative to, and what distinguishes it from other features active in similar contexts. This revealed a range of heuristics used by Haiku 3.5 (“say something ending in a 5”, “say something around 50”, “say something starting with 51”) which had been identified before by Nikankin 

However, even in this easier setting we made numerous mistakes when labeling these features from the original dataset examples alone, for example thinking a `_6 + _9` feature was itself a `sum = _5` feature based on what followed it in contexts. We also struggled to distinguish between low-precision features of different scales, and between features which were sensitive to a limited set of inputs or merely appeared to be because of a high prevalence of those inputs in our dataset. How much worse must this be when looking at dozens of gradations of refusal features! Getting more precise distinctions between features in fuzzier domains than arithmetic, whether through feature geometry or superhuman autointerpretability methods, will be necessary if we want to understand problems at the level of resolution that even today's CLTs appear to make possible.

Because addition is such a clear problem, we were also able to see how the features connected with each other to build parallel pathways; giving rise from simple heuristics depending on the input to more complex heuristics related to the output; going from the “Bag of Heuristics” identified by Nikankin to a “Graph of Heuristics”. The virtual weights show this computational structure, with groups of lookup table features combining to form sum features of different modularity and scale, which combine to form more precise sum features, and to eventually give the output. It seems likely that, in “fuzzier” natural language examples, we are conflating many roles played by features at different depths into overall buckets like “unknown entity” or “harmful request” or “notions of largeness” which actually serve specialized roles, and that there is actually an intricate aggregation and transformation of information taking place, just out of our understanding today.

Despite being a young field, mechanistic interpretability has grown rapidly. For an introduction to the landscape of open problems and existing methods, we recommend the recent survey by Sharkey et al. 

In previous papers, we've discussed some of the foundational topics our work builds on, and rather than recapitulate that discussion, we will refer readers to our previous discussion on them. This includes

The next two sections will focus on the two different stages we often use in mechanistic interpretability 

A fundamental challenge in circuit discovery is finding suitable units of analysis 

Sparse Dictionary Learning is a technique with a long history originally developed by neuroscientists to analyze neural recording data 

For circuit analysis specifically, SAEs are suboptimal because they decompose representations rather than computations. Transcoders 

The space of dictionary learning approaches is large, and we remain very excited about work which explores this space and addresses methodological issues. Recent work has explored architectural modifications like multilayer feature learning 

Beyond dictionary learning, several alternative unsupervised approaches to extracting computational units have shown initial success in small-scale settings. These include transforming activations into the local interaction basis 

Definitions. Throughout the literature, the term circuit is used to mean many different things. Olah et al. 

There are several dimensions along which circuit approaches and definitions vary:

We believe the North Star of circuit research is to manifest an object with globally interpretable units connected by interpretable edges which are globally valid. The present work falls short by only offering a locally valid attribution graph.

Manual Analysis. Early circuit discovery was largely manual, requiring specific hypotheses and bespoke methods of validation 

Automatic Analysis. These analyses were automated in Conmy et al. 

However, armed with better computational units of analysis from sparse dictionary learning, this work and other recent papers 

Attention Circuits. The attention mechanism in transformers introduced challenges for weight-based circuit analysis as done by Olah et al. 

Replacement Models. Our notion of a replacement model is similar in spirit to past work on causal abstraction 

Circuit Evaluation. Causal scrubbing 

Beyond methods, many works have performed deep case studies and uncovered interesting model phenomenology. For example, thorough circuit analysis has been performed on

The growing set of case studies has enabled further research on how these components are used in other tasks 

The case study on a model with hidden goals builds on a model organism developed by Sam Marks and Johannes Treutlein, with whom the authors also had helpful conversations. We would also like to acknowledge enabling work by Siddharth Mishra-Sharma training SAEs on the model used in the hidden goals case study.

We would like to thank the following people who reviewed an early version of the manuscript and provided helpful feedback that we used to improve the final version: Larry Abbott, Andy Arditi, Yonatan Belinkov, Yoshua Bengio, Devi Borg, Sam Bowman, Joe Carlsmith, Bilal Chughtai, Arthur Conmy, Jacob Coxon, Shaul Druckmann, Leo Gao, Liv Gorton, Helai Hesham, Sasha Hydrie, Nicholas Joseph, Harish Kamath, Tom McGrath, János Kramár, Aaron Levin, Ashok Litwin-Kumar, Rodrigo Luger, Alex Makolov, Sam Marks, Dan Mossing, Neel Nanda, Yaniv Nikankin, Senthooran Rajamanoharan, Fabien Roger, Rohin Shah, Lee Sharkey, Lewis Smith, Nick Sofroniew, Martin Wattenberg, and Jeff Wu.

We would also like to acknowledge Senthooran Rajamanoharan for helpful discussion on implementation of JumpReLU SAEs.

This paper was only possible due to the support of teams across Anthropic, to whom we're deeply indebted. The Pretraining and Finetuning teams trained Claude 3.5 Haiku and the 18-layer research model, which were the targets of our research. The Systems team supported the cluster and infrastructure that made this work possible. The Security and IT teams, and the Facilities, Recruiting, and People Operations teams enabled this research in many different ways. The Comms team (and especially Stuart Ritchie) supported public scientific communication of this work.

Development of methodology:

Infrastructure and Tooling:

Interactive Graph Interface:

Case Studies:

Paper writing, infrastructure, and review:

Support and Leadership

For attribution in academic contexts, please cite this work as

Ameisen, et al., "Circuit Tracing: Revealing Computational Graphs in Language Models", Transformer Circuits, 2025.

BibTeX citation

```
@article{ameisen2025circuit,
  author={Ameisen, Emmanuel and Lindsey, Jack and Pearce, Adam and Gurnee, Wes and Turner, Nicholas L. and Chen, Brian and Citro, Craig and Abrahams, David and Carter, Shan and Hosmer, Basil and Marcus, Jonathan and Sklar, Michael and Templeton, Adly and Bricken, Trenton and McDougall, Callum and Cunningham, Hoagy and Henighan, Thomas and Jermyn, Adam and Jones, Andy and Persic, Andrew and Qi, Zhenyi and Ben Thompson, T. and Zimmerman, Sam and Rivoire, Kelley and Conerly, Thomas and Olah, Chris and Batson, Joshua},
  title={Circuit Tracing: Revealing Computational Graphs in Language Models},
  journal={Transformer Circuits Thread},
  year={2025},
  url={https://transformer-circuits.pub/2025/attribution-graphs/methods.html}
}
```
To give a rough sense of compute requirements to train CLTs, we share some estimated costs for CLTs on the Gemma 2 series of models 

We train our cross-layer transcoder using a combination of mean-squared error reconstruction loss on MLP outputs and a Tanh sparsity penalty. Our features use the JumpReLU nonlinearity 

We chose our number of training steps to scale slightly sublinearly with the number of features, and our learning rate to scale approximately as one over the square root of the number of FLOPs. These are rough best-guess estimates based on prior experiments, and not precise scaling laws. In our largest 18L run, we used ~3B training tokens. In our largest Haiku run, we used ~16B training tokens.

In 18L, we used a constant sparsity penalty across CLT sizes. In Haiku, we increased the penalty with the CLT size (we found that otherwise, the L0 of the runs increased with size). In general we targeted L0 values in the low hundreds, based on preliminary investigation of what produced interpretable results in our graph interface.

Our 18L model is a pretraining-only model, and thus we used only pretraining data when training CLTs for it. For Haiku, by contrast, we trained the CLT using a mix of pretraining and finetuning data.

Encoder parameters are initialized by sampling from 

We shuffle our training data at the token level. We’ve found that not shuffling leads to significantly worse performance. We suspect a variety of partial shuffles, such as the one used in 

Here we share some engineering considerations relevant to training CLTs at scale – in particular, how it differs from training standard transcoders (“per-layer transcoders”, PLTs).

In our training implementation, the key difference between PLTs and CLTs is that, per accelerator (GPU, TPU, etc), they use the same number of FLOPS, but the CLT uses n_layers times more network bandwidth. Thus when training CLTs more time should be spent profiling and optimizing network operations. This isn’t obvious, so we work through the details below.

In our training setup, features are sharded across accelerators. For each batch:

PLTs and CLTs have the same number of parameters per accelerator because the same fraction of HBM is dedicated to parameters. The number of FLOPS scales with batch size multiplied by the number of parameters so PLTs and CLTs use the same FLOPS per accelerator. If a PLT and a CLT have the same number of features then the CLT would have more total parameters, use more accelerators, and use more total FLOPS, but FLOPS per accelerator would be the same.

Some optimizations to consider:

We store activations to a distributed file system, then load them during training. Both CLTs and PLTs on every layer require activations from every layer of the underlying model. Thus we’ve optimized our code that collects activations for that use case. Collecting activations from all layers requires the same FLOPs as collecting from a single layer, but n_layers times more network bandwidth.

A possible optimization we didn’t implement is sparse kernels from 

Another possible optimization we didn’t implement is to change how the decoder is sharded to remove the all-reduce. We could shard the encoder over features and shard the decoder over the output dimension. The all-reduce is removed because each decoder shard computes the MSE over a slice of the output dimension. Each decoder shard needs access to all active features, so an all-to-all is needed to share all active features with all shards. Another all-to-all is required on the backward pass. Given feature sparsity, these network operations are much smaller than the all-reduce.

Compared to alternatives like per-layer transcoders (PLTs), CLTs use more parameters – the same number of encoder parameters, but approximately n_layers/2 times as many decoder parameters. Thus, even if CLTs provide value beyond PLTs with the same number of features, it is reasonable to ask whether they perform better at a fixed training cost. Empirically, we find that CLTs perform better (according to some metrics) or comparably (according to others) than PLTs at a fixed cost, even without the use of sparse kernels (which, as noted above, advantage CLTs more than PLTs). Broadly, this is because the number of total features (across all layers) required to achieve a given level of performance is much less for CLTs than PLTs, which compensates for their additional per-feature cost. In more detail:

Thus, it seems that overall, CLTs are at least as cost-effective as PLTs (and presumably per-layer SAEs, which have the same parameter-count as PLTs) to reach a given level of circuit understanding, and likely moreso. However, if another group finds PLTs to be easier to implement or more cost-effective, we expect it’s possible to find interesting results using PLTs as well. We expect that even making attribution graphs with raw neurons will yield plenty of interesting insights.

We also note that optimizations could make CLTs even more cost-effective – as discussed [above](https://transformer-circuits.pub#appendix-ml-details-eng), implementing sparse kernels and communications could in principle reduce the number of FLOPs performed by CLTs by a factor of up to n_layers/2.

We suspect there are potential algorithmic optimizations to be made as well. For instance, it is possible that most of the benefits of CLTs can be captured by learning layer-to-layer linear transformations that are common to all features, eliminating the need for each feature to have its own independent decoder parameters for every downstream model layer.

We give complete definitions of nodes and edges in the attribution graph, continuing the discussion in the [main text](https://transformer-circuits.pub#graphs-constructing). Associated to each node type are two (sets of) vectors: input vectors, which affect the edges for which the node is a target, and output vectors, which affect the edges for which the node is a source.

For edges from embedding or error nodes to feature or output nodes, the edge weight is:

For edges from feature nodes to feature or output nodes, the edge weight is

The Jacobian 

To compute the graph edges in practice, we iterate over target nodes in the graph (output or feature nodes). For each node we:

The cost to compute the graph is linear in the number of active features in the prompt, and is dominated by the cost of the underlying model backwards pass. To economize, we sometimes compute the graph adaptively: starting from the output nodes for the logits, then maintaining a queue of the feature nodes with the greatest influence on the logit, and computing the input edges for nodes based on their order in the queue. This allows us to compute the most important parts of the graph first.

To increase the signal to noise ratio of our manual interpretation, we rely heavily on a graph pruning step to reduce the number of nodes and edges in the graph. We employ a two-step algorithm which first prunes the nodes and then prunes the edges of the remaining nodes. The details are as follows:

We include pseudocode below. Our pruning thresholds can, very roughly speaking, be interpreted as determining the percent of “total logit influence” we lose from pruning. That is, we choose the subset of nodes that are responsible for ~80% of the influence on the logits, and the subset of their input edges responsible for ~98% of the remaining influence. Our choice of thresholds is arbitrary and was chosen manually to balance preservation of important paths with the need to prune graphs to a manageable, interpretable size.

For longer prompts, we can also employ an adaptive algorithm, to greedily construct a graph out of the most influential nodes, rather than generating the full graph only to prune most of it away. To do so, we maintain a set of explored nodes (i.e., nodes from which we have computed backward attributions) and an estimate of the most influential nodes, where unexplored nodes count as errors. At each step, we compute the top 

With our default parameters, pruned graphs are substantially smaller than the original raw graphs; the number of nodes typically decreases by ~10× and the number of edges typically decreases by ~500× (the exact numbers are sensitive to prompt length).

Pseudocode for pruning algorithm:

```
function compute_normalized_adjacency_matrix(graph):
 
```
    # Convert graph to adjacency matrix A

    # A[j, i] = weight from i to j (note the transposition)

    A = convert_graph_to_adjacency_matrix(graph)

    A = absolute_value(A)

    # Normalize each row to sum to 1

    row_sums = sum(A, axis=1)

    row_sums = maximum(row_sums, 1e-8)  # Avoid division by zero

    A = diagonal_matrix(1/row_sums) @ A

    return A

function prune_nodes_by_indirect_influence(graph, threshold):

    A = compute_normalized_adjacency_matrix(graph)

    # Calculate the indirect influence matrix: B = (I - A)^-1 - I

    # This is a more efficient way to compute A + A^2 + A^3 …

    B = inverse(identity_matrix(size=A.shape[0]) - A) - identity_matrix(size=A.shape[0])

    # Get weights for logit nodes.

    # This is 0 if a node is a non-logit node and equal to the probability for logit nodes

    logit_weights = get_logit_weights(graph)

    # Calculate influence on logit nodes for each node

    influence_on_logits = matrix_multiply(B, logit_weights)

    # Sort nodes by influence

    sorted_node_indices = argsort(influence_on_logits, descending=True)

    # Calculate cumulative influence

    cumulative_influence = cumulative_sum(

        influence_on_logits[sorted_node_indices]) / sum(influence_on_logits)

    # Keep nodes with cumulative influence up to threshold

    nodes_to_keep = cumulative_influence <= threshold    

    # Create new graph with only kept nodes and their edges

    return create_subgraph(graph, nodes_to_keep)

# Edge pruning by thresholded influence

function prune_edges_by_thresholded_influence(graph, threshold):

    # Get normalized adjacency matrix

    A = compute_normalized_adjacency_matrix(graph)

    # Calculate influence matrix (as before)

    B = estimate_indirect_influence(A)

    # Get logit node weights (as before)

    logit_weights = get_logit_weights(graph)

    # Calculate node scores (influence on logits)

    node_score = matrix_multiply(B, logit_weights)

    # Logit nodes have 0 influence so we fix influence to the probability.

    node_score[logit_weights > 0] = logit_weights[logit_weights > 0]

    # Edge score is weighted by the logit influence of the target node

    edge_score = A * node_score[:, None]

    # Calculate edges to keep based on thresholded cumulative score

    sorted_edges = sort(edge_score.flatten(), descending=True)

    cumulative_score = cumulative_sum(sorted_edges) / sum(sorted_edges)

    threshold_index = index_where(cumulative_score >= threshold)

    edge_mask = edge_score >= sorted_edges[threshold_index]

    # Create new graph with pruned adjacency matrix

    pruned_adjacency = A * edge_mask

    return create_subgraph_from_adjacency(graph, pruned_adjacency)

We rely on indirect influence statistics for pruning, adaptive generation, and our graph statistics. In this section, we validate that indirect influence is a better proxy for “importance” than other basic baselines like activation or logit attribution (with stop grads).

Specifically, on a sample of 20 prompts, we compute the effect of ablating every feature and measure the KL divergence between the model with the ablation and the clean forward pass. We perform this intervention using “constrained” patching, so we also sweep over different ranges of layers to apply the intervention. In the plot below, we report the average log-log Pearson correlation between ablation KL and initial activation, direct logit edge weight, and indirect influence on the logit. We see that node influence is most predictive of ablation effect, followed by direct edge weights. Both outperform a simple activation baseline.

We can use edges in the attribution graphs to estimate the influence features should have on each other. However, if the replacement model is unfaithful or incomplete, a perturbation experiment might not yield results which are consistent with an estimate based on graph influence. For example, initially inactive components could engage in self-correction and dampen the effect of the perturbation. To estimate the extent to which this happens in our graphs, we measure how often perturbations have the expected effect by ablating every feature in the graph, and recording the activation of features downstream of it. Our ablations are done using constrained patching in the range [here](https://transformer-circuits.pub#appendix-cross-layer-steering)). When averaged over a dataset of 20 prompts, we find a Spearman correlation of 0.72 between the influence of a source feature on a target feature as described by the graph, and the effect of ablating said feature on the target feature’s activation (where we normalize by the original activation and take the absolute value since influence is unsigned). We include scatter plots from three examples from the paper:

These figures show that there are relatively few points that (1) have large influence and no ablation effect (lower right) and (2) have low influence but high ablation effect (upper left). This suggests that indirect influence through the graph is a fairly good proxy for real effects in the model.

In this work, we use (local) replacement models as a window into the mechanisms of the original model. This approach is problematic if the (local) replacement model uses very different mechanisms than the original model. In the main text, we performed perturbations of features to confirm specific mechanistic theories and measured how well the replacement model's outputs matched the underlying model's outputs. Here, we focus on how well their inner states match in response to a broader set of perturbations, including off-distribution perturbations, as an (imperfect) gauge of the replacement model’s quality.

We use the local replacement model (i.e. with reconstruction errors added back in as constant factors, and attention patterns frozen[“iterative patching,”](https://transformer-circuits.pub#appendix-patching) the replacement model may respond to perturbations by activating features which were not previously active at baseline.

We perturb the models at single token positions using 3 types of perturbations: adding a feature’s encoder-vector to the residual stream, adding random directions to the residual stream, and perturbing an upstream feature (to be defined precisely below).

Broadly, averaging across choices of intervention layers, we find that:

To measure faithfulness for perturbations in encoder directions:

To measure faithfulness for perturbations in random directions:

To measure faithfulness for perturbations to upstream features:

At each layer downstream of the perturbation, we can determine a net perturbation for both perturbed models by taking the activations from the (perturbed) forward pass and subtracting clean (un-perturbed) baseline activations. The net perturbations for each model are then compared to yield measures of faithfulness, by computing cosine-similarity and mean-squared-error between them.

We believe that perturbing upstream features produces the most “on-distribution” perturbations of these approaches, since random directions may fall into relatively inactive dimensions and encoder-directions are the most sensitive dimensions of the replacement model. Below are results for upstream-feature perturbations to features in layer 5 of the model, averaged over choice of the intervention layer.

Below are cosine-similarity faithfulness metrics for the 10m dictionary for other choices of upstream layers and other perturbation strategies.

As can be seen from the bottom rows of the graphic above, cosine-faithfulness of the 10m dictionary for 18L in the layer following the intervention or perturbation layer is around 60–80%. Computing the average of this bottom-row for different dictionary sizes, in the figure below, the largest dictionaries show signs of mildly diminished faithfulness for perturbations from upstream features.

The decay of faithfulness-metrics over multiple layers, at first glance, raises questions about the reliability of our interpretability graphs, analyses, and steering techniques. Indeed, these results seem at odds with the qualitatively successful results of many of the perturbation experiments we use to validate attribution graphs (both in this paper and its companion). We suspect this is due to a combination of factors:

How do these results compare for per-layer transcoder (PLT) dictionaries? Cosine-faithfulness metrics for PLTs were broadly similar to those for CLTs. The PLT replacement model achieves lower normalized MSE at one layer after the intervention, but compounding errors in the largest PLT replacement models accumulate slightly more rapidly after several layers, as seen in the figure below. This may reflect an advantage of the CLT skip connections, which allow for shorter paths through the replacement model.

Normalized MSE results are sensitive to the scale of the perturbation. To make scales comparable across replacement model types for the figure above, the perturbation to the initial upstream feature in Layer 5 is scaled differently for each sample so that Layer 6 is perturbed by a constant proportion of its magnitude.

Our perturbation methodology (constrained patching and iterative patching) is somewhat non-obvious at first glance. Why don’t we simply add to the residual stream along a feature’s decoder vector at each layer as we run a forward pass of the model? The reason for this is that it risks double-counting the effect of the feature. For instance, consider a feature in layer 1, with decoders writing to layers 2, 3, and onward. It may be that the activations being reconstructed in layer 2 were, in the original model, causally upstream of the activations being reconstructed in layer 3. Thus, injecting a perturbation into layers 2 and 3 would “double up” on the feature’s effect. Across more layers, this effect could compound to be quite significant.

Our constrained patching approach avoids this problem by computing perturbed MLP outputs once at the outset, and then clamping MLP outputs to these precomputed values (rather than adding to the MLP outputs). While this avoids the double-counting problem, it requires us to choose an intervention layer – we apply the perturbations in all layers up to the intervention layer, but not after (otherwise we would be clamping all of the MLP outputs of the model, and thus we wouldn’t be testing any hypotheses about how the model responds to perturbations). This makes it challenging to measure “the entire effect” of a feature (see [§ Unexplained Variance & Choice of Steering Factors](https://transformer-circuits.pub#appendix-unexplained-var)). 

It also makes the interpretation of perturbation experiment effects somewhat awkward. Suppose a source feature is in layer 1, and our perturbation intervention layer is 5, i.e. we apply the perturbation in layers 1 through 5. This perturbation will have an impact on, say, features in layer 3. However, the “knock-on” effects of this impact will be at least partially (and perhaps completely) overwritten, since the MLP outputs at layers 4 and 5 are clamped to values computed at the outset. Note that our alternative approach to patching, [“iterative patching,”](https://transformer-circuits.pub#appendix-patching) does capture such knock-on effects. However, because of this, it is not as suitable as a validation of mechanisms in an attribution graph, since the perturbation is less precise.

Another subtle aspect of our perturbation experiments is that direct feature-feature interactions described by an attribution graph are nearly forced to be confirmed in steering experiments (when we freeze attention patterns), since they measure linear effects of source feature decoders on target feature encoders (via fixed linear weights, conditioned on frozen attention patterns) – and guaranteed when choosing the intervention layer to be the same as the target feature’s encoder layer. Thus, the nontrivial thing we are testing with perturbation experiments is the validity of the attribution graph’s claims about “knock-on effects.” That is, suppose our attribution graph tells us that feature A excites feature B, which excites feature C, which upweights token X. When we perform a perturbation experiment by inhibiting feature A, we will get an inhibitory effect on feature B “for free,” and we are interested in validating whether the subsequent effects on features C and the output token are as expected.

Our choice of steering factor scales for intervention experiments is somewhat ad-hoc and empirically driven. For instance, in inhibition experiments, to meaningfully change the output predictions we often must clamp features to negative multiples of their original value, rather than simply to 0. In patching experiments where we add in a feature that was not originally active on a prompt, we often do so using activations significantly greater than that feature’s typical activations.

Why do we need to “overcompensate” in this fashion? We suspect that this is because even when our features play the mechanistic roles that we expect based on attribution graphs, our perturbation experiments capture these mechanisms incompletely:

We often find that there are many features in a given graph which seem to have similar roles. There are a few candidate theories for explaining this phenomenon (which we discuss below). Whatever the underlying reason, this suggests that individual features are often best understood as partial contributors to a component of the mechanism used by the model. To capture these components more completely, we group related features into “supernodes”. While this process is inherently somewhat subjective, we find it is important to clarify key mechanisms.

Grouping features produces a simplified “supernode graph,” where we compute the edges between supernodes according to the formula below:

Where:

Hypotheses for why there are so many related features include:

Instead of constrained patching, we may wish to measure the effect of a feature intervention within the patching range in addition to its effect outside of the range. To do so, we can iteratively recompute the values of features which read from layers in the patching range, letting them take on new values due to our intervention. We call this iterative patching. While iterative patching may seem advantageous since it accounts for all of the effects of a feature intervention (at least according to the replacement model), it can often lead to cascading errors, as we need to repeatedly encode and decode features. In addition, the main goal of interventions is to validate edges between nodes and supernodes in attribution graphs. Constrained patching provides a simpler way to do this by limiting indirect effects.

The illustration below shows cascading effects. We patch a feature at layer 1, and impact two features computed at layer 2, and so are affected by our patch. At the end of our patch, we inject the computed state of the residual stream back into the model.

In this section, we show full intervention results for some of the case studies we discussed above.

Below, we show the effect of inhibiting every supernode in the acronym case study with an inhibition factor of 1.

Supernodes require different steering strengths to show an effect because the strength of their outgoing edges varies. This strength depends both on the number of features in the supernode and their individual edge strengths. The steering plot above shows that inhibiting “say _A” or “say DA” has only a minor effect on the logit at a strength of −1. This is partially due to “say _A” containing only one feature, while supernodes like “say D_” contain six. “say _A”’s influence on the logit is also indirect and mediated by the “say DA” supernode. If we increase steering strength to −5, we do observe an effect from inhibiting “say _A”.

For reference, we also show the effect of inhibiting more of the supernodes in the 36+59 prompt.

We share an MIT licensed version of the interactive attribution graph interface used in this paper on [github](https://github.com/anthropics/attribution-graphs-frontend).

This interface has been simplified from the full version used internally for rapid exploration in favor of a publicly sharable version useful for inspecting labeled graphs. Analysis tools which were trimmed, and may be useful for practitioners to reimplement, include:

A rearrangeable grid containing these diverse “widgets” made it easier to experiment with new ideas and rescale the interface to show dozens or thousands of features.

In many models, the norm of the residual stream grows exponentially over the forward pass. Heimersheim and Turner [in practice](https://transformer-circuits.pub#evaluating-graphs-paths).

We observe the same phenomenon in our research model, 18L, which also has normalization denominators.

In the [main text](https://transformer-circuits.pub#global-weights), we showed a single feature’s virtual weights, and found that many large connections had weak coactivation statistics. Here, we run a similar analysis on a larger sample of features to better understand if this pattern generalizes.

We begin with the set of features that activate on our addition prompts and appear in our pruned attribution graphs. From these, we sample 1000 features with equal representation across layers. We then compute the virtual weights and coactivation statistics for all input and output connections of these features using a dataset of ~150M tokens of the CLT’s training data.

Below, we show the virtual weight distribution across all of these connections, and compare it to the distribution of weights between pairs that ever coactivate on the dataset. The percentage of coactive weights within each bin appears as text labels in the figure.

We consider weights between features with no coactivations to be “unused” in our dataset; a source feature that never coactivates with a target feature likely never causally affects the target's activation value.

In general, a large mass of small weights dominates the distribution (as expected). The smallest weights are very often unused with many bins having single-digit “use percentages.” Large negative weights also stand out as a difference between the raw and coactive distributions, yet this result isn’t surprising (even without interference); if an upstream feature is active, such a strong negative weight might guarantee that its downstream target never activates.

More significantly, we also observe that even among extremely large positive weights (beyond the 99.9999th percentile), a significant proportion are unused. This is not conclusive proof of interference, because we may still miss crucial tokens in our sampling where these edges are used. Still, we take this analysis as suggestive of interference in 18L.

We show the full output weights (over tokens 0, 1, ..., 999) for three features whose output patterns are neither periodic nor approximate magnitudes. The first promotes “simple numbers”, promoting both smaller numbers and rounder numbers. The second promotes numbers starting with 9, and the latter promotes numbers starting (or, to a lesser degree, ending) with 95. (In the color scheme, upweighted tokens are red and downweighted tokens are blue.)

We find similar families of features in Haiku and 18L for performing addition, but note that the ones for 18L are less precise. 18L performs slightly worse on these prompts (98.6% top-1 accuracy vs. 100% for Haiku 3.5), which possibly reflects this imprecision.

To evaluate our replacement model and attribution graphs, we curated a dataset of random but nontrivial pretraining tokens. Specifically, we considered short sequences from the Pile (minus books) dataset 

`1e-4` to filter out stop tokens and common tokens like articles, punctuations, and newlines;
Due to the expense of graph generation, we used a dataset size of n=260 for graph evaluation.

The below table contains a random sample of prompts and graphs for our Pile minus books evaluation set. It includes graphs for both our 10m cross-layer transcoder (CLT) and 10m per-layer transcoder (PLT) dictionaries on our 18L model.

| Links | Prompt | Target | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-29-clt)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-29-plt) | `arrangement, the optical fiber 104 is optically` | `coupled` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-37-clt)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-37-plt) | `" "I already called the police." "They\'re almost here." "You got to go." "Goodbye." "Good` | `luck` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-50-clt)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-50-plt) | `types, such as a front projection type and a rear projection type, depending on how images are` | `projected` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-99-clt)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-99-plt) | `for grants at the moment to pay a few students a paltry sum to stay and work in her lab over the` | `summer` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-179-clt)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pmb-179-plt) | `invention. In a separate aspect, the invention provides a method of potentiating the` actions of other CNS active compounds. This method comprises administering an | `effective` | 

We also include a sample of the basic curated prompts we used in the development of our techniques and their corresponding CLT and PLT graphs. These were designed to exercise basic capabilities such as factual recall, analogical reasoning, memorization, arithmetic, multilinguality, and in-context learning.

| Links | Prompt | Target | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=michael-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=michael-plt-clean) | `Fact: Michael Jordan plays the sport of` | `basketball` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=michael-fr-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=michael-fr-plt-clean) | `Fait: Michael Jordan joue au`  | `basket` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=sally-school-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=sally-school-plt-clean) | `At first, Sally hated school. But over time she changed her mind. Now she is`  | `happy` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=iasg-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=iasg-plt-clean) | `The International Advanced Security Group (IA`  | `SG` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=ndag-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=ndag-plt-clean) | `The National Digital Analytics Group (N`  | `DAG` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=count-by-sevens-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=count-by-sevens-plt-clean) | `7 14 21 28 35`  | `42` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=common-colors-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=common-colors-plt-clean) | `grass: green sky: blue corn: yellow carrot: orange strawberry:`  | `red` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=opposite-hot-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=opposite-hot-plt-clean) | `The opposite of "hot" is "`  | `cold` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=opposite-of-small-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=opposite-of-small-plt-clean) | `The opposite of "small" is "`  | `big` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=five-plus-three-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=five-plus-three-plt-clean) | `5 + 3 =`  | `8` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=uspto-telephone-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=uspto-telephone-plt-clean) | `Examiner interviews are available via`  | `telephone` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=currency-analogy-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=currency-analogy-plt-clean) | `Mexico:peso :: Europe:`  | `euro` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=capital-analogy-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=capital-analogy-plt-clean) | `Zagreb:Croatia :: Copenhagen:`  | `Denmark` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pandas-group-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=pandas-group-plt-clean) | `def customer_spending(transaction_df): for customer_id, customer_df in transaction_df.` | `group` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=season-after-spring-fr-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=season-after-spring-fr-plt-clean) | `La saison après le printemps s'appelle l'` | `été` | 
| [clt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=str-indexing-pos-0-clt-clean)[plt](https://transformer-circuits.pub/static_js/attribution_graphs/index.html?slug=str-indexing-pos-0-plt-clean) | `a = "Craig" assert a[0] == "`  | `C` |
