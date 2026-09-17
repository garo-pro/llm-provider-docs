Title: Toy Models of Superposition

URL Source: https://transformer-circuits.pub/2022/toy_model

Markdown Content:
It would be very convenient if the individual neurons of artificial neural networks corresponded to cleanly interpretable features of the input. For example, in an “ideal” ImageNet classifier, each neuron would fire only in the presence of a specific visual feature, such as the color red, a left-facing curve, or a dog snout. Empirically, in models we have studied, some of the neurons do cleanly map to features. But it isn't always the case that features correspond so cleanly to neurons, especially in large language models where it actually seems rare for neurons to correspond to clean features. This brings up many questions. Why is it that neurons sometimes align with features and sometimes don't? Why do some models and tasks have many of these clean neurons, while they're vanishingly rare in others?

In this paper, we use toy models — small ReLU networks trained on synthetic data with sparse input features — to investigate how and when models represent more features than they have dimensions. We call this phenomenon superposition 

Consider a toy model where we train an embedding of five features of varying importance

Not only can models store additional features in superposition by tolerating some interference, but we'll show that, at least in certain limited cases, models can perform computation while in superposition. (In particular, we'll show that models can put simple circuits computing the absolute value function in superposition.) This leads us to hypothesize that the neural networks we observe in practice are in some sense noisily simulating larger, highly sparse networks. In other words, it's possible that models we train can be thought of as doing “the same thing as” an imagined much-larger model, representing the exact same features but with no interference.

Feature superposition isn't a novel idea. A number of previous interpretability papers have considered it 

For interpretability researchers, our main contribution is providing a direct demonstration that superposition occurs in artificial neural networks given a relatively natural setup, suggesting this may also occur in practice. That is, we show a case where interpreting neural networks as having sparse structure in superposition isn't just a useful post-hoc interpretation, but actually the "ground truth" of a model. We offer a theory of when and why this occurs, revealing a  [phase diagram](https://transformer-circuits.pub#phase-change) for superposition. This [explains](https://transformer-circuits.pub#privileged-basis) why neurons are sometimes "monosemantic" responding to a single feature, and sometimes "polysemantic" [complex geometric structure](https://transformer-circuits.pub#geometry).

