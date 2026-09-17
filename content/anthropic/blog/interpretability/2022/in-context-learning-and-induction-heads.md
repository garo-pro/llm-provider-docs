Title: In-context Learning and Induction Heads

URL Source: https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads

Markdown Content:
As Transformer generative models continue to scale and gain increasing real world use 

In the past, mechanistic interpretability has largely focused on CNN vision models[presented](https://transformer-circuits.pub/2021/framework/index.html) some very preliminary progress on mechanistic interpretability for Transformer language models`A`), find the token that came after it last time (call it `B`), and then predict that the same completion will occur again (e.g. forming the sequence `[A][B] … [A] → [B]`). In other words, induction heads “complete the pattern” by copying and completing sequences that have occurred before. Mechanically, induction heads in our models are implemented by a circuit of two attention heads: the first head is a “previous token head” which copies information from the previous token into the next token, while the second head (the actual “induction head”) uses that information to find tokens preceded by the present token. For 2-layer attention-only models,

Ultimately, however, our goal is to reverse-engineer frontier language models (which often contain hundreds of layers and billions or trillions of parameters), not merely 2-layer attention-only models. Unfortunately, both the presence of many layers, and the presence of MLPs, makes it much more difficult to mathematically pin down the precise circuitry of these models. However, a different approach is possible: by empirically observing, perturbing, and studying the learning process and the formation of various structures, we can try to assemble an indirect case for what might be happening mechanistically inside the network. This is somewhat similar to how a neuroscientist might gain understanding of how part of the brain functions by looking at neural development over time, studying patients with an injury to that part of the brain, perturbing brain function in animals, or looking at a select small number of relevant neurons.

In this paper, we take the first preliminary steps towards building such an indirect case. In particular, we present preliminary and indirect evidence for a tantalizing hypothesis: that induction heads might constitute the mechanism for the actual majority of all in-context learning in large transformer models. Specifically, the thesis is that there are circuits which have the same or similar mechanism to the 2-layer induction heads and which perform a “fuzzy” or “nearest neighbor” version of pattern completion, completing `[A*][B*] … [A] → [B]` , where  `A* ≈ A` and `B* ≈ B` are similar in some space; and furthermore, that these circuits implement most in-context learning in large models.

The primary way in which we obtain this evidence is via discovery and study of a phase change that occurs early in training for language models of every size (provided they have more than one layer), and which is visible as a bump in the training loss. During this phase change, the majority of in-context learning ability (as measured by difference in loss between tokens early and late in the sequence) is acquired, and simultaneously induction heads form within the model that are capable of implementing fairly abstract and fuzzy versions of pattern completion. We study this connection in detail to try to establish that it is causal, including showing that if we perturb the transformer architecture in a way that causes the induction bump to occur in a different place in training, then the formation of induction heads as well as formation of in-context learning simultaneously move along with it.

Specifically, the paper presents six complementary lines of evidence arguing that induction heads may be the mechanistic source of general in-context learning in transformer models of any size:

Together the claims establish a circumstantial case that induction heads might be responsible for the majority of in-context learning in state-of-the-art transformer models. We emphasize that our results here are only the beginnings of evidence for such a case, and that like any empirical or interventional study, a large number of subtle confounds or alternative hypotheses are possible – which we discuss in the relevant sections. But we considered these results worth reporting, both because future work could build on our results to establish the claim more firmly, and because this kind of indirect evidence is likely to be common in interpretability as it advances, so we’d like to establish a norm of reporting it even when it is not fully conclusive.

Finally, in addition to being instrumental for tying induction heads to in-context learning, the phase change may have relevance to safety in its own right. Neural network capabilities — such as multi-digit addition — are known to sometimes abruptly form or change as models train or increase in scale

The rest of the paper is organized as follows. We start by clarifying several key concepts and definitions, including in-context learning, induction heads, and a “per-token loss analysis” method we use throughout. We then present the 6 arguments one by one, drawing on evidence from analysis of 34 transformers over the course of training, including more than 50,000 attention head ablations (the data of which is shown in the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table)). We then discuss some unexplained “curiosities” in our findings, as well as reviewing related work.

In modern language models, tokens later in the context are easier to predict than tokens earlier in the context. As the context gets longer, loss goes down. In some sense this is just what a sequence model is designed to do (use earlier elements in the sequence to predict later ones), but as the ability to predict later tokens from earlier ones gets better, it can increasingly be used in interesting ways (such as specifying tasks, giving instructions, or asking the model to match a pattern) that suggest it can usefully be thought of as a phenomenon of its own. When thought of in this way, it is usually referred to as in-context learning.

Emergent in-context learning was noted in GPT-2

At least two importantly different ways of conceptualizing and measuring in-context learning exist in the literature. The first conception, represented in Brown et al., focuses on few-shot learning

The “few-shot learning” conception of in-context learning has tended to receive greater community attention. The ability to do many different tasks with one large model, even without further fine-tuning, is a notable change to the basic economics of model training. Moreover, it gives evidence of wide-ranging general capabilities and the ability to adapt on the fly, which nudges us to re-examine what it means for a model to “understand” or to “reason”.

However, for the purposes of this work, we focus instead on the Kaplan et al. conception: decreasing loss at increasing token indices. We do so because it's a more general framing of the phenomenon than “few-shot learning”. A drawback of this definition is it fails to isolate specific behaviors of interest. At the same time, it allows us to measure models’ overall ability to learn on-the-fly from the context, without depending on our specific choices of “task”.  We’ll also see that, starting from this definition, we are also able to study a couple classic few-shot learning examples (see [Argument 4](https://transformer-circuits.pub#argument-induction-heads-can-implement-abstract-behaviors)).

Throughout this work we compute a simple heuristic measure of in-context learning:

We chose the 500th and 50th token indices somewhat arbitrarily. The 500th token is near the end of a length-512 context, and the 50th token is far enough into the context that some basic properties of the text have been established (such as language and document type) while still being near the start. We will also show that picking different numbers here does not change our conclusions.

Finally, it is worth noting that in-context learning is of potentially special relevance to safety. In-context learning makes it harder to anticipate how a model might behave after a long context. In the longer run, concepts such as mesa-optimization or inner-alignment

(See [Related Work](https://transformer-circuits.pub#in-context-learning-related-work) for more on in-context learning, and [Discussion](https://transformer-circuits.pub#safety-implications) for more on the connection to safety.)

In our [previous paper](https://transformer-circuits.pub/2021/framework/index.html), we discovered a special kind of attention head – which we named induction heads – in two layer attention-only models. Induction heads are implemented by a circuit consisting of a pair of attention heads in different layers that work together to copy or complete patterns. The first attention head copies information from the previous token into each token. This makes it possible for the second attention head to attend to tokens based on what happened before them, rather than their own content. Specifically, the second head (which we call the "induction head") searches for a previous place in the sequence where the present token `A` occurred and attends to the next token (call it `B`), copying it and causing the model to be more likely to output `B` as the next token. That is, the two heads working together cause the sequence `…[A][B]…[A]` to be more likely to be completed with `[B]`.

Induction heads are named by analogy to inductive reasoning. In inductive reasoning, we might infer that if `A` is followed by `B` earlier in the context, `A` is more likely to be followed by `B` again later in the same context. Induction heads crystallize that inference. They search the context for previous instances of the present token, attend to the token which would come next if the pattern repeated, and increase its probability. Induction heads attend to tokens that would be predicted by basic induction (over the context, rather than over the training data).

Notice that induction heads are implementing a simple algorithm, and are not memorizing a fixed table of n-gram statistics. The rule `[A][B] … [A] → [B]` applies regardless of what `A` and `B` are.`A` and `B`. While some induction heads may specialize on certain kinds of `A`s or `B`s, this significant decoupling of `A` and `B` means that they don't have a fixed table of bigram statistics they can update on, but rather can abstract to new patterns.

Our previous paper focused on a few explorations of induction heads, including showing that these heads occur in 2-layer attention-only models (but not 1-layer models); tracking down how they operate mechanistically as part of our mathematical decomposition of transformers; and making an eigenvalue-based test for detecting their presence. However, we were a bit vague on the exact definition of induction heads: it was more like we found a cluster of behaviors and mechanisms that tended to occur together, and called heads in that cluster “induction heads”.

In this paper our goal is to provide evidence for something more expansive: that induction heads play a major role in general in-context learning (not just literal `[A][B]...[A]→[B]` copying), for large models and not just for small 2-layer attention only models.  To do this clearly and coherently, we need a more precise definition of induction heads. Mechanistic analysis of weights and eigenvalue analysis are much more complicated in large models with MLP’s, so for this paper we choose to define induction heads by their narrow empirical sequence copying behavior (the `[A][B]...[A]→[B]`), and then attempt to show that they (1) also serve a more expansive function that can be tied to in-context learning, and (2) coincide with the mechanistic picture for small models.

Formally, we define an induction head as one which exhibits the following two properties

In other words, induction heads are any heads that empirically increase the likelihood of `[B]` given `[A][B]...[A]` when shown a repeated sequence of completely random tokens. An illustration of induction heads’ behavior is shown here:

Note that, as a consequence, induction heads will tend to be good at repeating sequences wholesale. For example, given “The cat sat on the mat. The cat …”, induction heads will promote the continuation “sat on the mat”. This gives a first hint of how they might be connected to general in-context learning and even few-shot learning: they learn to repeat arbitrary sequences, which is a (simple) form of few-shot learning.

One of the things we’ll be trying to establish is that when induction heads occur in sufficiently large models and operate on sufficiently abstract representations, the very same heads that do this sequence copying also take on a more expanded role of analogical sequence copying or in-context nearest neighbors. By this we mean that they promote sequence completions like `[A*][B*] … [A] → [B]` where `A*` is not exactly the same token as `A` but similar in some embedding space, and also `B` is not exactly the same token as `B*`. For example, `A` and `A*` (as well as B and `B*`) might be the same word in different languages, and the induction head can then translate a sentence word by word by looking for “something like `A`”, finding `A*` followed by `B*`, and then completing with “something like `B*`” (which is `B`). We are not yet able to prove mechanistically that induction heads do this in general, but in [Argument 4](https://transformer-circuits.pub#argument-induction-heads-can-implement-abstract-behaviors) we show empirical examples of induction heads behaving in this way (including on translation), and in [Argument 5](https://transformer-circuits.pub#argument-induction-head-mechanism) we point out that the known copying mechanism of induction heads in small models can be naturally adapted to function in this way.

To better understand how models evolve during training, we analyze what we call the "per-token loss vectors." The core idea traces back to a method used by Erhan et al. 

We start with a collection of models. (In our use, we'll train several different model architectures, saving dozens of “snapshots” of each over the course of training. We’ll use this set of snapshots as our collection of models.) Next, we collect the log-likelihoods each model assigns to a consistent set of 10,000 random tokens, each taken from a different example sequence. We combine these log-likelihoods into a "per-token loss vector" and apply Principal Component Analysis (PCA):

A more detailed discussion of technical details can be found in the [Appendix](https://transformer-circuits.pub#per-token-losses).

By applying this method to snapshots over training for multiple models, we can visualize and compare how different models' training trajectories evolve in terms of their outputs. Since we're using PCA, each direction can be thought of as a vector of log-likelihoods that models are moving along. We particularly focus on the first two principal components, since we can easily visualize those. Of course, models also move in directions not captured by the first two principal components, but it's a useful visualization for capturing the highest-level story of training.

Now we’ll proceed to the main part of the paper, which makes the case that induction heads may provide the primary mechanism for the majority of in-context learning for transformer models in general. As stated in the introduction, this is a very broad hypothesis and much of our evidence is indirect, but nevertheless we believe that all the lines of evidence together make a relatively strong, though not conclusive, case.

Before we go through the arguments, it’s useful to delineate where the evidence is more conclusive vs. less conclusive. This is shown in the table below. For small, attention-only models, we believe we have strong evidence that attention heads are the mechanism for the majority of in-context learning, as we have evidence supported by ablations and mechanistic reverse engineering. Conversely, for all models, we can make a strong case that induction heads play some role in in-context learning, as we can demonstrate examples and show suggestive correlations. However, the larger the models get, the harder it is to establish that induction heads account for the actual majority of in-context learning. For large models with MLP’s, we must therefore rely on mainly correlational evidence, which could be confounded. We explore alternate hypotheses throughout, including at the end of [Argument 1](https://transformer-circuits.pub#assessing-the-evidence) and again briefly in [Argument 6](https://transformer-circuits.pub#argument-extrapolation).

Here is the list of arguments we’ll be making, one per section, repeated from the introduction:

For each argument, we’ll have a similar table to the one in this section, showing the strength of the evidence provided by that claim as it applies to large/small models and some/most of context learning. The table above is the sum of the evidence from all six lines of reasoning.

Our first line of evidence comes from correlating measures of in-context learning and measures of induction head presence over the course of training. Specifically, we observe a tight co-variation between them across dozens of models, of different sizes, trained on different datasets (See [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table) for more on the models in which we observe this co-occurrence.).

The table below summarizes the quality of this evidence for the models we have studied: it applies to both large and small models, and is the expected outcome if induction heads were responsible for the majority of in-context learning, but it is only correlational and so could be confounded (discussed more below).

Our first observation is that if we measure in-context learning for transformer models over the course of training (defined as the 50th token loss minus the 500th token loss as described in [Key Concepts](https://transformer-circuits.pub#in-context-learning-key-concept)), it develops abruptly in a narrow window early in training (roughly 2.5 to 5 billion tokens) and then is constant for the rest of training (see figure below).  Before this window there is less than 0.15 nats of in-context learning, after it there is roughly 0.4 nats, an amount that remains constant for the rest of training and is also constant across many different model sizes (except for the one layer model where not much in-context learning ever forms).  This seems surprising – naively, one might expect in-context learning to improve gradually over training, and improve with larger model sizes,[later discussion](https://transformer-circuits.pub#seemingly-constant) for a way in which a constant 0.4 nats can be viewed as a “larger” improvement for a more-powerful model, because attaining the same magnitude of improvement starting from a better baseline is more challenging.

Although we only show three models above, the pattern holds true very generally: many examples are shown in the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table) later in the paper, including models of varying model architecture and size.

One might wonder if the sudden increase is somehow an artifact of the choice to define in-context learning in terms of the difference between the 500th and 50th tokens. We'll discuss this in more depth [later](https://transformer-circuits.pub#seemingly-constant). But for now, an easy way to see that this is a robust phenomenon is to look at the derivative of loss with respect to the logarithm token index in context. You can think of this as measuring something like "in-context learning per ε% increase in context length." We can visualize this on a 2D plot, where one axis is the amount of training that has elapsed, the other is the token index being predicted. Before the phase change, loss largely stops improving around token 50, but after the phase change, loss continues to improve past that point.

It turns out that a sudden improvement in in-context learning isn't the only thing that changes in this window. If we go through the attention heads of a model and score them for whether they are induction heads (using a prefix matching score which measures their ability to perform the task we used to define induction heads in [Key Concepts](https://transformer-circuits.pub#definition-of-induction-heads)), we find that induction heads form abruptly during exactly the same window where in-context learning develops (figure below). Again we show only a few models here, but a full set is shown in the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table). The exception is the one-layer model, where induction heads never form – just as in-context learning never substantially develops for the one-layer model.

This already strongly suggests some connection between induction heads and in-context learning, but beyond just that, it appears this window is a pivotal point for the training process in general: whatever's occurring is visible as a bump on the training curve (figure below). It is in fact the only place in training where the loss is not convex (monotonically decreasing in slope).

That might not sound significant, but the loss curve is averaging over many thousands of tokens. Many behaviors people find interesting in language models, such as the emergence of arithmetic, would be microscopic on the loss curve. For something to be visible at that scale suggests it's a widespread, major change in model behavior. This shift also appears to be the first point where, at least for small models, the loss curve diverges from a one-layer model – which does not display the bump, just as it does not display the other abrupt changes.

We can also apply principal components analysis (PCA) to the per-token losses, as described in [per-token-loss analysis](https://transformer-circuits.pub#per-token-loss-intro), which allows us to summarize the main dimensions of variation in how several models' predictions vary over the course of training.

Below we show the first two principal components of these models’ predictions, with the golden outline highlighting the same interval shown above, when in-context learning abruptly improved. We see that the training trajectories pivot during exactly the same window where the other changes happen. In some sense, whatever is occurring when in-context learning improves is the primary deviation from the basic trajectory our transformers follow during the course of their training. Once again the only exception is the one-layer model – where induction heads cannot form and in-context learning does not improve.

In summary, the following things all co-occur during the same abrupt window:

Collectively these results suggest that some important transition is happening during the 2.5e9 to 5e9 token window early in training (for large models this is maybe 1–2% of the way through training). We call this transition “the phase change”, in that it’s an abrupt change that alters the model’s behavior and has both macroscopic (loss and in-context learning curves) and microscopic (induction heads) manifestations, perhaps analogous to e.g. ice melting. [curious phenomena](https://transformer-circuits.pub#curiosities) occur during the phase change as well.

A natural explanation would be that for all these models, the induction heads implement in-context learning: their formation is what drives all the other changes observed. To strengthen this hypothesis a bit, we check a few things. First, the window where the phase change happens doesn’t appear to correspond to a scheduled change in learning rate, warmup, or weight decay; there is not some known exogenous factor precipitating everything. Second, we tried out training some of the small models on a different dataset, and we observed the phase change develop in the same way (see [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table) for more details). 

Third, to strengthen the connection a little more, we look qualitatively and anecdotally at what’s going on with the model’s behavior during the phase change. One way to do this is to look at specific tokens the model gets better and worse at predicting. The model's loss is an average of billions of log-likelihood losses for individual tokens. By pulling them apart, we can get a sense for what's changed.

Concretely, let's pick a piece of text – for fun, we'll use the first paragraph of Harry Potter – and look at the differences in log-likelihoods comparing the start and the end of the phase change.[Model Analysis Table](https://transformer-circuits.pub#model-analysis-table)

We can also do this same analysis over the course of model training. The loss curve is the average of millions of per-token loss curves. We can break this apart and look at the loss curves for individual tokens.

In particular, let’s take a look at per-token loss trajectories for two tokens in the first paragraph of Harry Potter. In red, we show a token whose prediction gets dramatically better during the phase change: it’s the last of four tokens in “ The Dursleys”, a sequence that appears several times in the text. In blue, we show a token that gets meaningfully worse during the phase change: it’s the first-ever appearance of “ Mrs Potter”, after both previous instances of “ Mrs” were followed by “ Dursley” instead.

All of this shows that during the phase change, we see exactly the behaviors we’d expect to see if induction heads were indeed contributing the majority of in-context learning.

Despite all of the co-occurrence evidence above (and more in the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table)), the fact remains that we haven’t shown that induction heads are the primary mechanism for in-context learning. We have merely shown that induction heads form at the same time as in-context learning, and that in-context learning does not improve thereafter. There are a number of potential confounds in this story. Below we summarize reasons in favor and against thinking the connection is causal. The argument in favor might look like this:

However, the following issues and confounds suggest caution:

One point worth noting here is that the argument that induction heads account for most in-context learning at the transition point of the phase change is more solid than the argument that they account for most in-context learning at the end of training – a lot could be changing during training even as the in-context learning score remains constant.

One thing that falls short about [Argument 1](https://transformer-circuits.pub#argument-phase-change) is that we’re just observing induction heads and in-context learning co-vary; like any observational study, it is not as convincing as if we actively changed one thing and measured what happened to the other. In this section we do a more “interventional” experiment, in which we change the model architecture in a way that makes induction heads easier to form, and observe the effect on in-context learning. The change makes a bigger difference for smaller models so is more convincing there, but also carries some amount of weight for larger models (see table above).

To design our experiment, we start with the observation (noted in the previous section) that the phase change and the corresponding improvement in in-context learning only occurs in transformers with more than one layer. This is what we'd predict if induction heads were the mechanism for the majority of in-context learning: induction heads require a composition of attention heads, which is only possible with two or more layers.

Of course, the observation about one-layer models is pretty weak evidence by itself. (One could imagine one-layer models being different from models with more layers in all sorts of ways!) But it suggests a more general line of attack. If induction heads are the mechanism behind the large improvement in in-context learning, that makes predictions about the minimum architectural requirements in order to achieve the observed improvement. For a standard transformer, the important thing is to have two attention layers. But that's only because the key vectors need to be a function of the attended token and the token before it.

We define a “smeared key" architecture with a very simple modification that should make it easy for transformers of any depth to express induction heads. In our modified models, for each head 

The results (figure below) are in line with the predictions: when we use the smeared-key architecture, in-context learning does indeed form for one-layer models (when it didn’t before), and it forms earlier for two-layer and larger models.  More such results can be seen in the [model analysis table](https://transformer-circuits.pub#model-analysis-table).

However, we probably shouldn't make too strong an inference about large models on this evidence. This experiment suggests that induction heads are the minimal mechanism for greatly increased in-context learning in transformers. But one could easily imagine that in larger models, this mechanism isn’t the whole story, and also this experiment doesn’t refute the idea of the mechanism of in-context learning changing over the course of training.

For the cases they cover, ablations are by far our strongest evidence. The basic argument is that knocking out induction heads decreases the amount of in-context learning we observe in our models. By “knocking out” we mean that we remove a given attention head from the model at test time, doing a forward pass on the transformer without it. (See [our methods section](https://transformer-circuits.pub#per-token-losses-ablated) for exact details of how the ablation is done.)

The ablations presented below show how attention heads contribute to in-context learning, but we can also do ablations to study how attention heads contribute to the overall behavior change that occurs during the phase change (see the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table)).

In fact, almost all the in-context learning in small attention-only models appears to come from these induction heads! This begins at the start of the phase change, and remains true through the end of training.

Unfortunately, we do not have ablations for our full-scale models.

All things considered, we feel comfortable concluding from this that induction heads are the primary mechanism for in-context learning in small attention-only models, but see this evidence as only suggestive for the MLP case.

All of our previous evidence (in Arguments 1–3) focused on observing or perturbing the connection between induction head formation and macroscopic in-context learning. A totally different angle is to just find examples of induction heads implementing seemingly-difficult in-context learning behaviors; this would make it plausible that induction heads account for the majority of in-context learning. This evidence applies even to the very largest models (and we study up to 12B parameter models), but since it shows only a small number of tasks, it’s only suggestive regarding in-context learning in general.

Recall that we [define](https://transformer-circuits.pub#definition-of-induction-heads) induction heads as heads that empirically copy arbitrary token sequences using a “prefix matching” attention pattern. Our goal is to find heads that meet this definition but also perform more interesting and sophisticated behaviors, essentially showing that induction heads in large models can be “generalizable”.

In this argument, we show some anecdotal examples of induction heads from larger transformers ([our 40-layer model](https://transformer-circuits.pub#full-scale-models) with 13 billion parameters) that exhibit exactly such behaviors – namely literal copying, translation, and a specific type of abstract pattern matching. The behaviors are all of the form `[A*][B*]...[A][B]`, aka the “fuzzy nearest neighbor match” or “find something similar early in the sequence and complete the sequence in analogy”. We verify that these heads score highly on our [“copying” and “prefix matching” evaluations](https://transformer-circuits.pub#head-activation-evaluators) (that is, they increase the probability of the token they attend to, and attend to tokens where the prefix matches the present token on random text), and are thus “induction heads” by our strict empirical [definition](https://transformer-circuits.pub#definition-of-induction-heads), at the same time as they also perform these more sophisticated tasks.

The results of some example heads are shown in the table below, and described in the subsections below.

| Head | Layer Depth | Copying score [(?)](https://transformer-circuits.pub#head-activation-evaluators) | Prefix matching score [(?)](https://transformer-circuits.pub#head-activation-evaluators) | 
| Literal copying head | 21 / 40 | 0.89 | 0.75 | 
| Translation head | 7 / 40 | 0.20 | 0.85 | 
| Pattern-matching head | 8 / 40 | 0.69 | 0.94 | 

We'll start with the simplest case of a head that literally copies repeated text, to get familiar with the visualization interface we're using and the basic dynamics of these heads. We've selected an induction head which seems to perform very basic copying behavior and will look at how it behaves on the first paragraph of Harry Potter. We've repeated the first few sentences afterwards to show the head's behavior on longer segments of repeated text.

The visualization will show two different things:

To start exploring the visualization, we suggest to try hovering your cursor over the second paragraph.

If you explore the visualization, you'll see that the head predicts repeating the names "Dursley" and "Potters"; the phrase "a small son”; and then the entire repeated sentences at the end. In all cases, these successful predictions are made by attending back to a previous instance where this phrase was present in the text.

It's a well-known result that language models can translate between languages. Intriguingly, we've encountered many examples of induction heads that can do translation. Here, we explore a head we found in layer 7 of our 40-layer model, showcasing translation between English, French, and German. (As with the others in this section, this head is also an “induction head” by the same definition we’ve been using all along, because when shown repeated random tokens, it uses a “prefix matching” attention pattern to copy the sequence verbatim.)

Note that the overall attention pattern (in red, top left) is more or less an “off-diagonal”, but it meanders a little bit away from a sharp diagonal. The meandering is because different languages have different word order and token lengths. As this head attends sequentially to past tokens that will semantically come next, the attended token position in the earlier sentences jumps around.

The logit attribution patterns for this head are not perfectly sharp; that is, even in cases where the attention head is attending to the matching word in an earlier language, it does not always directly increase the logit of the corresponding prediction. We would guess that this is because this head’s output needs to be further processed by later layers. However, taken overall, the direct logit attributions show clear evidence of contributing on net to the correct translation.

In this final example, we show an attention head (found at layer 26 of our 40-layer model) which does more complex pattern matching. One might even think of it as learning a simple function in context! (Again, this head also scores highly on our [measurements](https://transformer-circuits.pub#head-activation-evaluators) of “basic” induction behavior when shown repeated random sequences, so it is an induction head by that definition.)

To explore this behavior, we've generated some synthetic text which follows a simple pattern. Each line follows one of four templates, followed by a label for which template it is drawn from. The template is randomly selected, as are the words which fill in the template:

Below, we show how the attention head behaves on this synthetic example. To make the diagram easier to read, we've masked the attention pattern to only show the ":" tokens as the destination, and the logit attribution to only show where the output is the integer tokens.

This head attends back to a previous instance of the correct category more often than not. It often knows to skip over lines where one of the words is identical but the pattern is wrong (such as “January bird” primarily attending to “April fish” and not “grey bird”). This head isn’t perfect at this, but empirically it allocates about 65% of its attention from the colons to the correct positions, when tested on a range of similar problems.

We emphasize again that the attention heads that we described above simultaneously implement both the abstract behaviors that we described, and these very same attention heads (as in, the exact same head in the same layer) also satisfy the [formal definition](https://transformer-circuits.pub#definition-of-induction-heads) of induction head (literal copying of random sequences using prefix matching).  The comparison is not a metaphor or a blurring of the definition: induction heads which are defined by their ability to copy literal sequences turn out to also sometimes match more abstract patterns.  This is what the table at the beginning of the section shows empirically.

But this still leaves the question: why do the same heads that inductively copy random text also exhibit these other behaviors?  One hint is that these behaviors can be seen as “spiritually similar” to copying.  Recall that where an induction head is defined as implementing a rule like `[A][B] … [A] → [B]`, our empirically observed heads also do something like `[A*][B*] … [A] → [B]` where `A*` and `B*` are similar to `A` and `B` in some higher-level representation.  There are several ways these similar behaviors could be connected.  For example, note that the first behavior is a special case of the second, so perhaps induction heads are implementing a more general algorithm that reverts to the special case of copying when given a repeated sequence 

In [Argument 5](https://transformer-circuits.pub#argument-induction-head-mechanism) we’ll strengthen this argument by giving a mechanistic account of how induction heads (when doing simple copying with prefix-matching) attend back to the token that comes next in the pattern, and observe that the actual mechanism they use could naturally generalize to more abstract pattern matching.  Our point in this section is just that it's actually quite natural for these more abstract induction heads to also exhibit the basic copying behaviors underlying our definition.

One of the main reasons we care about whether induction heads drive in-context learning is that we can understand them and so have a path to understanding in-context learning. But we can also turn this around: we can use our understanding of induction heads to make a purely logical argument that they should contribute to in-context learning.

We begin with a semi-empirical argument. Let's take for granted that induction heads behave the way we've described and empirically seen, searching the context for previous examples and copying what happened next. We should expect such a procedure to improve a model's ability to predict tokens later in its context. We're essentially using the previous context as data points for a nearest neighbor algorithm, and nearest neighbors improves as one gives it more data points. Therefore, if induction heads exist as described, they would contribute to in-context learning as we've defined it.

In some sense, this argument is quite strong if we're only arguing that there exist some cases where induction heads contribute to in-context learning. We've seen concrete examples above where induction heads improve token predictions by copying earlier examples. If nothing else, they must help in those cases! And more generally, our definition of induction heads (in terms of their behavior on repeated random sequences) suggests they behave this way quite generally. This argument doesn't say anything about what fraction of in-context learning is performed by induction heads, but it seems like a very strong argument that some is, both in large and small models.

But the really satisfying thing about this line of attack — namely, using our understanding of induction heads to anticipate their impact on in-context learning — is that we can actually drop the dependency on empirical observations of induction head behavior, at the cost of needing to make a more complex argument. In our [previous paper](https://transformer-circuits.pub/2021/framework/index.html), we were able to [reverse engineer](https://transformer-circuits.pub/2021/framework/index.html#how-induction-heads-work) induction heads, showing from the parameter level how they implement induction behavior (and that they should). If we trust this analysis, we can know how induction heads behave without actually running them, and the argument we made in the previous paragraph goes through. Of course, there are some limitations. In the previous paper, we only reverse engineered a single induction head in a small attention-only model, although we can reverse engineer others (and have done so). A bigger issue is that right now we're unable to reverse engineer induction heads in models with MLP layers. But at least in some cases we’ve observed, we can look at the parameters of a transformer and identify induction heads, just as a programmer might identify an algorithm by reading through source code.

We can actually push this argument a bit further in the case of the two-layer attention only transformer we reverse engineered in the previous paper. Not only do we understand the induction heads and know that they should contribute to in-context learning, but there doesn't seem to really be an alternative mechanism that could be driving it.

The following section will briefly summarize reverse engineering induction heads. Note that it relies heavily on linking to our previous paper. We do not expect it to be possible to follow without reading the linked portions. After that, we briefly discuss how the described mechanism could also implement more abstract types of induction head behavior.

Recall from [Key Concepts](https://transformer-circuits.pub#definition-of-induction-heads) that induction heads are defined as heads that exhibit both copying and prefix matching.

Copying is done by the OV ("Output-Value") circuit. One of the defining properties of an induction head is that it copies. Induction heads are not alone in this! Transformers seem to have quite a number of [copying heads](https://transformer-circuits.pub/2021/framework/index.html#copying--primitive-in-context-learning), of which induction heads are a subset. This is done by having a "copying matrix" OV circuit, most easily characterized by its [positive eigenvalues](https://transformer-circuits.pub/2021/framework/index.html#copying-matrix).


Prefix matching is implemented with K-composition (and to a lesser extent Q-composition) in the QK ("Query-Key") Circuit. In order to do [prefix matching](https://transformer-circuits.pub#definition-of-induction-heads), the key vector at the attended token needs to contain information about the preceding tokens — in fact, information about the attended token itself is quite irrelevant to calculating the attention pattern for induction.[our previous work](https://transformer-circuits.pub/2021/framework/index.html#architecture-attn-as-movement), the parts of the head that calculate the attention pattern, and the output if attended to, are separable and are often useful to consider independently.[K-composition](https://transformer-circuits.pub/2021/framework/index.html#three-kinds-of-composition). That is to say that an induction head’s [causes the induction head](https://transformer-circuits.pub/2021/framework/index.html#how-induction-heads-work) to compare the current token with every earlier position's preceding token and look for places where they're similar. More complex QK circuit terms can be used to create induction heads which match on more than just the preceding token.

Combined, these are a detectable mechanism for induction heads. In the small models we studied in our previous paper, [all induction heads have](https://transformer-circuits.pub/2021/framework/index.html#checking-the-mechanistic-theory) the described QK term driving their attention pattern and a positive eigenvalue circuit performing copying.

Some models use a different mechanism to implement induction heads. In GPT-2["pointer-arithmetic" mechanism](https://transformer-circuits.pub/2021/framework/index.html#pointer-arithmetic) for induction heads. This mechanism makes use of the positional embedding and “Q-composition”. In GPT-2, the earlier attention head attends to previous copies of the current token, and its 

What about the induction heads we saw in [Argument 2](https://transformer-circuits.pub#argument-induction-heads-can-implement-abstract-behaviors) with more complex behavior? Can we also reverse engineer them? Do they operate on the same mechanisms? Presently, fully reverse engineering them is beyond us, since they exist in large models with MLPs, which we don't have a strong framework for mechanistically understanding.  However, we hypothesize they're different in two ways: (1) using more complex QK terms rather than matching on just the previous token; and (2) matching and copying more abstract and sophisticated linguistic features, rather than precise tokens. 

When we [first introduced](https://transformer-circuits.pub#definition-of-induction-heads) induction heads, we observed that they could be seen as a kind of "in-context nearest neighbor" algorithm. From this perspective, it seems natural that applying the same mechanism to more abstract features can produce more complex behavior.

This argument is really an extended inference from all the above arguments. Arguments 1–5 present fairly strong evidence that in small transformers (especially small attention-only models), induction heads are responsible for the majority of in-context learning, while for large transformers, the evidence is not as strong. To what extent can one reasonably infer from small models that the same thing is happening in larger models? Obviously this is a matter of judgment.

The measurements in the [model analysis table](https://transformer-circuits.pub#model-analysis-table) look fully analogous between the small attention-only, small models with MLPs, and full-scale model cases. Provided there's more than one layer, all of them go through a phase change. All of them have the same sharp increase in in-context learning, with the same rough amounts before and after the transition. All of them trace similar paths in PCA space. All of them form induction heads.

If things change from the small model case to the large model case, where do they change? And why is there no visible sign of the change in all our measurements?

On the flip side, there are many cases where large models behave very differently than small models (see discussion of phase changes with respect to model size in [Related Work](https://transformer-circuits.pub#related-work)). Extrapolating from small models to models many orders of magnitude larger is something one should do with caution.

The most compelling alternative possibility we see is that other composition mechanisms may also form during the phase change. Larger models have more heads, which gives them more capacity for other interesting Q-composition and K-composition mechanisms that small models can’t afford to express. If all “composition heads” form simultaneously during the phase change, then it’s possible that above some size, non-induction composition heads could together account for more of the phase change and in-context learning improvement than induction heads do.

The [arguments above](https://transformer-circuits.pub#argument-phase-change) are based on analysis of 34 decoder-only Transformer language models, with different snapshots saved over the course of training, for one run of training per model. The models are drawn from four different model series as follows:

The dataset used for training the small models and smeared key models was an earlier version of the dataset described in Askell et al.

For more details on model architecture and training, continue past the table below, to the [Model Details](https://transformer-circuits.pub#models) section.

In the Model Analysis Table below, each row includes a brief summary of the measurement shown. For a more in-depth explanation of the [data collection](https://transformer-circuits.pub#data-collection) and [results analysis](https://transformer-circuits.pub#results), see the Appendix.

The small models are 1- through 6-layer Transformers. These include models both with MLPs and without MLPs (i.e. “attention only” models). They have a context window of [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table).

We observe a “phase change” phenomenon that appears at approximately 

The “full-scale models” are from the same set of models as described in Askell et al.[Appendix](https://transformer-circuits.pub#more-details-about-models) for full details). The models have both dense and local attention heads. In a local attention head, each token may only attend to earlier tokens within a fixed window of relative positions. Dense heads are the standard head, where a token may attend to any earlier token (including itself). The training dataset is described earlier at the start of the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table).

Snapshots from these models were saved at exponential step numbers, at an interval of 

|  | Non-embedding parameter counts | Activation dimension  | Attention heads per layer | Attention dimension  | 
| 4 | 13M | 512 | 8 | 64 | 
| 6 | 42M | 768 | 12 | 64 | 
| 10 | 200M | 1280 | 20 | 64 | 
| 16 | 810M | 2048 | 32 | 64 | 
| 24 | 2.7B | 3072 | 48 | 64 | 
| 40 | 13B | 5120 | 40 | 128 | 

The “smeared key” architecture modification described in [Argument 2](https://transformer-circuits.pub#argument-architectural-requirements) is as follows: we introduce a trainable real parameter [small models](https://transformer-circuits.pub#small-models). We present these only at one-layer and two-layer sizes.

As with all scientific investigations, in the course of this work we’ve encountered a few unexplained phenomena. In this section, we discuss these and provide very preliminary investigations of a few that were especially surprising.

One of the stranger observations in this paper is that [in-context learning score](https://transformer-circuits.pub#in-context-learning-key-concept) (as we've defined it: the loss of the 500th token in the context minus the loss of the 50th token in the context) is more or less the same for all models after the phase change. It appears to not matter whether the model is a tiny two layer model or a fairly large 13 billion parameter model, nor whether the model has just gone through the phase change or trained much longer. The only thing that matters, seemingly, is whether the model has gone through the phase change at all.

A natural question is whether this might be an artifact of the relatively arbitrary definition. After all, there's no reason to privilege token index 50 or 500 in the context. But it appears that varying these doesn't matter. In the following plot, we show how the large models' "in-context learning score" varies if we define it instead as the difference between the loss at the final token in the context (8192) and other indices. While there are small differences between models – for some definitions, small models would do slightly more "in-context learning"!

How can this be? First, it's important to be clear that large models still predict tokens at all indices better than small models, and they're best at predicting later tokens. What's going on is that the large models gain all their advantage over small models very early in the context. In fact, the majority of the difference forms in the first ten tokens:

It seems that large models are able to pull a lot of information out of the very early context. (This might partly be, as an example, because their increased world knowledge mean they don't need to gain as much information from the context.) They then further decrease their loss by a roughly fixed amount over the remainder of the context.

Another observation we find quite striking is that if one looks at the derivative of the loss curves of models of different sizes, it appears that their order switches at the phase change. This is most easily seen by plotting the derivative of loss with respect to the log of elapsed tokens (since loss curves are often most easily reasoned about on a log x-axis). The key observation is that the loss decreases more slowly for small models than large models before the phase change, but the opposite is true after.

While it doesn't seem that surprising that small models learn more quickly in early training, it is striking that this inversion seems to coincide with the phase change. It's another piece of evidence that suggests the phase change is an important transition point in the training of transformers.

In the [model analysis table](https://transformer-circuits.pub#model-analysis-table):

And in the [Appendix](https://transformer-circuits.pub#evaluators-on-models): 

The ultimate motivation of our research is the theory that reverse engineering neural networks might help us be confident in their safety. Our work is only a very preliminary step towards that goal, but it does begin to approach several safety-relevant issues:

Phase changes: If neural network behavior discontinuously changes from one scale to the next, this makes it more challenging for researchers and society to prepare for future problems.

In-Context Learning: In-context learning has been a topic of concerned speculation in the safety community. With less-capable neural networks, one might be tempted to treat their behavior as relatively fixed after training. (That said, demonstrations of adversarial reprogramming

Mesa-Optimization: There have been some concerns that the underlying mechanism of in-context learning might be mesa-optimization

The in-context-learning phase change may be a useful "Rosetta stone" linking mechanistic interpretability, learning dynamics

The general approach of this paper to reverse engineering transformers is based heavily on our previous paper, [A Mathematical Framework for Transformer Circuits](https://transformer-circuits.pub/2021/framework/index.html). There is much to be said about how that framework relates to other work in interpretability. Rather than repeating it, we refer readers to [Related Work](https://transformer-circuits.pub/2021/framework/index.html#related-work) in our previous paper, especially discussion of the relationship [to circuits](https://transformer-circuits.pub/2021/framework/index.html#circuits) [to analysis of attention heads](https://transformer-circuits.pub/2021/framework/index.html#attention-head-analysis) (e.g. [to related mathematical analysis](https://transformer-circuits.pub/2021/framework/index.html#mathematical-framework) (e.g. 

Building on that perspective, we here focus on how aspects of this paper raise new connections to the machine learning literature, separate from the connections raised simply by the underlying framework.

Emergent in-context learning was compellingly demonstrated in GPT-3 

Some of the findings of these papers are consistent with the induction head hypothesis, or support our methodology:

However, there are also places where experiments in these papers seem in tension with the induction head hypothesis:

Note that we are using a broader conception of “in-context learning”, rather than something as specific as “few-shot learning”. This is in contrast with Brown et al.

Over the last few years, the observation that machine learning models change in smooth, predictable ways described by scaling laws 

The relationship between scaling laws and mechanistic interpretability might be seen as analogous to the relationship between thermodynamics and the physics of individual particles. For both thermodynamics and scaling laws, even though the underlying system is very complicated, we're able to find simple relationships between the variables – for thermodynamics: entropy, temperature, volume and pressure; for neural networks: loss, compute, parameters, and data. In contrast, mechanistic interpretability studies the individual circuits underlying our models, vaguely analogous to how one might carefully study individual particles in physics. In physics, these two layers of abstraction were bridged by statistical physics.

Can we bridge these two levels of abstraction in machine learning? The induction head phase change is the first time we're aware of a bridge. They give us a phenomenon at the level of macroscopic properties at loss which can be explained at the level of circuits and mechanistic interpretability.

In fact, induction heads may be able to explain previous observed exceptions to scaling laws. In our work, 1-layer transformers seem very different from deeper transformers. This was previously observed by Kaplan et al. 

In the previous section, we discussed how scaling laws describe smooth predictable relationships between model loss and properties like scale. However, a more recent set of results have made the situation seem more subtle. While models’ losses often scale in predictable ways, there are cases where behavior is more complex:

For more general discussion of these phase change phenomena, see a [recent blog post](https://bounded-regret.ghost.io/future-ml-systems-will-be-qualitatively-different/) by Steinhardt 

The discontinuous phase change behavior we observe with induction heads is most analogous to Power et al. 

If neural networks can genuinely be understood mechanistically, in terms of circuits, it seems like there almost has to be some way to understand the learning process in terms of the dynamics of circuits changing. Induction heads offer an interesting preliminary bridge between these topics, and are a source of optimism for such a connection. This section will briefly review some strands of work on the learning dynamics side, which seem particularly promising to think about if one wanted to pursue such a connection further.

One remarkable result in learning dynamics, by Saxe et al 

Another interesting line of work has been the study of the geometry of neural network loss surfaces (e.g. 

In the context of interpretability and circuits, ["universality"](https://distill.pub/2020/circuits/zoom-in/#claim-3)

[I]magine the study of anatomy in a world where every species of animal had a completely unrelated anatomy: would we seriously study anything other than humans and a couple domestic animals? In the same way, the universality hypothesis determines what form of circuits research makes sense. If it was true in the strongest sense, one could imagine a kind of “periodic table of visual features” which we observe and catalogue across models. On the other hand, if it was mostly false, we would need to focus on a handful of models of particular societal importance and hope they stop changing every year.   

Research on universality began with Li et al. 

Certain kinds of universality are often implicitly assumed in the language model attention head interpretability literature. For example, it seems widely accepted that "previous token" attention heads form across many transformer language models (e.g. 

Our work in this paper has analogies to many of these strands of prior work. Like the previous attention head papers, we describe the induction head pattern as a universal attention pattern. However, our analysis of these heads’ OV and QK circuits extends this claim of universality to the circuit level, similar to the original Circuits thread. And a corollary of our analysis of the OV circuit is a claim about what feature the attention head computes (roughly: the token embedding of the token following a previous copy of the present token) which is more similar to the traditional work on universality.

Separate from all of this, it's worth mentioning that increasingly there's evidence for a particularly extreme kind of universality at the intersection of neuroscience and deep learning. Increasingly, research suggests that biological and artificial neural networks learn similar representations (e.g. 

In [Argument 4](https://transformer-circuits.pub#argument-induction-heads-can-implement-abstract-behaviors), we saw an induction head that helps implement translation. Although we're not aware of anything quite so general in the prior literature, there are reports of attention patterns which, in retrospect, seem somewhat similar. Often, in translation-like tasks, we see attention attend to the token which is about to be translated. We see this in literal translation (e.g. 

Inspired by the original [Circuits Thread](https://distill.pub/2020/circuits/) and [Distill's Discussion Article experiment](https://distill.pub/2019/advex-bugs-discussion/), the authors invited several external researchers who were also investigating induction heads to comment on this work. Their comments are included below.

I used the empirical criterion for induction heads [presented](https://transformer-circuits.pub/2022/phase-change/index.html#definition-of-induction-heads) in this paper to look for induction heads in publicly available models. To reiterate: on a sequence of tokens `[A][B] .... [A] → [B]`, a head is called an induction head if it attends to the previous `[B]` when reading the last `[A]`, and if the previous `[B]` increased the logit of the last `[B]`.

Under this definition, I found potential induction heads in GPT2 and GPT-Neo mostly starting in the mid-depth region. I made an interactive version to explore attention and logit attribution across all layers and heads on the long Harry Potter prompt with the repetition of the first paragraph. It can be accessed [here](https://share.streamlit.io/tomfrederik/gpt_induction_vis/main). For example, for GPT2-XL, head 20 in layer 21 seems to be an induction head, as well as head 0 in layer 12 of GPT-Neo-2.7B. For these heads we can see that virtually every token in the repetition of the first paragraph attends to its following token in the original paragraph. Thanks to [EleutherAI](https://www.eleuther.ai/) for providing the compute resources for this project.

At one point the paper speculates on the minimal context length necessary to form induction heads. On a synthetic dataset, I have already found induction to be easily learned with a context length of 4. However, I believe that this is, in general, a property of the dataset/data distribution rather than the model itself, i.e. "how useful is induction for this task, on this dataset?". While it is intriguing to think about the statistics of language in this way, I am not sure how useful this line of research would be to the overall project of interpretability.

The following changes have been made after publication:

In writing this paper, our thinking and exposition was greatly clarified by detailed correspondence with Sam Bowman, Paul Christiano, Aidan Gomez, Dan Hendrycks, Jacob Hilton, Evan Hubinger, Andrew Ilyas, Percy Liang, Tom Lieberum, Chris Maddison, Aleksander Madry, Ethan Perez, Jacob Steinhardt, and Martin Wattenberg.

We're especially indebted to Adam Scherlis and Tom Lieberum who replicated our results and wrote comments to be included with this paper.

We're also deeply grateful to Daniela Amodei, Stanislav Fořt, Tristan Hume, Saurav Kadavath, Jamie Kerr, Shauna Kravec, Jeffrey Ladish, Jia Yuan Loke, Liane Lovitt, Rebecca Raible, Sheer El Showk, Timothy Telleen-Lawton, Matt O’Brien, Kate Rudolph, Jemima Jones, Geoffrey Irving, Tom McGrath, Michela Paganini, Allan Dafoe, Gabriel Goh, Nick Cammarata, Chelsea Voss, Shan Carter, Katherine Lee, Beth Barnes, Jan Leike, Nate Thomas, Buck Shlegeris, Alex Tamkin, Quinn Tucker, and Rob Harries; for their support, for comments on this work, and for conversations that contributed to the background thinking on interpretability and safety this work is based on.

Research: The bulk of this line of work, including many iterations of experiments that didn’t make it into the final paper, was conducted by Catherine Olsson with mentorship from Chris Olah. Chris first discovered the “induction bump” in a 2-layer model. He had the idea of investigating what aspects of the per-token loss changed most over the bump, and made the connection to in-context learning. Catherine led data collection and analysis throughout the project. Data collection made extensive use of Garcon infrastructure by Nelson Elhage. Nelson collected data and ran analyses for the smeared key models. Chris Olah, Neel Nanda, and Catherine Olsson wrote the head activation evaluators. Research ideas were significantly influenced and improved by others at Anthropic.

Writing: This article was drafted by Catherine Olsson, Chris Olah, and Nelson Elhage, with contributions and editing from Neel Nanda and Dario Amodei. Dario Amodei originally suggested the “arguments” structure. The specific arguments were iteratively refined in collaborative conversations between Dario Amodei, Chris Olah, Catherine Olsson, and Nelson Elhage. Nick Joseph contributed significant editing. Other members of Anthropic made miscellaneous contributions throughout the writing process.

Figures and visualizations: Catherine Olsson generated the graphs in the model analysis table, and did preliminary design and styling. Chris Olah finalized the design and styling, and created “pull-out” versions for smaller figures earlier in the paper. Nelson Elhage contributed to the graphs of per-token losses on Harry Potter. Chris discovered the heads showcased in Claim 2 and designed the “month-animal-color-fruit” task. Nelson set up the interactive visualizations, using existing visualization tools built by Chris, with contributions from Nelson and Catherine, using PySvelte (see [github](https://github.com/anthropics/PySvelte)).


Model training: Nelson Elhage created the attention-only models and the smeared-key models. Dario Amodei collected data for models trained on the alternate books-only dataset. All our interpretability research is enabled by having access to models to study, including large models. Led by Tom Brown, Sam McCandlish, and Jared Kaplan, the majority of Anthropic's technical staff contributed to the development of our efficient distributed training infrastructure and the underlying machine learning. Core contributors include Nicholas Joseph, Tom Henighan, and Ben Mann. Nelson Elhage, Kamal Ndousse, Andy Jones, Zac Hatfield-Dodds, and Danny Hernandez also contributed to this infrastructure.

Cluster: Tom Henighan and Nova DasSarma, advised by Tom Brown and Sam McCandlish with contributions from many others at Anthropic, managed the research cluster our research depended on and maintained its stability. Nova provided important support when interpretability required unorthodox jobs to run on our cluster.

Please cite as:

Olsson, et al., "In-context Learning and Induction Heads", Transformer Circuits Thread, 2022.

BibTeX Citation:

```
@article{olsson2022context,
   title={In-context Learning and Induction Heads},
   author={Olsson, Catherine and Elhage, Nelson and Nanda, Neel and Joseph, Nicholas and DasSarma, Nova and Henighan, Tom and Mann, Ben and Askell, Amanda and Bai, Yuntao and Chen, Anna and Conerly, Tom and Drain, Dawn and Ganguli, Deep and Hatfield-Dodds, Zac and Hernandez, Danny and Johnston, Scott and Jones, Andy and Kernion, Jackson and Lovitt, Liane and Ndousse, Kamal and Amodei, Dario and Brown, Tom and Clark, Jack and Kaplan, Jared and McCandlish, Sam and Olah, Chris},
   year={2022},
   journal={Transformer Circuits Thread},
   note={https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html}
}
```
In this section we examine: Where are induction heads located in models? Do they generally form in the second layer, or the last layer, or somewhere else?

In our attention-only models, induction heads form in the last layer.

(In these plots, each point represents a head, colored by layer and arranged along the x-axis by depth in the model. The height of the point is the [prefix matching score](https://transformer-circuits.pub#head-activation-evaluators) of that head.)

In small models with MLPs, by the 5L and 6L size, the induction heads form more in the second-to-last layer.

In the models in our full-scale sweep, induction heads form earlier. In fact, in 24L and 40L models the majority of induction heads form before the halfway point in model depth.

Note that in these plots, only a randomized sample of 100 heads (with selection biased towards those scoring highly on prefix matching) are shown as opaque dots; most of the datapoints are left translucent for ease of reading.

What is the distribution of heads’ scores on our [head activation evaluators](https://transformer-circuits.pub#head-activation-evaluators)?

The distribution is sharper in our small models than our full-size models. In both cases, heads with a positive prefix matching score are more likely to have a positive copying score, and this correlation is even stronger for the highest prefix matching scores.

(For models with more than 100 heads, a subset of heads is plotted, as seen in the opaque versus translucent datapoints [immediately above](https://transformer-circuits.pub#where-are-induction-heads-located-in-models).)

We show here that our [head activation evaluators](https://transformer-circuits.pub#head-activation-evaluators) (that we use to measure the properties in our [definition of induction heads](https://transformer-circuits.pub#definition-of-induction-heads)) correlate well with mathematically-based measures in small attention-only models.

Our copying evaluator correlates well with the OV matrix of each head. Specifically here we plot against the sum of the eigenvalues, divided by the absolute value of the eigenvalues, which is a measure of what fraction of the eigenvalues are positive vs. negative.

Our prefix matching evaluator correlates well with the [trace of QK eigenvalues](https://transformer-circuits.pub#qk-eig-trace) for the previous-token QK-circuit term.

We measure several properties of the model’s overall behavior and of each constituent attention head. With the exception of the attention head ablations (which are the most computationally-intensive, and so were run only on the small models), we measure these properties on all snapshots of all models across training (that is, 200 snapshots each for the twelve small models, and 14 or 15 snapshots each for the six full-scale models). By measuring them on each snapshot, we can see how each property evolves over the course of training, rather than simply measuring the final model.

The properties we collect data on are as follows:

The most straightforward property we measure is per-token losses from each model snapshot. We run the model snapshot on a set of 10,000 randomly-selected examples from the dataset, each of which is 512 tokens in length, and save individual token losses for every token. The set is held the same between all snapshots and between all models in all three families.

When analyzing the per-token loss data, we summarize it in two different ways:

One way to study individual attention heads is to measure the difference between the complete model’s overall behavior and the overall behavior of a model that is exactly the same except for missing that head. We do this by successively ablating (i.e. zeroing-out) each attention head’s contribution to the residual stream. The approach of removing a head’s contribution can be thought of as loosely similar to a genetic “knockout” experiment in biology, where observing the effect of removing a specific gene can provide information about the gene’s function.

We could do these ablations in two different ways: a “full” ablation, and a “pattern-preserving” ablation. (The [model analysis table](https://transformer-circuits.pub#model-analysis-table) shows only pattern-preserving ablations; we explain the other method for comparison) The full ablation would remove all of a head’s contributions to model behavior, whereas the pattern-preserving ablation removes the ablated head’s contribution to all later heads’ output vectors, but preserves the ablated head’s contribution to all later heads’ attention patterns.[separable](https://transformer-circuits.pub/2021/framework/index.html#splitting-attention-head-terms-into-circuits), that is, implemented by different circuits consisting of different sets of parameters which may be run and studied in isolation

Full ablation

Pattern-preserving ablation (shown in [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table))

This is by far the most computationally-intensive measurement we perform, although fortunately it consists of many small single-GPU jobs and so we could often run it in “spare” capacity.

As a quick note of comparison, our previous work in [Transformer Circuits](https://transformer-circuits.pub/2021/framework/index.html#term-importance-analysis) also contained ablation experiments. Although there are similarities (most notably the decision to freeze patterns), the specific ablations we describe here are individual attention head ablations where only one attention head is modified at a time, whereas the ablations described in our prior work are nth-order term ablations in which all attention heads are manipulated at once and the process is repeated iteratively n times.

We construct three heuristic measurements of attention heads, to score how strongly a head exhibits specific properties of interest. These are empirical measurements, based on observed activations on example data.

We evaluate heads on the following properties:

(Note that “copying” is a property that depends on a head’s OV circuit, whereas “prefix matching” and “previous token attention” are properties that depend on a head’s QK circuit.)

We are interested in these properties in particular — copying, prefix matching, and previous token attention — because they are relevant to induction heads’ function of predicting repeated sequences of random tokens. Specifically, (1) a head with both a “copying” OV circuit and a “prefix matching” QK circuit is thereby an induction head by our definition, because it attends to previous instances of the present token and increases the logit of the token that followed; and (2) a “previous token” head can be used as an algorithmic component of a future induction head in a later layer (as we explain in more detail in [Argument 5](https://transformer-circuits.pub#argument-induction-head-mechanism)).

In more detail, each head evaluator is implemented as follows, and each score is averaged over 10 examples.

In the Appendix we [validate the connection](https://transformer-circuits.pub#validating-qk-and-ov-eigenvalues) between these empirical measurements and the mathematical properties of attention head weight matrices explored in [Transformer Circuits](https://transformer-circuits.pub/2021/framework/index.html), and show a [joint distribution of scores](https://transformer-circuits.pub#evaluators-on-models) for copying and prefix matching, for heads in the small and large models.

In addition to the heuristic measurements described above, we construct another measure of prefix matching that draws on concepts from [A Mathematical Framework for Transformer Circuits](https://transformer-circuits.pub/2021/framework/index.html). In short, we measure how overall large and positive the eigenvalues are of the [K-composition](https://transformer-circuits.pub/2021/framework/index.html#three-kinds-of-composition)” with heads that attend to the previous token. Positive eigenvalues of 

The “QK circuit” is a term we coined in [our previous work](https://transformer-circuits.pub/2021/framework/index.html#path-expansion-of-attention-scores-qk-circuit) to refer specifically to the entire transformation [Path Expansion of Attention Scores QK Circuit](https://transformer-circuits.pub/2021/framework/index.html#path-expansion-of-attention-scores-qk-circuit) in our earlier work shows all the terms of a full expansion for a two-layer model; models with more layers have QK circuits with even more terms.

For this measure, we narrow the scope of our calculation to the simplest possible term of the QK circuit that could contribute to the “prefix-matching” behavior of an induction head: namely, only pure [K-composition](https://transformer-circuits.pub/2021/framework/index.html#three-kinds-of-composition) terms (in which the query-side term is just the direct path from the embeddings), and furthermore only the subset of those that correspond specifically to K-composing with an earlier head that consistently attends to the immediately previous token. 

In order to compute this measure for a given head, we do the following:

Because this measure relies on K-composition with heads in previous layers, it is only defined for models more than one layer deep, and only for attention heads in the second layer and beyond.

We show in the [Appendix](https://transformer-circuits.pub#validating-qk-and-ov-eigenvalues) that this measure correlates well with [our heuristic prefix matching evaluator](https://transformer-circuits.pub#head-activation-evaluators) on our small models.

The second through fourth rows of the [Model Analysis Table](https://transformer-circuits.pub#model-analysis-table) show results derived from [the per-token losses](https://transformer-circuits.pub#per-token-losses-unmodified) from the small models and full-scale models. We show the following “views” on the per-token loss data:

Additionally in a separate row at the top, we show PCA results from the per-token results. These are displayed separately because, unlike all the other graphs, the x-axis is not elapsed training tokens.

The next two rows of Figure 1 show results derived from the [attention head measurements](https://transformer-circuits.pub#head-activation-evaluators): one row shows the score of the “prefix matching” head activation evaluator over training time, and the following row shows the prefix-matching QK-circuit eigenvalue trace over training time.

For the small models, all attention heads are shown. For the full-scale models, a subset of 100 heads is shown; see the [Appendix](https://transformer-circuits.pub#where-are-induction-heads-located-in-models) for an explanation of how the 100 heads were chosen.

Note that in the QK-circuit plot, the change in weight decay at approximately 5 billion tokens (described in [Model Details](https://transformer-circuits.pub#models)) can be seen particularly prominently as a bend or hiccup in the plotted trajectories, but occurs after the phase change and is an unrelated phenomenon.

For the small models only, the final two rows of Figure 1 show results derived from the per-token loss data from the attention head ablation experiments. We show only the [pattern-preserving ablation](https://transformer-circuits.pub#per-token-losses-ablated) results here; full ablation results can be found in the appendix. Note also that ablation experiments were not conducted for the full-scale models.

The first “view” on the ablation data is the attribution to the “before-and-after” vector, which relates the result of each head ablation to the change in model behavior over the course of the phase change, and which we explain here. First: For a given model, a length-10,000 vector is computed by taking the delta of the random-token-per-example loss data for the “phase change end” snapshot, minus the “phase change start” snapshot. This represents how the model’s behavior changed over the course of the phase change. This vector is normalized to a unit vector; we call this the before-and-after vector. It can be thought of as the “direction in token-loss space” that is affected by the phase change. Second: Note that ablating a given head also induces a change in the behavior of a model snapshot, and thus a corresponding length-10,000 vector change to the random-token-per-example loss data (calculated by subtracting the un-ablated snapshot’s per-token loss vector from the ablated snapshot’s per-token loss vector). We are interested in the similarity between these two changes, which we compute as the dot product between the model’s before-and-after vector and the ablation change vector for this head and this snapshot. For induction heads, this dot product is negative, indicating that removing induction heads is like undoing the changes to model behavior that happened during the phase change.

The second “view” on the ablation data is the attribution to the in-context learning score. We operationalize the model’s in-context learning abilities as how much better the model is at predicting the 500th token than the 50th token: this difference measured as Loss(ntrain, icontext= 500) − Loss(ntrain, icontext= 50) is the in-context learning score.

## Replication

Adam Scherlisis a researcher at Redwood Research.
Redwood Research has been working on language model interpretability, inspired in part by Anthropic’s work. We’ve found induction heads reliably appearing in two-layer attention-only transformers. Their structure roughly matches the description in “Analyzing a Two-Layer Model” from Anthropic’s previous paper, with previous-token heads in layer 0 and induction heads in layer 1 (often one of each). These each have the expected attention behaviors. We tested this by replacing attention-score matrices with idealized versions and comparing the change in loss to the change from ablating that head. Replacing the previous-token head’s scores with exact previous-token attention recovered 99% of the loss difference. Replacing the induction head’s scores with a simple approximation (attending to the first token and to exact

`[A][B]...[A]`matches) recovered about 65% of the loss difference. Our induction heads also match patterns of the form`[A][B][C]...[A][B]→[C]`; including this in the substituted attention scores recovered an additional 10% of loss difference. The induction head’s OV circuit copies tokens, including some fuzzy matching of semantically similar tokens. Its QK circuit is dominated by K-composition with the previous-token head; the previous-token head’s OV matrix copies information to a new subspace and the induction head’s QK matrix copies it back to the usual token embedding. We’ve also cataloged a few other kinds of attention heads, including skip-trigram heads.