But our results may also be of broader interest. We find preliminary evidence that superposition may be linked to adversarial examples and grokking, and might also suggest a theory for the performance of mixture of experts models. More broadly, the toy model we investigate has unexpectedly rich structure, exhibiting [phase changes](https://transformer-circuits.pub#phase-change), a [geometric structure](https://transformer-circuits.pub#geometry) based on uniform polytopes, ["energy level"-like jumps](https://transformer-circuits.pub#learning-jumps) during training, and a [phenomenon](https://transformer-circuits.pub#geometry-uniform) which is qualitatively similar to the fractional quantum Hall effect in physics, among other striking phenomena. We originally investigated the subject to gain understanding of cleanly-interpretable neurons in larger models, but we've found these toy models to be surprisingly interesting in their own right.

In our toy models, we are able to demonstrate that:

Our toy models are simple ReLU networks, so it seems fair to say that neural networks exhibit these properties in at least some regimes, but it's very unclear what to generalize to real networks.

In our work, we often think of neural networks as having features of the input represented as directions in activation space. This isn't a trivial claim. It isn't obvious what kind of structure we should expect neural network representations to have. When we say something like "word embeddings have a gender direction" or "vision models have curve detector neurons", one is implicitly making strong claims about the structure of network representations.

Despite this, we believe this kind of "linear representation hypothesis" is supported both by significant empirical findings and theoretical arguments. One might think of this as two separate properties, which we'll explore in more detail shortly:

If we hope to reverse engineer neural networks, we need a property like decomposability. Decomposability is what [allows us to reason about the model](https://transformer-circuits.pub/2022/mech-interp-essay/index.html) without fitting the whole thing in our heads! But it's not enough for things to be decomposable: we need to be able to access the decomposition somehow. In order to do this, we need to identify the individual features within a representation. In a linear representation, this corresponds to determining which directions in activation space correspond to which independent features of the input.

Sometimes, identifying feature directions is very easy because features seem to correspond to neurons. For example, many neurons in the early layers of InceptionV1 clearly correspond to features (e.g. curve detector neurons 

Superposition has been hypothesized in previous work 

The goal of this section will be to motivate these ideas and unpack them in detail.

It's worth noting that many of the ideas in this section have close connections to ideas in other lines of interpretability research (especially disentanglement), neuroscience (distributed representations, population codes, etc), compressed sensing, and many other lines of work. This section will focus on articulating our perspective on the problem. We'll discuss these other lines of work in detail in [Related Work](https://transformer-circuits.pub#related-work).

When we talk about "features" and how they're represented, this is ultimately theory building around several observed empirical phenomena. Before describing how we conceptualize those results, we'll simply describe some of the major results motivating our thinking:

`V("king") - V("man") + V("woman") = V("queen")` (but see 
As a result, we tend to think of neural network representations as being composed of features which are represented as directions. We'll unpack this idea in the following sections.

Our use of the term "feature" is motivated by the interpretable properties of the input we observe neurons (or word embedding directions) responding to. There's a rich variety of such observed properties![curve detectors](https://distill.pub/2020/circuits/curve-detectors/)[high-low frequency detectors](https://distill.pub/2020/circuits/frequency-edges/)[oriented dog-head detectors](https://distill.pub/2020/circuits/zoom-in/#claim-2-dog) or [car detectors](https://distill.pub/2020/circuits/zoom-in/#claim-2-superposition)[famous people](https://distill.pub/2021/multimodal-neurons/#person-neurons), [emotions](https://distill.pub/2021/multimodal-neurons/#emotion-neurons), [geographic regions](https://distill.pub/2021/multimodal-neurons/#region-neurons), and [more](https://distill.pub/2021/multimodal-neurons/#guided-tour-of-neuron-families) 

But even with that motivation, it turns out to be quite challenging to create a satisfactory definition of a feature. Rather than offer a single definition we're confident about, we consider three potential working definitions:

We've written this paper with the final "neurons in sufficiently large models" definition in mind. But we aren't overly attached to it, and actually think it's probably important to not prematurely attach to a definition.

As we've mentioned in previous sections, we generally think of features as being represented by directions. For example, in word embeddings, "gender" and "royalty" appear to correspond to directions, allowing arithmetic like `V("king") - V("man") + V("woman") = V("queen")` 

Let's call a neural network representation linear if features correspond to directions in activation space. In a linear representation, each feature 

We don't think it's a coincidence that neural networks empirically seem to have linear representations. Neural networks are built from linear functions interspersed with non-linearities. In some sense, the linear functions are the vast majority of the computation (for example, as measured in FLOPs). Linear representations are the natural format for neural networks to represent information in! Concretely, there are three major benefits:

It is possible to construct non-linear representations, and retrieve information from them, if you use multiple layers (although even these examples can be seen as linear representations with more exotic features). We provide an example in the appendix. However, our intuition is that non-linear representations are generally inefficient for neural networks.

One might think that a linear representation can only store as many features as it has dimensions, but it turns out this isn't the case! We'll see that the phenomenon we call superposition will allow models to store more features – potentially many more features – in linear representations.

For discussion on how this view of features squares with a conception of features as being multidimensional manifolds, see the appendix “What about Multidimensional Features?”.

Even if features are encoded as directions, a natural question to ask is which directions? In some cases, it seems useful to consider the basis directions, but in others it doesn't. Why is this?

When researchers study word embeddings, it doesn't make sense to analyze basis directions. There would be no reason to expect a basis dimension to be different from any other possible direction. One way to see this is to imagine applying some random linear transformation 

But many neural network layers are not like this. Often, something about the architecture makes the basis directions special, such as applying an activation function. This "breaks the symmetry", making those directions special, and potentially encouraging features to align with the basis dimensions. We call this a privileged basis, and call the basis directions "neurons." Often, these neurons correspond to interpretable features.

From this perspective, it only makes sense to ask if a neuron is interpretable when it is in a privileged basis. In fact, we typically reserve the word "neuron" for basis directions which are in a privileged basis. (See longer discussion [here](https://transformer-circuits.pub/2022/solu/index.html#section-3-2).)

Note that having a privileged basis doesn't guarantee that features will be basis-aligned – we'll see that they often aren't! But it's a minimal condition for the question to even make sense.

Even when there is a privileged basis, it's often the case that neurons are "polysemantic", responding to several unrelated features. One explanation for this is the [superposition hypothesis](https://distill.pub/2020/circuits/zoom-in/#claim-2-superposition)

Several results from mathematics suggest that something like this might be plausible:

Concretely, in the superposition hypothesis, features are represented as almost-orthogonal directions in the vector space of neuron outputs. Since the features are only almost-orthogonal, one feature activating looks like other features slightly activating. Tolerating this "noise" or "interference" comes at a cost. But for neural networks with highly sparse features, this cost may be outweighed by the benefit of being able to represent more features! (Crucially, sparsity greatly reduces the costs since sparse features are rarely active to interfere with each other, and non-linear activation functions create opportunities to filter out small amounts of noise.)

One way to think of this is that a small neural network may be able to noisily "simulate" a sparse larger model:

Although we've described superposition with respect to neurons, it can also occur in representations with an unprivileged basis, such as a word embedding. Superposition simply means that there are more features than dimensions.

The ideas in this section might be thought of in terms of four progressively more strict properties that neural network representations might have.

The first two (decomposability and linearity) are properties we hypothesize to be widespread, while the latter (non-superposition and basis-aligned) are properties we believe only sometimes occur.

If one takes the superposition hypothesis seriously, a natural first question is whether neural networks can actually noisily represent more features than they have neurons. If they can't, the superposition hypothesis may be comfortably dismissed.


The intuition from linear models would be that this isn't possible: the best a linear model can do is to store the principal components. But we'll see that adding just a slight nonlinearity can make models behave in a radically different way! This will be our first demonstration of superposition. (It will also be an object lesson in the complexity of even very simple neural networks.)

Our goal is to explore whether a neural network can project a high dimensional vector 

We begin by describing the high-dimensional vector 

Since we don't have any ground truth for features, we need to create synthetic data for 

Concretely, our synthetic data is defined as follows: The input vectors 

We will actually consider two models, which we motivate below. The first "linear model" is a well understood baseline which does not exhibit superposition. The second "ReLU output model" is a very simple model which does exhibit superposition. The two models vary only in the final activation function.

Why these models?

The superposition hypothesis suggests that each feature in the higher-dimensional model corresponds to a direction in the lower-dimensional space. This means we can represent the down projection as a linear map 

To recover the original vector, we'll use the transpose of the same matrix 

We also add a bias. One motivation for this is that it allows the model to set features it doesn't represent to their expected value. But we'll see later that the ability to set a negative bias is important for superposition for a second set of reasons – roughly, it allows models to discard small amounts of noise.

The final step is whether to add an activation function. This turns out to be critical to whether superposition occurs. In a real neural network, when features are actually used by the model to do computation, there will be an activation function, so it seems principled to include one at the end.

Our loss is weighted mean squared error weighted by the feature importances, 

Our first experiment will simply be to train a few ReLU output models with different sparsity levels and visualize the results. (We'll also train a linear model – if optimized well enough, the linear model solution does not depend on sparsity level.)

The main question is how to visualize the results. The simplest way is to visualize 

But the thing we really care about is this hypothesized phenomenon of superposition – does the model represent "extra features" by storing them non-orthogonally? Is there a way to get at it more explicitly? Well, one question is just how many features the model learns to represent. For any feature, whether or not it is represented is determined by 

We'd also like to understand whether a given feature shares its dimension with other features. For this, we calculate 

We can visualize the model we looked at previously this way:

Now that we have a way to visualize models, we can start to actually do experiments.  We'll start by considering models with only a few features (

As our standard intuitions would expect, the linear model always learns the top-[The Geometry of Superposition](https://transformer-circuits.pub#geometry).

The results are qualitatively similar for models with more features and hidden dimensions. For example, if we consider a model with 

In the previous section, we observed a surprising empirical result: adding a ReLU to the output of our model allowed a radically different solution – superposition – which doesn't occur in linear models.

The model where it occurs is still quite mathematically simple. Can we analytically understand why superposition is occurring? And for that matter, why does adding a single non-linearity make things so different from the linear model case? It turns out that we can get a fairly satisfying answer, revealing that our model is governed by balancing two competing forces – feature benefit and interference – which will be useful intuition going forwards. We'll also discover a connection to the famous Thomson Problem in chemistry.

Let's start with the linear case. This is well understood by prior work! If one wants to understand why linear models don't exhibit superposition, the easy answer is to observe that linear models essentially perform PCA. But this isn't fully satisfying: if we set aside all our knowledge and intuition about linear functions for a moment, why exactly is it that superposition can't occur?

A deeper understanding can come from the results of Saxe et al. 

The Saxe results reveal that there are fundamentally two competing forces which control learning dynamics in the considered model. Firstly, the model can attain a better loss by representing more features (we've labeled this "feature benefit"). But it also gets a worse loss if it represents more than it can fit orthogonally due to "interference" between features.[coherence](<https://en.wikipedia.org/wiki/Mutual_coherence_(linear_algebra)>) in compressed sensing, 

Can we achieve a similar kind of understanding for the ReLU output model? Concretely, we'd like to understand 

The integral over 

This new equation is vaguely similar to the famous [Thomson problem](https://en.wikipedia.org/wiki/Thomson_problem) in chemistry. In particular, if we assume uniform importance and that there are a fixed number of features with 

Another interesting property is that ReLU makes negative interference free in the 1-sparse case. This explains why the solutions we've seen prefer to only have negative interference when possible. Further, using a negative bias can convert small positive interferences into essentially being negative interferences.

What about the terms corresponding to less sparse vectors? We leave explicitly writing these out to the reader, but the main idea is that there are multiple compounding interferences, and the "active features" can experience interference. In a [later section](https://transformer-circuits.pub#geometry-dimensionality), we'll see that features often organize themselves into sparse interference graphs such that only a small number of features interfere with another feature – it's interesting to note that this reduces the probability of compounding interference and makes the 1-sparse loss term more important relative to others.

The results in the previous section seem to suggest that there are three outcomes for a feature when we train a model: (1) the feature may simply not be learned; (2) the feature may be learned, and represented in superposition; or (3) the model may represent a feature with a dedicated dimension. The transitions between these three outcomes seem sharp. Possibly, there's some kind of phase change. 

One way to understand this better is to explore if there's something like a "phase diagram" from physics, which could help us understand when a feature is expected to be in one of these regimes.  Although we can see hints of this in [our previous experiment](https://transformer-circuits.pub#demonstrating-basic-results), it's hard to really isolate what's going on because many features are changing at once and there may be interaction effects. As a result, we set up the following experiment to better isolate the effects.

As an initial experiment, we consider models with 2 features but only 1 hidden layer dimension. We still consider the ReLU output model, 

We can compare this to a theoretical "toy model of the toy model" where we can get closed form solutions for the loss of different weight configurations as a function of importance and sparsity. There are three natural ways to store 2 features in 1 dimension: [this notebook](https://github.com/wattenberg/superposition/blob/main/Exploring_Exact_Toy_Models.ipynb)).

As expected, sparsity is necessary for superposition to occur, but we can see that it interacts in an interesting way with relative feature importance. But most interestingly, there appears to be a real phase change, observed in both the empirical and theoretical diagrams! The optimal weight configuration discontinuously changes in magnitude and superposition. (In the theoretical model, we can analytically confirm that there's a first-order phase change: there's crossover between the functions, causing a discontinuity in the derivative of the optimal loss.)

We can ask this same question of embedding three features in two dimensions. This problem still has a single "extra feature" (now the third one) we can study, asking what happens as we vary its importance relative to the other two and change sparsity.

For the theoretical model, we now consider four natural solutions. We can describe solutions by asking "what feature direction did [this notebook](https://github.com/wattenberg/superposition/blob/main/Exploring_Exact_Toy_Models.ipynb). We do not consider a last solution of putting all the features in joint superposition, 

These diagrams suggest that there really is a phase change between different strategies for encoding features. However, we'll see in the next section that there's much more complex structure this preliminary view doesn't capture.

We've seen that superposition can allow a model to represent extra features, and that the number of extra features increases as we increase sparsity. In this section, we'll investigate this relationship in more detail, discovering an unexpected geometric story: features seem to organize themselves into geometric structures such as pentagons and tetrahedrons! In some ways, the structure described in this section seems "too elegant to be true" and we think there's a good chance it's at least partly idiosyncratic to the toy model we're investigating. But it seems worth investigating because if anything about this generalizes to real models, it may give us a lot of leverage in understanding their representations.

We'll start by investigating uniform superposition, where all features are identical: independent, equally important and equally sparse. It turns out that uniform superposition has a surprising connection to the geometry of uniform polytopes! Later, we'll move on to investigate non-uniform superposition, where features are not identical. It turns out that this can be understood, at least to some extent, as a deformation of uniform superposition.

As mentioned above, we begin our investigation with uniform superposition, where all features have the same importance and sparsity. We'll see later that this case has some unexpected structure, but there's also a much more basic reason to study it: it's much easier to reason about than the non-uniform case, and has fewer variables we need to worry about in our experiments.

We'd like to understand what happens as we change feature sparsity, 

A convenient way to measure the number of features the model has learned is to look at the Frobenius norm, 

We'll plot 

Surprisingly, we find that this graph is "sticky" at [this diagram](https://www.researchgate.net/profile/Charles-Dunkl/publication/318205131/figure/fig3/AS:512607505350656@1499226561051/Fractional-quantum-Hall-effect.png).) Why is this? On inspection, the 

It turns out that antipodal pairs are just the tip of the iceberg. Hiding underneath this curve are a number of extremely specific geometric configurations of features.

In the previous section, we saw that there's a sticky regime where the model has "half a dimension per feature" in some sense. This is an average statistical property of the features the model represents, but it seems to hint at something interesting. Is there a way we could understand what "fraction of a dimension" a specific feature gets?

We'll define the dimensionality of the 

where 

Intuitively, the numerator represents the extent to which a given feature is represented, while the denominator is "how many features share the dimension it is embedded in" by projecting each feature onto its dimension. In the antipodal case, each feature participating in an antipodal pair will have a dimensionality of 

We can now break the above plot down on a per-feature basis. This reveals many more of these "sticky points"! To help us understand this better, we're going to create a scatter plot annotated with some additional information:

Let's look at the resulting plot, and then we'll try to figure out what it's showing us:

What is going on with the points clustering at specific fractions?? We'll see shortly that the model likes to create specific weight geometries and kind of jumps between the different configurations.

In the previous section, we developed a theory of superposition as a phase change. But everything on this plot between 0 (not learning a feature) and 1 (dedicating a dimension to a feature) is superposition. Superposition is what happens when features have fractional dimensionality. That is to say – superposition isn't just one thing!

How can we relate this to our original understanding of the phase change? We often think of water as only having three phases: ice, water and steam. But this is a simplification: there are actually many phases of ice, often corresponding to different crystal structures (eg. hexagonal vs cubic ice). In a vaguely similar way, neural network features seem to also have many other phases within the general category of "superposition."

In the previous diagram, we found that there are distinct lines corresponding to dimensionality of: ¾ (tetrahedron), ⅔ (triangle), ½ (antipodal pair), ⅖ (pentagon), ⅜ (square antiprism), and 0 (feature not learned). We believe there would also be a 1 (dedicated dimension for a feature) line if not for the fact that basis features are indistinguishable from other directions in the dense regime.

Several of these configurations may jump out as solutions to the famous [Thomson problem](https://en.wikipedia.org/wiki/Thomson_problem). (In particular, [square antiprisms](https://en.wikipedia.org/wiki/Square_antiprism) are much less famous than cubes and are primarily of note for their [role in molecular geometry](https://en.wikipedia.org/wiki/Square_antiprismatic_molecular_geometry) due to being a Thomson problem solution.) As we saw earlier, there is a very real sense in which our model can be understood as solving a generalized version of the Thomson problem. When our model chooses to represent a feature, the feature is embedded as a point on an 

A second clue as to what's going on is that there are lines for the Thomson solutions which are uniform polyhedra (e.g. tetrahedron), but there seem to be split lines where we'd expect to see non-uniform solutions (e.g. instead of a ⅗ line for triangular bipyramids we see a co-occurrence of points at ⅔ for triangles and points at ½ for antipodes). In a uniform polyhedron, all vertices have the same geometry, and so if we embed features as them each feature has the same dimensionality. But if we embed features as a non-uniform polyhedron, different features will have more or less interference with others.

In particular, many of the Thomson solutions can be understood as [tegum products](https://polytope.miraheze.org/wiki/Tegum_product) (an operation which constructs polytopes  by embedding two polytopes in orthogonal subspaces) of smaller uniform polytopes. (In the earlier graph visualizations of feature geometry, two subgraphs are disconnected if and only if they are in different tegum factors.) As a result, we should expect their dimensionality to actually correspond to the underlying factor uniform polytopes. 

This also suggests a possible reason why we observe 3D Thomson problem solutions, despite the fact that we're actually studying a higher dimensional version of the problem. Just as many 3D Thomson solutions are tegum products of 2D and 1D solutions, perhaps higher dimensional solutions are often tegum products of 1D, 2D, and 3D solutions.

The orthogonality of factors in tegum products has interesting implications. For the purposes of superposition, it means that there can't be any "interference" across tegum-factors. This may be preferred by the toy model: having many features interfere simultaneously could be really bad for it. (See related discussion in [our earlier mathematical analysis](https://transformer-circuits.pub#demonstrating-math).)

At this point, it's worth making explicit that there's a correspondence between polytopes and symmetric, positive-definite, low-rank matrices (i.e. matrices of the form 

In some ways, the correspondence is trivial. If one has a rank-

Put another way, there's an exact correspondence between polytopes and strategies for superposition. For example, every strategy for putting three features in superposition in a 2-dimensional space corresponds to a triangle, and every triangle corresponds to such a strategy. From this perspective, it doesn't seem surprising that if we have three equally important and equally sparse features, the optimal strategy is an equilateral triangle.

This correspondence also goes the other direction. Suppose we have a rank 

In fact, given such a set of orthogonal vectors, we can construct a polytope by starting with [regular](https://polytope.miraheze.org/wiki/Simplex) . This is interesting because it's in some sense the "minimal possible superposition." Assuming that features are equally important and sparse, the best possible direction to not represent is the fully dense vector 

So far, this section has focused on the geometry of uniform superposition, where all features are of equal importance, equal sparsity, and independent. The model is essentially solving a variant of the Thomson problem. Because all features are the same, solutions corresponding to uniform polyhedra get especially low loss. In this subsection, we'll study non-uniform superposition, where features are somehow not uniform. They may vary in importance and sparsity, or have a correlational structure that makes them not independent. This distorts the uniform geometry we saw earlier.

In practice, it seems like superposition in real neural networks will be non-uniform, so developing an understanding of it seems important. Unfortunately, we're far from a comprehensive theory of the geometry of non-uniform superposition at this point. As a result, the goal of this section will merely be to highlight some of the more striking phenomena we observe:

We attempt to illustrate these phenomena with some representative experiments below.

The simplest kind of non-uniform superposition is to vary one feature and leave the others uniform. As an experiment, let's consider an experiment where we represent 

If we make it sufficiently sparse, there's a phase change, and it collapses from a pentagon to a pair of digons with the sparser point at zero. The phase change corresponds to loss curves corresponding to the two different geometries crossing over. (This observation allows us to directly confirm that it is genuinely a first order phase change.)

To visualize the solutions, we canonicalize them, rotating them to align with each other in a consistent manner.

These results seem to suggest that, at least in some cases, non-uniform superposition can be understood as a deformation of uniform superposition and jumping between uniform superposition configurations rather than a totally different regime. Since uniform superposition has a lot of understandable structure, but real world superposition is almost certainly non-uniform, this seems very promising!

The reason pentagonal solutions are not on the unit circle is because models reduce the effect of positive interference, setting a slight negative bias to cut off noise and setting their weights to 

A note for reimplementations: optimizing with a two-dimensional hidden space makes this easier to study, but the actual optimization process turns out to be really challenging from gradient descent – a lot harder than even just having three dimensions. Getting clean results required fitting each model multiple times and taking the solution with the lowest loss. However, there's a silver lining to this: visualizing the sub-optimal solutions on a scatter plot as above allows us to see the loss curves for different geometries and gain greater insight into the phase change.

A more complicated form of non-uniform superposition occurs when there are correlations between features. This seems essential for understanding superposition in the real world, where many features are correlated or anti-correlated.

For example, one very pragmatic question to ask is whether we should expect polysemantic neurons to group the same features together across models. If the groupings were random, you could use this to detect polysemantic neurons, by comparing across models! However, we'll see that correlational structure strongly influences which features are grouped together in superposition.

The behavior seems to be quite nuanced, with a kind of "order of preferences" for how correlated features behave in superposition. The model ideally represents correlated features orthogonally, in separate tegum factors with no interactions between them. When that fails, it prefers to arrange them so that they're as close together as possible – it prefers positive interference between correlated features over negative interference. Finally, when there isn't enough space to represent all the correlated features, it will collapse them and represent their principal component instead! Conversely, when features are anti-correlated, models prefer to have them interfere, especially with negative interference. We'll demonstrate this with a few experiments below.

Throughout this section we'll refer to "correlated feature sets" and "anticorrelated feature sets".

Correlated Feature Sets. Our correlated feature sets can be thought of as "bundles" of co-occurring features. One can imagine a highly idealized version of what might happen in an image classifier: there could be a bundle of features used to identify animals (fur, ears, eyes) and another bundle used to identify buildings (corners, windows, doors). Features from one of these bundles are likely to appear together. Mathematically, we represent this by linking the choice of whether all the features in a correlated feature set are zero or not together. Recall that we [originally defined](https://transformer-circuits.pub#demonstrating-setup-x) our synthetic distribution to have features be zero with probability 

Anticorrelated Feature Sets. One could also imagine anticorrelated features which are extremely unlikely to occur together. To simulate these, we'll have anticorrelated feature sets where only one feature in the set can be active at a time. To simulate this, we'll have the feature set be entirely zero with probability 

For our initial investigation, we simply train a number of small toy models with correlated and anti-correlated features and observe what happens. To make this easy to study, we limit ourselves to the 

It turns out that the tendency of models to arrange correlated features to be orthogonal is actually quite a strong phenomenon. In particular, for larger models, it seems to generate a kind of "local almost-orthogonal basis" where, even though the model as a whole is in superposition, the correlated feature sets considered in isolation are (nearly) orthogonal and can be understood as having very little superposition.

To investigate this, we train a larger model with two sets of correlated features and visualize 

If this result holds in real neural networks, it suggests we might be able to make a kind of "local non-superposition" assumption, where for certain sub-distributions we can assume that the activating features are not in superposition. This could be a powerful result, allowing us to confidently use methods such as PCA which might not be principled to generally use in the context of superposition.


One of the most interesting properties is that there seems to be a trade off with Principal Components Analysis (PCA) and superposition. If there are two correlated features 

As an experiment, we consider six features, organized into three sets of correlated pairs. Features in each correlated pair are represented by a given color (red, green, and blue). The correlation is created by having both features always activate together – they're either both zero or neither zero. (The exact non-zero values they take when they activate is uncorrelated.)

As we vary the sparsity of the features, we find that in the very sparse regime, we observe superposition as expected, with features arranged in a hexagon and correlated features side-by-side. As we decrease sparsity, the features progressively "collapse" into their principal components. In very dense regimes, the solution becomes equivalent to PCA.

These results seem to hint that PCA and superposition are in some sense complementary strategies which trade off with one another. As features become more correlated, PCA becomes a better strategy. As features become sparser, superposition becomes a better strategy. When features are both sparse and correlated, mixtures of each strategy seem to occur. It would be nice to more deeply understand this space of tradeoffs.

It's also interesting to think about this in the context of continuous [equivariant features](https://distill.pub/2020/circuits/equivariance/), such as features which occur in different rotations. 

The focus of this paper is how superposition contributes to the functioning of fully trained neural networks, but as a brief detour it's interesting to ask how our toy models – and the resulting superposition – evolve over the course of training.

There are several reasons why these models seem like a particularly interesting case for studying learning dynamics. Firstly, unlike most neural networks, the fully trained models converge to a simple but non-trivial structure that rhymes with an emerging thread of evidence that neural network learning dynamics might have geometric weight structure that we can understand. One might hope that understanding the final structure would make it easier for us to understand the evolution over training. Secondly, superposition hints at surprisingly discrete structure (regular polytopes of all things!). We'll find that the underlying learning dynamics are also surprisingly discrete, continuing an emerging trend of evidence that neural network learning might be less continuous than it seems. Finally, since superposition has significant implications for interpretability, it would be nice to understand how it emerges over training – should we expect models to use superposition early on, or is it something that only emerges later in training, as models struggle to fit more features in?

Unfortunately, we aren't able to give these questions the detailed investigation they deserve within the scope of this paper. Instead, we'll limit ourselves to a couple particularly striking phenomena we've noticed, leaving more detailed investigation for future work.

Perhaps the most striking phenomenon we've noticed is that the learning dynamics of toy models with large numbers of features appear to be dominated by "energy level jumps" where features jump between different feature dimensionalities. (Recall that a feature's [dimensionality](https://transformer-circuits.pub#geometry-dimensionality) is the fraction of a dimension dedicated to representing a feature.)

Let's consider the problem setup we studied when investigating the geometry of uniform superposition in the [previous section](https://transformer-circuits.pub#geometry-dimensionality), where we have a large number of features of equal importance and sparsity. As we saw previously, the features ultimately arrange themselves into a small number of polytopes with fractional dimensionalities.

A natural question to ask is what happens to these feature dimensionalities over the course of training. Let's pick one model where all the features converge into digons and observe. In the first plot, each colored line corresponds to the dimensionality of a single feature. The second plot shows how the loss curve changes over the same duration.

Note how the dimensionality of some features "jump" between different values and swap places. As this happens, the loss curve also undergoes a sudden drop (a very small one at the first jump, and a larger one at the second jump).

These results make us suspect that seemingly smooth decreases of the loss curve in larger models are in fact composed of many small jumps of features between different configurations. (For similar results of sudden mechanistic changes, see Olsson et al.'s induction head phase change 

Many of our toy model solutions can be understood as corresponding to geometric structures. This is especially easy to see and study when there are only 

It turns out that, at least in some cases, the learning dynamics leading to these structures can be understood as a sequence of simple, independent geometric transformations!

One particularly interesting example of this phenomenon occurs in the context of [correlated features](https://transformer-circuits.pub#geometry-correlated-setup), as studied in the previous section. Consider the problem of representing 

(Although the last solution – an octahedron with features from different correlated sets arranged in antipodal pairs – seems to be a strong attractor, the learning trajectory visualized above appears to be one of a few different learning trajectories that attract the model. The different trajectories vary at step C: sometimes the model gets pulled directly into the antiprism configuration from the start or organizes features into antipodal pairs. Presumably this depends on which feature geometry the model is closest to when step B ends.)

The learning dynamics we observe here seem directly related to previous findings on simple models. 

Although we're most interested in the implications of superposition for interpretability, there appears to be a connection to adversarial examples. If one gives it a little thought, this connection can actually be quite intuitive.

In a model without superposition, the end-to-end weights for the first feature are:

But in a model with superposition, it's something like:

The 

To test this, we generated L2 adversarial examples (allowing a max L2 attack norm of 0.1 of the average input norm). We originally generated attacks with gradient descent, but found that for extremely sparse examples where ReLU neurons are in the zero regime 99% of the time, attacks were difficult, effectively due to gradient masking 

We find that vulnerability to adversarial examples sharply increases as superposition forms (increasing by >3×), and that the level of vulnerability closely tracks the number of features per dimension (the reciprocal of [feature dimensionality](https://transformer-circuits.pub#geometry-dimensionality)).

We're hesitant to speculate about the extent to which superposition is responsible for adversarial examples in practice. There are compelling theories for why adversarial examples occur without reference to superposition (e.g. [earlier results on this](https://transformer-circuits.pub#geometry-correlated)). It might be interesting for future work to see how far the hypothesis that superposition is a significant contributor to adversarial examples can be driven.

In addition to observing that superposition can cause models to be vulnerable to adversarial examples, we briefly experimented with adversarial training to see if the relationship could be used in the other direction to reduce superposition. To keep training reasonably efficient, we used the analytic optimal attack against a random feature. We found that this did reduce superposition, but attacks had to be made unreasonably large (80% input L2 norm) to fully eliminate it, which didn't seem satisfying. Perhaps stronger adversarial attacks would work better. We didn't explore this further since the increased cost and complexity of adversarial training made us want to prioritize other lines of attack on superposition first.

So far, we've explored superposition in a model without a privileged basis. We can rotate the hidden activations arbitrarily and, as long as we rotate all the weights, have the exact same model behavior. That is, for any ReLU output model with weights 

Models without a privileged basis are elegant, and can be an interesting analogue for certain neural network representations which don't have a privileged basis – word embeddings, or the transformer residual stream. But we'd also (and perhaps primarily) like to understand neural network representations where there are neurons which do impose a privileged basis, such as transformer MLP layers or conv net neurons.

Our goal in this section is to explore the simplest toy model which gives us a privileged basis. There are at least two ways we could do this: we could add an activation function or apply L1 regularization to the hidden layer. We'll focus on adding an activation function, since the representation we are most interested in understanding is hidden layers with neurons, such as the transformer MLP layer.

This gives us the following "ReLU hidden layer" model:

We'll train this model on the same data as before.

Adding a ReLU to the hidden layer radically changes the model from an interpretability perspective. The key thing is that while 

We'll discuss this in much more detail shortly, but here's a comparison of weights resulting from a linear hidden layer model and a ReLU hidden layer model:

Recall that we think of basis elements in the input as "features," and basis elements in the middle layer as "neurons". Thus 

What we see in the above plot is that the features are aligning with neurons in a structured way! Many of the neurons are simply dedicated to representing a feature! (This is the critical property that justifies why neuron-focused interpretability approaches – such as much of the work in the original Circuits thread – can be effective in some circumstances.)

Let's explore this in more detail.

Having a privileged basis opens up new possibilities for visualizing our models. As we saw above, we can simply inspect 

This stack plot visualization can be nice as models get bigger. It also makes polysemantic neurons obvious: they simply correspond to having more than one weight.

We'll now visualize a ReLU hidden layer toy model with 

However, we found that these small models were harder to optimize. For each model shown, we trained 1000 models and visualized the one with the lowest loss. Although the typical solutions are often similar to the minimal loss solutions shown, selecting the minimal loss solutions reveals even more structure in how features align with neurons. It also reveals that there are ranges of sparsity values where the optimal solution for all models trained on data with that sparsity has the same weight configurations.

The solutions are visualized below, both visualizing the raw 

The most important thing to pay attention to is how there's a shift from monosemantic to polysemantic neurons as sparsity increases. Monosemantic neurons do exist in some regimes! Polysemantic neurons exist in others. And they can both exist in the same model! Moreover, while it's not quite clear how to formalize this, it looks a great deal like there's a neuron-level phase change, mirroring the feature phase changes we saw earlier.

It's also interesting to examine the structure of the polysemantic solutions, which turn out to be surprisingly structured and neuron-aligned. Features typically correspond to sets of neurons (monosemantic neurons might be seen as the special case where features only correspond to singleton sets). There's also structure in how polysemantic neurons are. They transition from monosemantic, to only representing a few features, to gradually representing more. However, it's unclear how much of this is generalizable to real models.

Unfortunately, the toy model described in this section has a significant weakness, which limits the regimes in which it shows interesting results. The issue is that the model doesn't benefit from the ReLU hidden layer – it has no role except limiting how the model can encode information. If given any chance, the model will circumvent it. For example, given a hidden layer bias, the model will set all the biases to be positive, shifting the neurons into a positive regime where they behave linearly. If one removes the bias, but gives the model enough features, it will simulate a bias by averaging over many features. The model will only use the ReLU activation function if absolutely forced, which is a significant mark against studying this toy model.

We'll introduce a model without this issue in the next section, but wanted to study this model as a simpler case study.

So far, we've shown that neural networks can store sparse features in superposition and then recover them. But we actually believe superposition is more powerful than this – we think that neural networks can perform computation entirely in superposition rather than just using it as storage. This model will also give us a more principled way to study a privileged basis where features align with basis dimensions.

To explore this, we consider a new setup where we imagine our input and output layer to be the layers of our hypothetical disentangled model, but have our hidden layer be a smaller layer we're imagining to be the observed model which might use superposition. We'll then try to compute a simple non-linear function and explore whether it can use superposition to do this. Since the model will have (and need to use) the hidden layer non-linearity, we'll also see features align with a privileged basis.

Specifically, we'll have the model compute 

Since this model needs ReLU to compute absolute value, it doesn't have the issues the model in the previous section had with trying to avoid the activation function.

The input feature vector, 

Following the previous section, we'll consider the "ReLU hidden layer" toy model variant, but no longer tie the two weights to be identical:

The loss is still the mean squared error weighted by feature importances 

With this model, it's a bit less straightforward to study how individual features get embedded; because of the ReLU on the hidden layer, we can't just study 

As we saw in the previous section, having a hidden layer activation function means that it makes sense to visualize the weights in terms of neurons. We can visualize 

Let's look at what happens when we train a model with 

The resulting model – modulo a subtle issue about rescaling input and output weights

We've seen that – as expected – our toy model can learn to implement absolute value. But can it use superposition to compute absolute value for more features? To test this, we train models with 

A couple of notes on visualization: Since we're primarily interested in understanding superposition and polysemantic neurons, we'll show a stacked weight plot of the absolute values of weights. The features are colored by superposition. To make the diagrams easier to read, neurons are faintly colored based on how polysemantic they are (as judged by eye based on the plots). Neuron order is sorted by the importance of the largest feature.

Much like we saw in the ReLU hidden layer models, these results demonstrate that activation functions, under the right circumstances, create a privileged basis and cause features to align with basis dimensions. In the dense regime, we end up with each neuron representing a single feature, and we can read feature values directly off of neuron activations.

However, once the features become sufficiently sparse, this model, too, uses superposition to represent more features than it has neurons. This result is notable because it demonstrates the ability of neural networks to perform computation even on data that is represented in superposition.

Focusing on the intermediate sparsity regimes, we find several additional qualitative behaviors that we find fascinatingly reminiscent of behavior that has been observed in real, full-scale neural networks:

To begin, we find that in some regimes, many of the model's neurons will encode pure features, but a subset of them will be highly polysemantic. This is similar to the [phase change](https://transformer-circuits.pub#phase-change) we saw earlier in the ReLU output model. However, in that case, the phase change was with respect to features, with more important features not being put in superposition. In this experiment, the neurons don't have any intrinsic importance, but we see that the neurons representing the most important features (on the left) tend to be monosemantic.

We find this to bear a suggestive resemblance to [some previous work in vision models](https://distill.pub/2020/circuits/zoom-in/#claim-2-superposition), which found some layers that contained "mostly pure" feature neurons, but with some neurons representing additional features on a different scale.

We also note that many neurons appear to be associated with a single "primary" feature – encoded by a relatively large weight – coupled with one or more "secondary" features encoded with smaller-magnitude weights to that neuron. If we were to observe the activations of such a neuron over a range of input examples, we would find that the largest activations of that neuron were all or nearly-all associated with the presence of the "primary" feature, but that the lower-magnitude activations were much more polysemantic.

Intriguingly, that description closely matches what researchers have found in previous work on language models 

If neural networks can perform computation in superposition, a natural question is to ask how exactly they're doing so. What does that look like mechanically, in terms of the weights? In this subsection, we'll (mostly) work through one such model and see an interesting motif of asymmetric superposition. (We use the term "motif" in the [sense](https://distill.pub/2020/circuits/zoom-in/#claim-2-motifs) of the original circuit thread, inspired by its use in systems biology 

The model we're trying to understand is shown below on the left, visualized as a neuron weight stack plot, with features corresponding to colors. The model is only doing a limited amount of superposition, and many of the weights can be understood as simply implementing absolute value in the expected way.

However, there are a few neurons doing something else…

These other neurons implement two instances of asymmetric superposition and inhibition. Each instance consists of two neurons:

One neuron does asymmetric superposition. In normal superposition, one might store features with equal weights (eg. 

To avoid the consequences of that interference, the model has another neuron heavily inhibit the feature in the case where there would have been positive interference. This essentially converts positive interference (which could greatly increase the loss) into negative interference (which has limited consequences due to the output ReLU).

There are a few other weights this doesn't explain. (We believe they're effectively small conditional biases.) But this asymmetric superposition and inhibition pattern appears to be the primary story.

Although superposition is scientifically interesting, much of our interest comes from a pragmatic motivation: we believe that superposition is deeply connected to the challenge of using interpretability to make claims about the safety of AI systems. In particular, it is a clear challenge to the most promising path we see to be able to say that neural networks won't perform certain harmful behaviors or to catch "unknown unknowns" safety problems. This is because superposition is deeply linked to the ability to identify and enumerate over all features in a model, and the ability to enumerate over all features would be a powerful primitive for making claims about model behavior.

We begin this section by describing how "solving superposition" in a certain sense is equivalent to many strong interpretability properties which might be useful for safety. Next, we'll describe three high level strategies one might take to "solving superposition." Finally, we'll describe a few other additional strategic considerations.

We'd like a way to have confidence that models will never do certain behaviors such as "deliberately deceive" or "manipulate." Today, it's unclear how one might show this, but we believe a promising tool would be the ability to identify and enumerate over all features. The ability to have a universal quantifier over the fundamental units of neural network computation is a significant step towards saying that certain types of circuits don't exist.

How does this relate to superposition? It turns out that the ability to enumerate over features is deeply intertwined with superposition. One way to see this is to imagine a neural network with a privileged basis and without superposition (like the monosemantic neurons found in early InceptionV1, e.g. 

For this reason, we'll call any method that gives us the ability to enumerate over features – and equivalently, unfold activations – a "solution to superposition". Any solution is on the table, from creating models that just don't have superposition, to identifying what directions correspond to features after the fact. We'll discuss the space of possibilities shortly.

We've motivated "solving superposition" in terms of feature enumeration, but it's worth noting that it's equivalent to (or necessary for) many other interpretability properties one might care about:

At a very high level, there seem to be three potential approaches to resolving superposition:

Our sense is that all of these approaches are possible if one doesn't care about having a competitive model. For example, we believe it's possible to accomplish any of these for the toy models described in this paper. However, as one starts to consider serious neural networks, let alone modern large language models, all of these approaches begin to look very difficult. We'll outline the challenges we see for each approach in the following sections.

With that said, it's worth highlighting one bright spot before we focus on the challenges. You might have believed that superposition was something you could never fully get rid of, but that doesn't seem to be the case. All our results seem to suggest that superposition and polysemanticity are phases with sharp transitions. That is, there may exist a regime for every model where it has no superposition or polysemanticity. The question is largely whether the cost of getting rid of or otherwise resolving superposition is too high.

It's actually quite easy to get rid of superposition in the toy models described in this paper, albeit at the cost of a higher loss. Simply apply an L1 regularization term to the hidden layer activations (i.e. add 

However, it seems likely that models are significantly benefitting from superposition. Roughly, the sparser features are, the more features can be squeezed in per neuron. And many features in language models seem very sparse! For example, language models know about individuals with only modest public presences, such as several of the authors of this paper. Presumably we only occur with frequency significantly less than one in a million tokens. As a result, it may be the case that superposition effectively makes models much bigger.

All of this paints a picture where getting rid of superposition may be fairly achievable, but doing so will have a large performance cost. For a model with a fixed number of neurons, superposition helps – potentially a lot.

But this is only true if the constraint is thought of in terms of neurons. That is, a superposition model with 

One family of models which change the flop-neuron relationship are Mixture of Experts (MoE) models (see review 

It's unclear how far this can be pushed, especially given difficult engineering constraints. But there's an obvious lower bound, which is likely too optimistic but is interesting to think about: what if models only expended flops on neuron activations, and recovered the compute of all non-activating neurons? In this world, it seems unlikely that superposition would be optimal: you could always split a polysemantic neuron into dedicated neurons for each feature with the same cost, except for the cases where there would have been interference that hurt the model anyways. Our preliminary investigations comparing various types of superposition in terms of "loss reduction per activation frequency" seem to suggest that superposition is not optimal on these terms, although it asymptotically becomes as good as dedicated feature dimensions. Another way to think of this is that superposition exploits a gap between the sparsity of neurons and the sparsity of the underlying features; MoE eats that same gap, and so we should expect MoE models to have less superposition.

To be clear, MoE models are already well studied, and we don't think this changes the capabilities case for them. (If anything, superposition offers a theory for why MoE models have not proven more effective for capabilities when the case for them seems so initially compelling!) But if one's goal is to create competitive models that don't have superposition, MoE models become interesting to think about. We don't necessarily think that they specifically are the right path forward – our goal here has been to use them as an example of why we think it remains plausible there may be ways to build competitive superposition-free models.

The opposite strategy of creating a superposition-free model is to take a regular model, which has superposition, and find an overcomplete basis describing how features are embedded after the fact. This appears to be a relatively standard sparse coding (or dictionary learning) problem, where we want to take the activations of neural network layers and find out which directions correspond to features.

The advantage of this is that we don't need to worry about whether we're damaging model performance. On the other hand, many other things are harder:

In addition to approaches which address superposition purely at training time, or purely after the fact, it may be possible to take "hybrid approaches" which do a mixture. For example, even if one can't change models without superposition, it may be possible to produce models with less superposition, which are then easier to decode.

Phase Changes as Cause For Hope. Is totally getting rid of superposition a realistic hope? One could easily imagine a world where it can only be asymptotically reduced, and never fully eliminated. While the results in this paper seem to suggest that superposition is hard to get rid of because it's actually very useful, the upshot of it corresponding to a phase change is that there's a regime where it totally doesn't exist. If we can find a way to push models in the non-superposition regime, it seems likely it can be totally eliminated.

Any superposition-free model would be a powerful tool for research. We believe that most of the research risk is in whether one can make performant superposition free models, rather than whether it's possible to make superposition free models at all. Of course, ultimately, we need to make performant models. But a non-performant superposition free model could still be a very useful research tool for studying superposition in normal models. At present, it's challenging to study superposition in models because we have no ground truth for what the features are. (This is also the reason why the toy models described in this paper can be studied – we do know what the features are!) If we had a superposition-free model, we may be able to use it as a ground truth to study superposition in regular models.

Local bases are not enough. Earlier, when we considered [the geometry of non-uniform superposition](https://transformer-circuits.pub#geometry-non-uniform), we observed that models often form local orthogonal bases, where co-occurring features are orthogonal. This suggests a strategy for locally understanding models on sufficiently narrow sub-distributions. However, if our goal is to eventually make useful statements about the safety of models, we need mechanistic accounts that hold for the full distribution (and off distribution). Local bases seem unlikely to give this to us.

Why are we interested in toy models? We believe they are useful proxies for studying the superposition we suspect might exist in real neural networks. But how can we know if they're actually a useful toy model? Our best validation is whether their predictions are consistent with empirical observations regarding polysemanticity. To the best of our knowledge they are. In particular:

This doesn't mean that everything about our toy model reflects real neural networks. Our intuition is that some of the phenomena we observe (superposition, monosemantic vs polysemantic neurons, perhaps the relationship to adversarial examples) are likely to generalize, while other phenomena (especially the geometry and learning dynamics results) are much more uncertain.

This paper has shown that the superposition hypothesis is true in certain toy models. But if anything, we're left with many more questions about it than we had at the start. In this final section, we review some of the questions which strike us as most important: what do we know, and would we like for future work to clarify?

Our work is inspired by research exploring the features that naturally occur in neural networks. Many models form at least some interpretable features. Word embeddings have semantic directions (see 

The earliest reference to superposition in artificial neural networks that we're aware of is Arora et al.'s work 

In parallel with this, investigations of individual neurons in models with privileged bases were beginning to grapple with "polysemantic" neurons which respond to unrelated inputs 

Separate from all of this, Cheung et al. ["almost orthogonal bases" experiment](https://transformer-circuits.pub#geometry-local-bases) we considered above.

The goal of learning disentangled representations arises from Bengio et al.'s influential position paper on representation learning 

Concretely, disentanglement research often explores whether one can train a VAE or GAN where basis dimensions correspond to the major features one might use to describe the problem (e.g. rotation, lighting, gender… as relevant). Early work often focused on semi-supervised approaches where the features were known in advance, but fully unsupervised approaches started to develop around 2016 

Put another way, the goal of disentanglement might be described as imposing a strong privileged basis on representations which are rotationally invariant by default. This helps get at ways in which the questions of polysemanticity and superposition are a bit different from disentanglement. Consider that when we deal with neurons, rather than embeddings, we have a privileged basis by default. It varies by model, but many neurons just cleanly respond to features. This means that polysemanticity arises as a kind of anomalous behavior, and superposition arises as a hypothesis for explaining it. The question then isn't how to impose a privileged basis, but how to remove superposition as a fundamental problem to accessing features.

Of course, if the superposition hypothesis is true, there are still a number of connections to disentanglement. On the one hand, it seems likely superposition occurs in the latent spaces of generative models, even though that isn't an area we've investigated. If so, it may be that superposition is a major reason why disentanglement is difficult. Superposition may allow generative models to be much more effective than they would otherwise be without. Put another way, disentanglement often assumes a small number of important latent variables to explain the data. There are clearly examples of such variables, like the orientation of objects – but what if a large number of sparse, rare, individually unimportant features are collectively very important? Superposition would be the natural way for models to represent this.

The toy problems we consider are quite similar to the problems considered in the field of [compressed sensing](https://en.wikipedia.org/wiki/Compressed_sensing), which is also known as compressive sensing and sparse recovery. However, there are some important differences:

In general, our toy model is solving a similar problem using less powerful methods than compressed sensing algorithms, especially because the computational model is so much more restricted (to just a single linear transformation and a non-linearity) compared to the arbitrary computation that might be used by a compressed sensing algorithm.

As a result, compressed sensing lower bounds—which give lower bounds on the dimension of the embedding such that recovery is still possible—can be interpreted as giving an upper bound on the amount of superposition in our toy model. In particular, in various compressed sensing settings, one can recover an [in the appendix](https://transformer-circuits.pub#compressed-sensing-connection-proof).

At first, this bound appears to allow a number of features that is exponential in 

A striking parallel between our toy model and compressed sensing is the existence of phase changes.

Another interesting line of work has tried to build useful sparse recovery algorithms using neural networks 

[Sparse Coding](https://en.wikipedia.org/wiki/Sparse_dictionary_learning) studies the problem of finding a sparse representation of dense data. One can think of it as being like compressed sensing, except the matrix projecting sparse vectors into the lower dimensional space is also unknown. This topic goes by many different names including sparse coding (most common in neuroscience), dictionary learning (in computer science), and sparse frame design (in mathematics). For a general introduction, we refer readers to a textbook by Michael Elad 

Classic sparse coding algorithms take an expectation-maximization approach (this includes Olshausen et al's early work 

From our perspective, sparse coding is interesting because it's probably the most natural mathematical formulation of trying to "solve superposition" by discovering which directions correspond to features.[research](https://www.lesswrong.com/posts/z6QQJbtpkEAX3Aojj/interim-research-report-taking-features-out-of-superposition) by Sharkey et al [Approach 2: Finding an Overcomplete Basis](https://transformer-circuits.pub#strategic-approach-overcomplete) for more discussion.

Our work explores representations in artificial “neurons”. Neuroscientists study similar questions in biological neurons. There are a variety of theories for how information could be encoded by a group of neurons. At one extreme is a local code, in which every individual stimulus is represented by a separate neuron. At the other extreme is a maximally-dense distributed code, in which the information-theoretic capacity of the population is fully utilized, and every neuron in the population plays a necessary role in representing every input.

One challenge in comparing our work with the neuroscience literature is that a “distributed representation” seems to mean different things. Consider an overly-simplified example of a population of neurons, each taking a binary value of active or inactive, and a stimulus set of sixteen items: four shapes, with four colors  (example borrowed from 

Decomposability doesn’t necessarily mean each feature gets its own neuron. Instead, it could be that each feature corresponds to a “direction in activation-space”

Any decomposable linear code that uses orthogonal feature vectors is functionally equivalent from the viewpoint of a linear readout. So, a code can both be “maximally distributed” — in the sense that every neuron participates in representing every input, making each neuron extremely polysemantic — and also have no more features than it has dimensions. In this conception, it’s clear that a code can be fully “distributed” and also have no superposition.

A notable difference between our work, and the neuroscience literature we have encountered, is that we consider as a central concept the likelihood that features co-occur with some probability.

One hypothesis in neuroscience is that highly compressed representations might have an important use in long-range communication between brain areas[the absolute value toy model](https://transformer-circuits.pub#superposition-computation) shows that networks can do useful computation even under a code with a moderate degree of superposition. This suggests that all neural codes, not just those used for efficient communication, could plausibly be “compressed” to some degree; the regional code might not necessarily need to be decompressed to a fully sparse one.

It's worth noting that the term "distributed representation" is also used in deep learning, and has the same ambiguities of meaning there. Our sense is that some influential early works (e.g. 

After publishing the original version of this paper, a number of readers generously brought to our attention additional connections to prior work. We don't have a sufficiently deep understanding of this work to offer a detailed review, but we offer a brief overview below:

Inspired by the original [Circuits Thread](https://distill.pub/2020/circuits/) and [Distill's Discussion Article experiment](https://distill.pub/2019/advex-bugs-discussion/), the authors invited several external researchers who we had previously discussed our preliminary results with to comment on this work. Their comments are included below.

[Tom McGrath](https://tommcgrath.github.io/) is a research scientist at [DeepMind](https://www.deepmind.com/).

The results in this paper are an important contribution – they really further our theoretical understanding of a phenomenon that may be central to interpretability research and understanding network representations more generally. It’s surprising that such simple settings can produce these rich phenomena. We’ve reproduced the experiments in the [Demonstrating Superposition](https://transformer-circuits.pub#demonstrating)  and [Superposition as a Phase Change](https://transformer-circuits.pub#phase-change) sections and have a minor additional result to contribute.

It is possible to exactly solve the expected loss for the [ReLU output toy model](https://transformer-circuits.pub#demonstrating-setup-model) (ignoring bias terms). The derivation is mathematically simple but somewhat long-winded: the ‘tricks’ are to (1) represent the sparse portion of the input distribution with delta functions, and (2) replace the ReLu with a restriction of the domain of integration:

Making this substitution renders the integral analytically tractable, which allows us to plot the full loss surface and solve for the loss minima directly. We show some example loss surfaces below:

Although many of these loss surfaces (Figure 1a, 1b) have minima qualitatively similar to one of the network weights used in the section [Superposition as a Phase Change](https://transformer-circuits.pub#phase-change), we also find a new phase where [The Geometry of Superposition – Collapsing of Correlated Features](https://transformer-circuits.pub#geometry-collapsing), but occurs without the features being correlated!) Further, although the solutions we find are often qualitatively similar to the weights used in [Superposition as a Phase Change](https://transformer-circuits.pub#phase-change), they can be quantitatively different, as Figure 1a shows. The transition from Figure 1a to Figure 1b is continuous: the minima moves smoothly in weight space as the degree of sparsity alters. This explains the ‘blurry’ region around the triple point in the phase diagram.

As Figure 1c shows, some combinations of sparsity and relative feature importance lead to loss surfaces with two minima (once the symmetry [Discrete "Energy Level" Jumps phenomenon](https://transformer-circuits.pub#learning-jumps) as solutions hop between minima. In some cases (e.g. when parameters approach those needed for a phase transition) the global minimum can have a considerably smaller basin of attraction than local minima. The transition between the antipodal and confused-feature solutions appears to be discontinuous.

Original Authors' Response:  This closed form analysis of the 

[Jeffrey Wu](https://www.wuthejeff.com/) and [Dan Mossing](https://dmossing.github.io/) are members of the Alignment team at [OpenAI](https://openai.com/).

We are very excited about these toy models of polysemanticity. This work sits at a rare intersection of being plausibly very important for training more interpretable models and being very simple and elegant. The results have been surprisingly easy to replicate – we have reproduced (with very little fuss) plots similar to those in the [Demonstrating Superposition – Basic Results](https://transformer-circuits.pub#demonstrating-basic-results), [Geometry – Feature Dimensionality](https://transformer-circuits.pub#geometry-dimensionality), and [Learning Dynamics – Discrete "Energy Level" Jumps](https://transformer-circuits.pub#learning-jumps) sections.

Original Authors' Response: We really appreciate this replication of our basic results. Some of our findings were quite surprising to us, and this gives us more confidence that they aren't the result of an idiosyncratic quirk or bug in our implementations.

[Spencer Becker-Kahn](https://spencerbeckerkahn.wixsite.com/stbk) is a senior research scholar at the Future of Humanity Institute and a SERI Machine Learning Alignment Theory Scholar.

After seeing preliminary results, I independently replicated some of the key graphs from [Demonstrating Superposition – Basic Results](https://transformer-circuits.pub#demonstrating-basic-results) and, using very small toy models, produced a series of plots consistent with the conceptual picture emerging in [Geometry – Feature Dimensionality](https://transformer-circuits.pub#geometry-dimensionality) and [Superposition and Learning Dynamics](https://transformer-circuits.pub#learning).

See also [Twitter thread](https://twitter.com/sbeckerkahn/status/1572255478666649602).

[Adam Jermyn](https://adamjermyn.com/) is an independent researcher focused on AI alignment and interpretability. He was previously a Research Fellow at the Flatiron Institute’s Center for Computational Astrophysics. [Evan Hubinger](https://evhub.github.io/) is a Research Fellow at MIRI. [Nicholas Schiefer](https://nicholasschiefer.com/) is a member of the technical staff at Anthropic and an author of the original paper.

Inspired by the results in this paper and the [previous paper](https://transformer-circuits.pub/2022/solu/index.html) introducing the SoLU activation, we have been investigating whether changes to the model architecture or training process can reduce superposition in toy models. After replicating several of these results independently, we made the following extensions in that direction:

At least in some limits, this suggests there may not be a price to be paid for monosemanticity. Detailed results can be found in our paper, Engineering Monosemanticity in Toy Models ([Alignment Forum](https://www.alignmentforum.org/posts/LvznjZuygoeoTpSE6/engineering-monosemanticity-in-toy-models), [ArXiV](https://arxiv.org/pdf/2211.09169.pdf)).

Tom Henighan and Chris Olah are authors of the original paper.

In the ["Feature Dimensionality" section](https://transformer-circuits.pub#geometry-dimensionality), we found that features organized into clean polytopes when there are more features than can be easily represented in the embedding dimensions. 

We briefly investigated this further and found that the number of features competing to be represented significantly influences this phenomenon. Cleaner structure often seems to emerge when there is more "pressure" – more features competing to be represented. This is especially true at high sparsity levels. Additionally, training longer seems to also produce cleaner structure.

More investigation would be needed to really understand this phenomenon.

[Marius Hobbhahn](https://www.mariushobbhahn.com/) is a PhD student at the University of Tuebingen.

I replicated the [“Basic Results”](https://transformer-circuits.pub#demonstrating-basic-results) in [Section 2 (“Demonstrating Superposition”)](https://transformer-circuits.pub#demonstrating) and all of [Section 7 (“Superposition in a Privileged Basis”)](https://transformer-circuits.pub#privileged-basis) of the “Toy Models of Superposition” paper. All of my findings are identical to the ones described in the paper. I replicated most findings in the follow-up [“Superposition, Memorization, and Double Descent”](https://transformer-circuits.pub/2023/toy-double-descent/index.html) paper.

The details of my replication can be found in my write-up [“More Findings on Memorization and Double Descent”](https://www.lesswrong.com/posts/KzwB4ovzrZ8DYWgpw/more-findings-on-memorization-and-double-descent).

Lee Sharkey, Dan Braun, and Beren Millidge are researchers at Conjecture.

The results from this paper, and the strategic picture it paints, inspired our [preliminary follow-up work](https://www.alignmentforum.org/posts/z6QQJbtpkEAX3Aojj/interim-research-report-taking-features-out-of-superposition) that aimed to address some of the challenges described in the section titled '[Approach 2: Finding an overcomplete basis](https://transformer-circuits.pub/2022/toy_model/index.html#comment-jermyn:~:text=APPROACH%202%3A%20FINDING%20AN%20OVERCOMPLETE%20BASIS)'. 

Before studying the activations of real neural networks, where we're not sure what the 'ground truth' features are, we studied a toy example. We generated a set of toy ground truth features and created a dataset using sparse combinations of them. We found that a one-layer sparse autoencoder with an 

For the toy dataset, we knew how many ground truth features there were. But we ultimately want to count the features used by real neural networks, where the number of features is unknown. We explored three ways to count the features in the toy dataset: a) Counting dead neurons in the autoencoders; b) looking at autoencoder losses; and c) comparing the features learned by autoencoders of different sizes. We found indications that these methods might be suitable to count the number of features in superposition in real neural data.

We also applied our method to real activations from a small language model. Our initial, preliminary investigations led to inconclusive results, possibly resulting from having used autoencoders that were either too small or undertrained. At the time of writing, investigations are ongoing.

Neel Nanda is an external researcher in mechanistic interpretability. This is a description of his blog post, [Actually, Othello-GPT Has A Linear Emergent World Representation](https://neelnanda.io/othello).

I describe a natural experiment of the [linear representation hypothesis](https://transformer-circuits.pub#motivation-directions) described in this paper – the idea that features correspond to directions in neural networks.

Background: Martin Wattenberg (an author on this paper) and colleagues [recently found](https://arxiv.org/pdf/2210.13382.pdf) that, if you train transformer language model trained to predict the next token in synthetic Othello games (where each move is a randomly chosen legal move), it forms an emergent model of the board state (despite only being trained to predict the next move!). They showed that the state of the board (whether each cell was empty, black or white) could be recovered with high accuracy by a one hidden layer MLP probe. They further showed that you could use the world model to causally intervene on the model’s residual stream. By choosing another board state, and changing the residual stream (with gradient descent) such that the probe indicates that new board state, they caused the model to output legal moves in the new board state, even if the edited board state was impossible to reach via legal Othello play!

Pre-Registered Hypothesis: The probing and causal intervention together provided strong evidence that the model had learned to represent features corresponding to the state of each square on the board. Yet, notably, linear probes were not able to recover the board state. Since linear features should be recoverable with a one-layer probe, and the causal intervention suggests the model both computes and uses the board state, this seemed like significant evidence against the linear representation hypothesis.

However, Chris Olah (an author on this paper) argued the model might still be representing features linearly if it used a different set of features, and that the probe and causal intervention may be picking up on this different set of features. This created an informal pre-registered prediction of the hypothesis which was contrary to the evidence at the time.

Results: I independently came to the same conclusion as Chris and investigated [the Othello playing model](https://www.neelnanda.io/othello). I found that the model does form an emergent model of the board state that is linearly represented and can be extracted with a linear probe. But that as the model plays both black and white moves, the model represents the state of a cell as whether it has “my colour” vs “opponent’s colour”. Further, I found circumstantial evidence that these features are used by the model, as we can linearly intervene on the residual stream using the directions given by the probe to edit the represented board state, and the model plays legal moves in the new board state.

I consider these results to be notable, as the paper’s results provided evidence against the linear representation hypothesis and the hypothesis faced genuine risk of falsification. And the hypothesis made non-trivial predictions that were contrary to where the evidence pointed, but these turned out to be true. This is both a proof of concept that there are underlying principles of neural networks which have predictive power about models, and a natural experiment supporting the linear representation hypothesis.

I think that there is further work interpreting the Othello playing model that could test other hypotheses in this paper and our broader conceptual frameworks about neural networks and transformers, such as by looking for monosemantic vs superposed neurons in its MLP layers. The model is both complex enough to be interesting and expose principles of how transformers learn algorithms, yet the algorithmic nature of the task and existence of the probe suggests that finding circuits will be tractable. I elaborate on what I consider promising directions of future work in [a follow-up post](https://www.alignmentforum.org/posts/qgK7smTvJ4DB8rZ6h/othello-gpt-future-work-i-am-excited-about).

[Fred Zhang](https://fredzhang.me/) is a PhD student in the Theory Group of the EECS Department at UC Berkeley.

In the [Geometry of Superposition](https://transformer-circuits.pub#geometry) section, the paper defines a notion of feature dimensionality, 

Following this definition, the paper makes the remark that "Empirically, it seems that the dimensionality of all features add up to the number of embedding dimensions when the features are 'packed efficiently' in some sense." In this comment, I point out a natural, theoretical explanation of this observation. The argument is via the notion of leverage score in matrix approximation. I’ll define it first, then explain how it connects to feature dimensionality.

At a conceptual level, leverage score is a measure of the importance of a row in composing the row space of a matrix. For instance, if a row is orthogonal to all other rows, its leverage score is 1, meaning that it’s maximally important. This is natural, since removing it would decrease the rank of the matrix and completely change the row space. Formally, if 

Notice that the denominator term equals 

Three quick remarks on this definition:

Returning to my main point, another nice fact about leverage score is they sum up to the rank of the matrix. In the tall and thin case above, they sum up to d (if the matrix is full-rank). Given that, it is natural this paper makes the empirical observation that the sum of 

We provide a notebook to reproduce some of the core diagrams in this article [here](https://colab.research.google.com/github/anthropics/toy-models-of-superposition/blob/main/toy_models.ipynb). (It isn't comprehensive, since we needed to rewrite code for our experiments to run outside our codebase.) We provide a [separate notebook](https://github.com/wattenberg/superposition/blob/main/Exploring_Exact_Toy_Models.ipynb) for the theoretical phase change diagrams.

Note that the reproductions by other researchers mentioned in comments above were not based on this code, but are instead fully independent replications with clean code from the description in an early draft of this article.

We're extremely grateful to a number of colleagues across several organizations for their invaluable support in our writing of this paper.

Jeff Wu, Daniel Mossing, Tom McGrath, and Kshitij Sachan did independent replications of many of our experiments, greatly increasing our confidence in our results. Kshitij Sachan's and Tom McGrath's additional investigations and insightful questions both pushed us to clarify our understanding of the superposition phase change (both as reflected in this paper, and in further understanding which we learned from them not captured here). Buck Shlegeris, Adam Scherlis, and Adam Jermyn shared valuable insights into the mathematical nature of the toy problem and related work. Adam Jermyn also coined the term "virtual neurons."

Gabriel Goh, Neel Nanda, Vladimir Mikulik, and Nick Cammarata gave detailed feedback which improved the paper, in addition to being motivating. Alex Dimakis, Piotr Indyk, Dan Yamins generously took time to discuss these results with us and give advice on how they might connect to their area of expertise. Finally, we benefited from the feedback and comments of James Bradbury, Sebastian Farquhar, Shan Carter, Patrick Mineault, Alex Tamkin, Paul Christiano, Evan Hubinger, Ian McKenzie, and Sid Black. We're additionally grateful to Trenton Bricken and Manjari Narayan for referring us to valuable related work we originally missed. Thanks to Ken Kahn for typo corrections.

Finally, we're very grateful to all our colleagues at Anthropic for their advice and support: Daniela Amodei, Jack Clark, Tom Brown, Ben Mann, Nick Joseph, Danny Hernandez, Amanda Askell, Kamal Ndousse, Andy Jones, Timothy Telleen-Lawton, Anna Chen, Yuntao Bai, Jeffrey Ladish, Deep Ganguli, Liane Lovitt, Nova DasSarma, Jia Yuan Loke, Jackson Kernion, Tom Conerly, Scott Johnston, Jamie Kerr, Sheer El Showk, Stanislav Fort, Rebecca Raible, Saurav Kadavath, Rune Kvist, Jarrah Bloomfield, Eli Tran-Johnson, Rob Gilson, Guro Khundadze, Filipe Dobreira, Ethan Perez, Sam Bowman, Sam Ringer, Sebastian Conybeare, Jeeyoon Hyun, Michael Sellitto, Jared Mueller, Joshua Landau, Cameron McKinnon, Sandipan Kundu, Jasmine Brazilek, Da Yan, Robin Larson, Noemí Mercado, Anna Goldie, Azalia Mirhoseini, Jennifer Zhou, Erick Galankin, James Sully, Dustin Li, James Landis.

Basic Results – The basic toy model results demonstrating the existence of superposition were done by Nelson Elhage and Chris Olah. Chris suggested the toy model and Nelson ran the experiments.

Phase Change – Chris Olah ran the empirical phase change experiments, with help from Nelson Elhage. Martin Wattenberg introduced the theoretical model where exact losses for specific weight configurations can be computed.

Geometry – The uniform superposition geometry results were discovered by Nelson Elhage and Nicholas Schiefer, with help from Chris Olah. Nelson discovered the original 

Learning Dynamics – Nelson Elhage discovered the "energy level jump" phenomenon, in collaboration with Nicholas Schiefer and Chris Olah. Martin Wattenberg discovered the "geometric transformations" phenomenon.

Adversarial Examples – Chris Olah and Catherine Olsson found evidence of a connection between superposition and adversarial examples.

Superposition with a Privileged Basis / Doing Computation – Chris Olah did the basic investigation of superposition in a privileged basis. Nelson Elhage, with help from Chris, investigated the "absolute value" model which provided a more principled demonstration of superposition and showed that computation could be done while in superposition. Nelson discovered the "asymmetric superposition" motif.

Theory – The theoretical picture articulated over the course of this paper (especially in the "mathematical understanding" section) was developed in conversations between all authors, but especially Chris Olah, Jared Kaplan, Martin Wattenberg, Nelson Elhage, Tristan Hume, Tom Henighan, Catherine Olsson, Nicholas Schiefer, Dawn Drain, Shauna Kravec, Roger Grosse, Robert Lasenby, and Sam McCandlish. Jared introduced the strategy of rewriting the loss by grouping terms with the number of active features. Both Jared and Martin independently noticed the value of investigating the 

Strategic Picture – The strategic picture articulated in this paper – What does superposition mean for interpretability and safety? What would a suitable solution be? How might one solve it? – developed in extensive conversations between authors, and in particular Chris Olah, Tristan Hume, Nelson Elhage, Dario Amodei, Jared Kaplan. Nelson Elhage recognized the potential importance of "enumerative safety", further articulated by Dario. Tristan brainstormed extensively about ways one might solve superposition and pushed Chris on this topic.

Writing – The paper was primarily drafted by Chris Olah, with some sections by Nelson Elhage, Tristan Hume, Martin Wattenberg, and Catherine Olsson. All authors contributed to editing, with particularly significant contributions from Zac Hatfield Dodds, Robert Lasenby, Kipply Chen, and Roger Grosse.

Illustration – The paper was primarily illustrated by Chris Olah, with help from Tristan Hume, Nelson Elhage, and Catherine Olsson.

Please cite as:

Elhage, et al., "Toy Models of Superposition", Transformer Circuits Thread, 2022.

BibTeX Citation:

```
@article{elhage2022superposition,
   title={Toy Models of Superposition},
   author={Elhage, Nelson and Hume, Tristan and Olsson, Catherine and Schiefer, Nicholas and Henighan, Tom and Kravec, Shauna and Hatfield-Dodds, Zac and Lasenby, Robert and Drain, Dawn and Chen, Carol and Grosse, Roger and McCandlish, Sam and Kaplan, Jared and Amodei, Dario and Wattenberg, Martin and Olah, Christopher},
   year={2022},
   journal={Transformer Circuits Thread},
   note={https://transformer-circuits.pub/2022/toy_model/index.html}
}
```
This paper focuses on the assumption that representations are linear. But what if models don’t use linear feature directions to represent information? What might such a thing concretely look like?

Neural networks have nonlinearities that make it theoretically possible to compress information even more compactly than a linear superposition. There are reasons we think models are unlikely to pervasively use nonlinear compression schemes:

Regardless of whether large models end up using nonlinear compression, it should be possible to view directions being used with nonlinear compression as linear feature directions and reverse engineer the computation being used for compression like any other circuit. If this kind of encoding is pervasive throughout the network then it may merit some kind of automated decoding. It shouldn’t pose a fundamental challenge to interpretability unless the model learns a scheme for doing complex computation while staying in a complicated nonlinear representation, which we suspect is unlikely.

To help provide intuition, the simplest example of what a nonlinear compression scheme might look like is compressing two [0,1) dimensions 

This works by quantizing the 

We can compare the mean squared error loss on random uniform dense values of 

Here, we formalize the relationship between a compressed sensing lower bound and the toy model.

Let 

We derive the following theorem:

Theorem 1. Suppose that the toy model recovers all 

We prove this result by framing our toy model as a compressed sensing algorithm. The primary barrier to doing so is that our optimization only searches for vectors that are close in 

Lemma 1. Suppose that we have a toy model 

Proof. We construct 

Lastly, we use the deterministic compressed sensing lower bound of Do Ba, Indyk, Price, and Woodruff 


Theorem 2 (Corollary 3.1 in 

for an approximation factor 

Theorem 1 follows directly from Lemma 1 and Theorem 2.

## Replication & Forthcoming Paper

Kshitij Sachan is a research intern at Redwood Research.

Redwood Research has been working on toy models of polysemanticity, inspired by Anthropic's work. We plan to separately publish our results, and during our research we replicated many of the experiments in this paper. Specifically, we replicated all plots in the Demonstrating Superposition and Superposition as a Phase Change sections (visualizations of the relu models with different sparsities and the phase diagrams) as well as the plot in The Geometry of Superposition – Uniform Superposition. We found the phase diagrams look quite different depending on the activation function, suggesting that in this toy model some activation functions induce more polysemanticity than others.

Original Authors' Response: Redwood's further analysis of the superposition phase change significantly advanced our own understanding of the issue – we're very excited for their analysis to be shared with the world. We also appreciate the independent replication of our basic results.

Update: The research by Redwood mentioned in the previous comment, Polysemanticity and Capacity in Neural Networks (Alignment Forum, Arxiv) is out! They study a slightly different toy model, and get some really interesting results. Highlights include analytical traction on understanding a variant of the toy model, understanding superposition in terms of constrained optimization, and analysis of the role different activation functions play.
