Title: A Mathematical Framework for Transformer Circuits

URL Source: https://transformer-circuits.pub/2021/framework

Markdown Content:
Transformer

One avenue for addressing these issues is mechanistic interpretability, attempting to reverse engineer the detailed computations performed by transformers, similar to how a programmer might try to reverse engineer complicated binaries into human-readable source code.  If this were possible, it could potentially provide a more systematic approach to explaining current safety problems, identifying new ones, and perhaps even anticipating the safety problems of powerful future models that have not yet been built.  A previous project, the [Distill Circuits thread](https://distill.pub/2020/circuits/) 

In this paper, we attempt to take initial, very preliminary steps towards reverse-engineering transformers. Given the incredible complexity and size of modern language models, we have found it most fruitful to start with the simplest possible models and work our way up from there. Our aim is to discover simple algorithmic patterns, motifs, or frameworks that can subsequently be applied to larger and more complex models. Specifically, in this paper we will study transformers with two layers or less which have only attention blocks – this is in contrast to a large, modern transformer like GPT-3, which has 96 layers and alternates attention blocks with MLP blocks.

We find that by conceptualizing the operation of transformers in a new but mathematically equivalent way, we are able to make sense of these small models and gain significant understanding of how they operate internally. Of particular note, we find that specific attention heads that we term “induction heads” can explain in-context learning in these small models, and that these heads only develop in models with at least two attention layers. We also go through some examples of these heads operating in action on specific data.

We don’t attempt to apply to our insights to larger models in this first paper, but in a [forthcoming paper](https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html), we will show that both our mathematical framework for understanding transformers, and the concept of induction heads, continues to be at least partially relevant for much larger and more realistic models – though we remain a very long way from being able to fully reverse engineer such models.

To explore the challenge of reverse engineering transformers, we reverse engineer several toy, attention-only models. In doing so we find:

We’ve found that many subtle details of the transformer architecture require us to approach reverse engineering it in a pretty different way from how the InceptionV1 Circuits work 

Before we attempt to reverse engineer transformers, it's helpful to briefly review the high-level structure of transformers and describe how we think about them.

In many cases, we've found it helpful to reframe transformers in equivalent, but non-standard ways. Mechanistic interpretability requires us to break models down into human-interpretable pieces. An important first step is finding the representation which makes it easiest to reason about the model. In modern deep learning, there is — for good reason! — a lot of emphasis on computational efficiency, and our mathematical descriptions of models often mirror decisions in how one would write efficient code to run the model. But when there are many equivalent ways to represent the same computation, it is likely that the most human-interpretable representation and the most computationally efficient representation will be different.

Reviewing transformers will also let us align on terminology, which can sometimes vary. We'll also introduce some notation in the process, but since this notation is used across many sections, we provide a detailed description of all notation in the [notation appendix](https://transformer-circuits.pub#notation) as a concise reference for readers.

To demonstrate the ideas in this paper in their cleanest form, we focus on "toy transformers" with some simplifications.

In most parts of this paper, we will make a very substantive change: we focus on “attention-only” transformers, which don't have MLP layers. This is a very dramatic simplification of the transformer architecture. We're partly motivated by the fact that circuits with attention heads present new challenges not faced by the Distill circuits work, and considering them in isolation allows us to give an especially elegant treatment of those issues. But we've also simply had much less success in understanding MLP layers so far; in normal transformers with both attention and MLP layers there are many circuits mediated primarily by attention heads which we can study, some of which seem very important, but the MLP portions have been much harder to get traction on. This is a major weakness of our work that we plan to focus on addressing in the future. Despite this, we will have some discussion of transformers with MLP layers in later sections.

We also make several changes that we consider to be more superficial and are mostly made for clarity and simplicity. We do not consider biases, but a model with biases can always be simulated without them by folding them into the weights and creating a dimension that is always one. Additionally, biases in attention-only transformers mostly multiply out to functionally be biases on the logits. We also ignore layer normalization. It adds a fair amount of complexity to consider explicitly, and up to a variable scaling, layer norm can be merged into adjacent weights. We also expect that, modulo some implementational annoyances, layer norm could be substituted for batch normalization (which can fully be folded into adjacent parameters).

There are several variants of transformer language models. We focus on autoregressive, decoder-only transformer language models, such as GPT-3. (The original transformer paper had a special encoder-decoder structure to support translation, but many modern language models don't include this.)

A transformer starts with a token embedding, followed by a series of “residual blocks”, and finally a token unembedding. Each residual block consists of an attention layer, followed by an MLP layer. Both the attention and MLP layers each “read” their input from the residual stream (by performing a linear projection), and then “write” their result to the residual stream by adding a linear projection back in. Each attention layer consists of multiple heads, which operate in parallel.

One of the main features of the high level architecture of a transformer is that each layer adds its results into what we call the “residual stream.”

The residual stream has a deeply linear structure.["privileged basis"](https://transformer-circuits.pub#def-privileged-basis); we could rotate it by rotating all the matrices interacting with it, without changing model behavior.

An especially useful consequence of the residual stream being linear is that one can think of implicit "virtual weights" directly connecting any pair of layers (even those separated by many other layers), by multiplying out their interactions through the residual stream. These virtual weights are the product of the output weights of one layer with the input weights

The residual stream is a high-dimensional vector space. In small models, it may be hundreds of dimensions; in large models it can go into the tens of thousands. This means that layers can send different information to different layers by storing it in different subspaces. This is especially important in the case of attention heads, since every individual head operates on comparatively small subspaces (often 64 or 128 dimensions), and can very easily write to completely disjoint subspaces and not interact.

Once added, information persists in a subspace unless another layer actively deletes it. From this perspective, dimensions of the residual stream become something like "memory" or "bandwidth". The original token embeddings, as well as the unembeddings, mostly interact with a relatively small fraction of the dimensions.

It seems like we should expect residual stream bandwidth to be in very high demand! There are generally far more "computational dimensions" (such as neurons and attention head result dimensions) than the residual stream has dimensions to move information. Just a single MLP layer typically has four times more neurons than the residual stream has dimensions. So, for example, at layer 25 of a 50 layer transformer, the residual stream has 100 times more neurons as it has dimensions before it, trying to communicate with 100 times as many neurons as it has dimensions after it, somehow communicating in superposition! We call tensors like this ["bottleneck activations"](https://transformer-circuits.pub#def-bottleneck-activation) and expect them to be unusually challenging to interpret. (This is a major reason why we will try to pull apart the different streams of communication happening through the residual stream apart in terms of virtual weights, rather than studying it directly.)

Perhaps because of this high demand on residual stream bandwidth, we've seen hints that some MLP neurons and attention heads may perform a kind of "memory management" role, clearing residual stream dimensions set by other layers by reading in information and writing out the negative version.

As seen above, we think of transformer attention layers as several completely independent attention heads 

In the original Vaswani et al. paper on transformers 

Revealing it to be equivalent to running heads independently, multiplying each by its own output matrix, and adding them into the residual stream. The concatenate definition is often preferred because it produces a larger and more compute efficient matrix multiply. But for understanding transformers theoretically, we prefer to think of them as independently additive.

But if attention heads act independently, what do they do? The fundamental action of attention heads is moving information. They read information from the residual stream of one token, and write it to the residual stream of another token. The main observation to take away from this section is that which tokens to move information from is completely separable from what information is “read” to be moved and how it is “written” to the destination.

To see this, it’s helpful to write attention in a non-standard way. Given an attention pattern, computing the output of an attention head is typically described in three steps:

Each of these steps can be written as matrix multiply: why don’t we collapse them into a single step? If you think of [short introduction](https://transformer-circuits.pub#notation-tensor-product) in the notation appendix).  One piece of motivation that may be helpful is to note that we want to express linear maps from matrices to matrices: 

Using tensor products, we can describe the process of applying attention as:

Applying the mixed product property and collapsing identities yields:

What about the attention pattern? Typically, one computes the keys 

It's worth noting that although this formulation is mathematically equivalent, actually implementing attention this way (ie. multiplying by 

A major benefit of rewriting attention heads in this form is that it surfaces a lot of structure which may have previously been harder to observe:

Watch videos covering similar content to this section: [0 layer theory](https://www.youtube.com/watch?v=V3NQaDR3xI4&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51&index=1)

Before moving on to more complex models, it’s useful to briefly consider a “zero-layer” transformer. Such a model takes a token, embeds it, unembeds it to produce logits predicting the next token:

Because the model cannot move information from other tokens, we are simply predicting the next token from the present token. This means that the optimal behavior of 

This is relevant to transformers more generally. Terms of the form 

Watch videos covering similar content to this section: [1 layer theory](https://www.youtube.com/watch?v=7crsHGsh3p8&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51&index=3), [1 layer results](https://www.youtube.com/watch?v=ZBlHFFE-ng8&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51&index=4).

We claim that one-layer attention-only transformers can be understood as an ensemble of a bigram model and several "skip-trigram" models (affecting the probabilities of sequences "A… BC").

The goal of this section is to rigorously show this correspondence, and demonstrate how to convert the raw weights of a transformer into interpretable tables of skip-trigram probability adjustments.

Recall that a one-layer attention-only transformer consists of a token embedding, followed by an attention layer (which [independently applies](https://transformer-circuits.pub#architecture-attn-independent) attention heads), and finally an unembedding:

Using [tensor notation](https://transformer-circuits.pub#notation-tensor-product) and the [alternative representation of attention heads](https://transformer-circuits.pub#architecture-attn-as-movement) we previously derived, we can represent the transformer as a product of three terms.

Our key trick is to simply expand the product. This transforms the product (where every term corresponds to a layer), into a sum where every term corresponds to an end-to-end path.

We claim each of these end-to-end path terms is tractable to understand, can be reasoned about independently, and additively combine to create model behavior.

The direct path term, 

The more interesting terms are the attention head terms.

For each attention head 

The key thing to notice is that these terms consist of two separable operations, which are at their heart two 

To intuitively understand these products, it can be helpful to think of them as paths through the model, starting and ending at tokens. The QK circuit is formed by tracing the computation of a query and key vector up to their attention head, where they dot product to create a bilinear form. The OV circuit is created by tracing the path computing a value vector and continuing it through up to the logits.

The attention pattern is a function of both the source and destination token

Thinking of the OV and QK circuits separately can be very useful, since they're both individually functions we can understand (linear or bilinear functions operating on matrices we understand).

But is it really principled to think about them independently? One thought experiment which might be helpful is to imagine running the model twice. The first time you collect the attention patterns of each head. This only depends on the QK circuit.

One of the core challenges of mechanistic interpretability is to make neural network parameters meaningful by contextualizing them (see discussion by Voss et al. in [Visualizing Weights](https://distill.pub/2020/circuits/visualizing-weights/) `[source]... [destination][out]`, and the "out" is modified.

It's important to note that this doesn't mean that interpretation is trivial. For one thing, the resulting matrices are enormous (our vocabulary is ~50,000 tokens, so a single expanded OV matrix has ~2.5 billion entries); we revealed the one-layer attention-only model to be a compressed Chinese room, and we're left with a giant pile of cards. There's also all the usual issues that come with understanding the weights of generalized linear models acting on correlated variables and fungibility between variables. For example, an attention head might have a weight of zero because another attention head will attend to the same token and perform the same role it would have. Finally, there's a technical issue where QK weights aren't comparable between different query vectors, and there isn't a clear right answer as to how to normalize them.

Despite this, we do have transformers in a form where all parameters are contextualized and understandable. And despite these subtleties, we can simply read off skip-trigrams from the joint OV and QK matrices. In particular, searching for large entries in these matrices reveals lots of interesting behavior.

In the following subsections, we give a curated tour of some interesting skip-trigrams and how they're embedded in the QK/OV circuits. But full, non-cherrypicked examples of the largest entries in several models are available by following the links:

One of the most striking things about looking at these matrices is that most attention heads in one layer models dedicate an enormous fraction of their capacity to copying. The OV circuit sets things up so that tokens, if attended to by the head, increase the probability of that token, and to a lesser extent, similar tokens. The QK circuit then only attends back to tokens which could plausibly be the next token. Thus, tokens are copied, but only to places where bigram-ish statistics make them seem plausible.

In the above example, we fix a given source token and look at the largest corresponding QK entries (the destination token) and largest corresponding OV entries (the out token). The source token is selected to show interesting behavior, but the destination and out token are the top entries unless entries are explicitly skipped with an ellipsis; they are colored by the intensity of their value in the matrix.

Most of the examples are straightforward, but two deserve explanation: the fourth example (with skip-trigrams like `lambda… $\lambda`) appears to be the model learning LaTeX, while the fifth example (with the skip-trigram `nbsp… > `) appears to be the model learning HTML escape sequences.

Note that most of these examples are copying; this appears to be very common.

We also see more subtle kinds of copying. One particularly interesting one is related to how tokenization for transformers typically works. Tokenizers typically merge spaces onto the start of words. But occasionally a word will appear in a context where there isn't a space in front of it, such as at the start of a new paragraph or after a dialogue open quote. These cases are rare, and as such, the tokenization isn't optimized for them. So for less common words, it's quite common for them to map to a single token when a space is in front of them (`" Ralph" → [" Ralph"]`) but split when there isn't a space (`"Ralph" → ["R", "alph"]`). 

It's quite common to see skip-trigram entries dealing with copying in this case. In fact, we sometimes observe attention heads which appear to partially specialize in handling copying for words that split into two tokens without a space. When these attention heads observe a fragmented token (e.g. `"R"`) they attend back to tokens which might be the complete word with a space (`" Ralph"`) and then predict the continuation (`"alph"`). (It's interesting to note that this could be thought of as a very special case where a one-layer model can kind of mimic the [induction heads](https://transformer-circuits.pub#induction-heads) we'll see in two layer models.)

We can summarize this copying behavior into a few abstract patterns that we've observed:

All of these can be seen as a kind of very primitive in-context learning. The ability of transformers to adapt to their context is one of their most interesting properties, and this kind of simple copying is a very basic form of it. However, we'll see when we look at a two-layer transformer that a much more interesting and powerful algorithm for in-context learning is available to deeper transformers.

Of course, copying isn't the only behavior these attention heads encode.

Skip-trigrams seem trivial, but can actually produce more complex behavior than one might expect. Below are some particularly striking skip-trigram examples we found in looking through the largest entries in the expanded OV/QK matrices of our models.

`else`, `elif` and `except` are more likely after an indentation is reduced using skip-trigrams of the form: `\n\t\t\t … \n\t\t → else/elif/except` where the first part is indented `open()` will have a file mode string argument: `open … "," → [rb / wb / r / w]` (for example `open("abc.txt","r")`)`self`: `def … ( → self` (for example `def method_name(self):`)`super` is often used to call `.__init__()` after being invoked on `super … self → ).__` (for example `super(Parent, self).__init__()`)`upper … . → upper/lower/capitalize/isdigit`, `tf … . → dtype/shape/initializer`, `datetime… → date / time / strftime / isoformat`, `QtWidgets … . → QtCore / setGeometry / QtGui`, `pygame … . → display / rect / tick``for... in [range/enumerate/sorted/zip/tqdm]` `tbody` is often followed by `<td>` tags: `tbody … < → td``(** … X → **)`, `(' … X → ')` , `"% … X → %"` , `'</ … X → >'` (see `\left` command must have a corresponding `\right` command; conversely `left … \ → right` `keep … [in → mind / at → bay / under → wraps]`,  `difficult … not → impossible`)`" and"`:   `back and → forth`,  `eat and → drink`, `trying and → failing`, `day and → night`, `far and → away`, `created and → maintained`, `forward and → backward`, `past and → present`, `happy and → satisfied`, `walking and → talking`, `sick and → tired`, … (see `twitter … / → status`, `github … / → [issues / blob / pull / master]`, `gmail … . → com`, `http … / → [www / google / localhost / youtube / amazon]`, `http … : → [8080 / 8000]`, `www … . → [org / com / net]`
One thing to note is that the learned skip-trigrams are often related to idiosyncrasies of one's tokenization. For example collapsing whitespace together allows individual tokens to reveal indentation. Not merging backslash into text tokens means that when the model is predicting LaTeX, there's a token after backslash that must be an escape sequence. And so on.

Many skip tri-grams can be difficult to interpret without specific knowledge (e.g. `Israel … K → nes` only makes sense if you know Israel's legislative body is called the "Knesset"). A useful tactic can be to try typing potential skip tri-grams into Google search (or similar tools) and look at autocompletions.

Our treatment of attention heads hasn't discussed how attention heads handle position, largely because there are now several competing methods (e.g. 

In practice, the one-layer models tend to have a small number of attention heads that are primarily positional, strongly preferring certain relative positions. Below, we present one attention head which either attends to the present token or the previous token.

One of the most interesting things about looking at the expanded QK and OV matrices of one layer transformers is that they can shed light on transformer behavior that seems incomprehensible from the outside.

Our one-layer models represent skip-trigrams in a "factored form" split between the OV and QK matrices. It's kind of like representing a function `keep… in mind` and `keep… at bay`, it must also increase the probability of `keep… in bay` and `keep… at mind`. This is likely a good trade for the model on balance, but is also, in some sense, a bug. We frequently observe these in attention heads.

Highlighted text denotes skip-trigram continuations that the model presumably ideally wouldn't increase the probability of. Note that `QCanvas` [is a class](https://doc.qt.io/archives/3.3/qcanvas.html) involving pixmaps in the popular Qt library. `Lloyd... Catherine` likely refers to Catherine Lloyd Burns. These examples are slightly cherry-picked to be interesting, but very common if you look at the expanded weights for models linked above.

Even though these particular bugs seem in some sense trivial, we’re excited about this result as an early demonstration of using interpretability to understand model failures. We have not further explored this phenomenon, but we’d be curious to do so in more detail. For instance, could we characterize how much performance (in points of loss or otherwise) these “bugs” are costing the model? Does this particular class continue to some extent in larger models (presumably partially, but not completely, masked by other effects)?

We've turned the problem of understanding one-layer attention-only transformers into the problem of understanding their expanded OV and QK matrices. But as mentioned above, the expanded OV and QK matrices are enormous, with easily billions of entries. While searching for the largest entries is interesting, are there better ways to understand them? There are at least three reasons to expect there are:

`" by"` (e.g. `"Anne… by → Anne"`) while location names have top queries like `" from"` (e.g. `"Canada… from → Canada"`). This hints at something like cluster structure in the matrix.
We don't yet feel like we have a clear right answer, but we're optimistic that the right kind of matrix decomposition or dimensionality reduction could be highly informative. (See the technical details appendix for notes on how to efficiently work with these large matrices.)

The type of behavior we're most excited to detect in an automated way is copying. Since copying is fundamentally about mapping the same vector to itself (for example, having a token increase its own probability) it seems unusually amenable to being captured in some kind of summary statistic.

However, we've found it hard to pin down exactly what the right notion is; this is likely because there are lots of slightly different ways one could draw the boundaries of whether something is a "copying matrix" and we're not yet sure what the most useful one is. For example, we don't observe this in the models discussed in this paper, but in slightly larger models we often observe attention heads which "copy" some mixture of gender, plurality, and tense from nearby words, helping the model use the correct pronouns and conjugate verbs. The matrices for these attention heads aren't exactly copying individual tokens, but it seems like they are copying in some very meaningful sense. So copying is actually a more complex concept than it might first appear.

One natural approach might be to use eigenvectors and eigenvalues. Recall that 

The eigendecomposition expresses the matrix as a set of such eigenvectors and eigenvalues. For a random matrix, we expect to have an equal number of positive and negative eigenvalues, and for many to be complex.

One can even collapse that down further and get a histogram of how many of the attention heads are copying (if one trusts the eigenvalues as a summary statistic):

It appears that 10 out of 12 heads are significantly copying! (This agrees with qualitative inspection of the expanded weights.)

But while copying matrices must have positive eigenvalues, it isn't clear that all matrices with positive eigenvalues are things we necessarily want to consider to be copying. A matrix's eigenvectors aren't necessarily orthogonal, and this allows for pathological examples;

One might try to formalize "copying matrices" in other ways. One possibility is to look at the diagonal of a matrix, which describes how each token affects its own probability. As expected, entries on the diagonal are very positive-leaning. We can also ask how often a random token increases its own probability more than any other token (or is one of the k-most increased tokens, to allow for tokens which are the same with a different capitalization or with a space). All of these seem to point in the direction of these attention heads being copying matrices, but it's not clear that any of them is a fully robust formalization of "the primary behavior of this matrix is copying". It's worth noting that all of these potential notions of copying are linked by the fact that the sum of the eigenvalues is equal to the trace is equal to the sum of the diagonal.

For the purposes of this paper, we'll continue to use the eigenvalue-based summary statistic. We don't think it's perfect, but it seems like quite strong evidence of copying, and empirically aligns with manual inspection and other definitions.

There's often skepticism that it's even possible or worth trying to truly reverse engineer neural networks. That being the case, it's tempting to point at one-layer attention-only transformers and say "look, if we take the most simplified, toy version of a transformer, at least that minimal version can be fully understood."

But that claim really depends on what one means by fully understood. It seems to us that we now understand this simplified model in the same sense that one might look at the weights of a giant linear regression and understand it, or look at a large database and understand what it means to query it. That is a kind of understanding. There's no longer any algorithmic mystery. The contextualization problem of neural network parameters has been stripped away. But without further work on summarizing it, there's far too much there for one to hold the model in their head.

Given that regular one layer neural networks are just generalized linear models and can be interpreted as such, perhaps it isn't surprising that a single attention layer is mostly one as well.

Videos covering similar content to this section: [2 layer theory](https://www.youtube.com/watch?v=UM-eJbx_YDk&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51&index=5), [2 layer term importance](https://www.youtube.com/watch?v=qom0nxou4f4&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51&index=6), [2 layer results](https://www.youtube.com/watch?v=VuxANJDXnIY&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51&index=7)

Deep learning studies models that are deep, which is to say they have many layers. Empirically, such models are very powerful. Where does that power come from? One intuition might be that depth allows composition, which creates powerful expressiveness.

Composition of attention heads is the key difference between one-layer and two-layer attention-only transformers. Without composition, a two-layer model would simply have more attention heads to implement skip-trigrams with. But we'll see that in practice, two-layer models discover ways to exploit attention head composition to express a much more powerful mechanism for accomplishing in-context learning. In doing so, they become something much more like a computer program running an algorithm, rather than look-up tables of skip-trigrams we saw in one-layer models.

Recall that we think of the residual stream [as a communication channel](https://transformer-circuits.pub#residual-comms). Every attention head reads in subspaces of the residual stream determined by 

When attention heads do compose, there are three options:

Q- and K-Composition are quite different from V-Composition. Q- and K-Composition both affect the attention pattern, allowing attention heads to express much more complex patterns. V-Composition, on the other hand, affects what information an attention head moves when it attends to a given position; the result is that V-composed heads really act more like a single unit and can be thought of as creating an additional "virtual attention heads". Composing movement of information with movement of information gives movement of information, whereas attention heads affecting attention patterns is not reducible in this way.

To really understand these three kinds of composition, we'll need to study the OV and QK circuits again.

The most basic question we can ask of a transformer is "how are the logits computed?" Following [our approach](https://transformer-circuits.pub#onel-path-expansion) to the one-layer model, we write out a product where every term is a layer in the model, and expand to create a sum where every term is an end-to-end path through the model.

Two of these terms, the direct path term and individual head terms, are identical to the one-layer model. The final "virtual attention head" term corresponds to V-Composition. Virtual attention heads are conceptually very interesting, and we'll discuss them more later. However, in practice, we'll find that they tend to not play a significant role in small two-layer models.

Just looking at the logit expansion misses what is probably the most radically different property of a two-layer attention-only transformer: Q-composition and K-composition cause them to have much more expressive second layer attention patterns.

To see this, we need to look at the QK circuits computing the attention patterns. Recall that the attention pattern for a head 

But for the second layer QK-circuit, both Q-composition and K-composition come into play, with previous layer attention heads potentially influencing the construction of the keys and queries. Ultimately, 

One complicating factor is that we have to write it as a 6-dimensional tensor, using two tensor products on matrices. This is because we're trying to express a multilinear function of the form  

Each of these terms corresponds to a way the model can implement more complex attention patterns. In the abstract, it can be hard to reason about them. But we'll return to them with a concrete case shortly, when we talk about induction heads.

So far, we've developed a theoretical model for understanding two-layer attention-only models. We have an overall equation describing the logits (the OV circuit), and then an equation describing how each attention head's attention pattern is computed (the QK circuit). But how do we understand them in practice? In this section, we'll reverse engineer a single two-layer model.

Recall that the key difference between a two-layer model and a one-layer model is Q-, K-, and V-composition. Without composition, the model is just a one-layer model with extra heads.

Small two-layer models seem to often (though not always) have a very simple structure of composition, where the only type of composition is K-composition between a single first layer head and some of the second layer heads.*induction heads*.

The above diagram shows Q-, K-, and V-Composition between attention heads in the first and second layer. That is, how much does the query, key or value vector of a second layer head read in information from a given first layer head? This is measured by looking at the Frobenius norm of the product of the relevant matrices, divided by the norms of the individual matrices. For Q-Composition, 

One quick observation from this is that most attention heads are not involved in any substantive composition. We can think of them as, roughly, a larger collection of skip tri-grams. This two-layer model has a mystery for us to figure out, but it's a fairly narrowly scoped one. (We speculate this means that having a couple induction heads in some sense "outcompetes" a few potential skip-trigram heads, but no other type of composition did. That is, having more skip-trigram heads is a competitive use of second layer attention heads in a small model.)

In the next few sections, we'll develop a theory of what's going on, but before we do, we provide an opportunity to poke around at the attention heads using the interactive diagram below, which displays value-weighted attention patterns over the first paragraph of *Harry Potter and the Philosopher's Stone*. We've colored the attention heads involved in K-composition using the same scheme as above. (This makes it a bit hard to investigate the other heads; if you want to look at those, an interface for general exploration is available [here](https://transformer-circuits.pub/2021/framework/2L_HP_normal.html)). 

We recommend isolating individual heads and both looking at the pattern and hovering over tokens. For induction heads, note especially the off-diagonal lines in the attention pattern, and the behavior on the tokens compositing Dursley and Potters.

The above diagram shows the *value-weighted attention pattern* for various attention heads; that is, the attention patterns with attention weights scaled by the norm of the value vector at the source position 

The interface allows one to isolate attention heads, shows the overall attention pattern, and allows one to explore the attention for individual tokens. Attention heads involved in K-composition are colored using the same scheme as above. We suggest trying to isolate these heads.

If you look carefully, you'll notice that the aqua colored "induction heads" often attend back to previous instances of the token which *will* come next. We'll investigate this more in the next section. Of course, looking at attention patterns on a single piece of text — especially a well-known paragraph like this one — can't give us very high confidence as to how these heads behave in generality. We'll return to this later, once we have a stronger hypothesis of what's going on.

In small two-layer attention-only transformers, composition seems to be primarily used for one purpose: the creation of what we call induction heads. We previously saw that the one-layer model dedicated a lot of its capacity to copying heads, as a crude way to implement in-context learning. Induction heads are a much more powerful mechanism for achieving in-context learning. (We will explore the role of induction heads in in-context learning in more detail in our [next paper](https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html).)

If you played around with the attention patterns above, you may have already guessed what induction heads do. Induction heads search over the context for previous examples of the present token. If they don't find it, they attend to the first token (in our case, a special token placed at the start), and do nothing. But if they do find it, they then look at the next token and copy it. This allows them to repeat previous sequences of tokens, both exactly and approximately.

It's useful to compare induction heads to the types of in-context learning we observed in one layer models:

`[b] … [a] → [b]``[ab] … [a] → [b]``[a][b] … [a] → [b]`
The two-layer algorithm is more powerful. Rather than generically looking for places it might be able to repeat a token, it knows how the token was previously used and looks out for similar cases. This allows it to make much more confident predictions in those cases. It's also less vulnerable to distributional shift, since it doesn't depend on learned statistics about whether one token can plausibly follow another. (We'll see later that induction heads can operate on repeated sequences of completely random tokens)

The following examples highlight a few cases where induction heads help predict tokens in the first paragraph of Harry Potter:

Raw attention pattern and logit effect for the induction head `1:8` on some segments of the first paragraph of *Harry Potter and the Philosopher's Stone*. The "logit effect" value shown is the effect of the result vector for the present token on the logit for the next token, 

Earlier, we promised to show induction heads on more tokens in order to better test our theory of them. We can now do this.

Given that we believe induction heads are attending to previous copies of the token and shifting forward, they should be able to do this on totally random repeated patterns. This is likely the hardest test one can give them, since they can't rely on normal statistics about which tokens typically come after other tokens. Since the tokens are uniformly sampled random tokens from our vocabulary, we represent the `<n>`, with the exception of the special token `<START>`. (Notice that this is totally off distribution. Induction heads can operate on wildly different distributions as long as the more abstract property that repeated sequences are more likely to reoccur holds true.)

As in our previous attention pattern diagram, this diagram shows the value-weighted attention pattern for various heads, with each head involved in K-composition colored by our theory. Attention heads are shown acting on a random sequence of tokens, repeated three times.`<n>` denotes the 

This seems like pretty strong evidence that our hypothesis of induction heads is right. We now know what K-composition is used for in our two layer model. The question now is *how* K-composition accomplishes it.

The central trick to induction heads is that the key is computed from tokens shifted one token back.

The following example, from a larger model with more sophisticated induction heads, is a useful illustration:

QK circuits can be expanded in terms of tokens instead of attention heads. Above, key and query intensity represent the amount each token increases the attention score. Logit effect is the OV circuit.

The minimal way to create an induction head is to use K-composition with a previous token head to shift the key vector forward one token. This creates a term of the form ["copying matrix"](https://transformer-circuits.pub#copying-matrix) — then this term will increase attention scores when the previous token before the source position is the same as the destination token. (Induction heads can be more complicated than this; for example, other 2-layer models will develop an attention head that attends a bit further than the previous token, presumably to create a 

Our mechanistic theory suggests that induction heads must do two things:

Although we're not confident that the eigenvalue summary statistic from the [Detecting Copying section](https://transformer-circuits.pub#copying-matrix) is the best possible summary statistic for detecting "copying" or "matching" matrices, we've chosen to use it as a working formalization. If we think of our attention heads as points in a 2D space of QK and OV eigenvalue positivity, all the induction heads turn out to be in an extreme right hand corner.

One might wonder if this observation is circular. We originally looked at these attention heads because they had larger than random chance K-composition, and now we've come back to look, in part, at the K-composition term. But in this case, we're finding that the K-composition creates a matrix that is extremely biased towards positive eigenvalues — there wasn't any reason to suggest that a large K-composition would imply a positive K-composition. Nor any reason for all the OV circuits to be positive.

But that is exactly what we expect if the algorithm implemented is the described algorithm for implementing induction.

Earlier, we decided to ignore all the "virtual attention head" terms because we didn't observe any significant V-composition. While that seems like it's probably right, there's ways we could be mistaken. In particular, it could be the case that while every individual virtual attention head wasn't important, they matter in aggregate. This section will describe an approach to double checking this using ablations.

Ordinarily, when we ablate something in a neural network, we're ablating something that's explicitly represented in the activations. We can simply multiply it by zero and we're done. But in this case, we're trying to ablate an implicit term that only exists if you expand out the equations. We could do this by trying to run the version of a transformer described in our equations, but that would be horribly slow, and get exponentially worse as we considered deeper models.

But it turns out there's an algorithm which can determine the marginal effect of ablating the 

(Note that freezing the attention patterns to ground truth is what makes this ablation only target V-composition. Although this is in some ways the simplest algorithm, focused on the OV circuit, variants of this algorithm could also be used to isolate Q- or K-composition.)

As the V-composition results suggested, the second order “virtual attention head” terms have a pretty small marginal effect in this model. (Although they may very well be more important in other — especially larger — models.)

We conclude that for understanding two-layer attention only models, we shouldn’t prioritize understanding the second order “virtual attention heads” but instead focus on the direct path (which can only contribute to bigram statistics) and individual attention head terms. (We emphasize that this says nothing about Q- and K-Composition; higher order terms in the OV circuit not mattering only rules out V-Composition as important. Q- and K-Composition correspond to terms in the QK circuits of each head instead.)

We can further subdivide these individual attention head terms into those in layer 1 and layer 2:

This suggests we should focus on the second layer head terms.

Although virtual heads turned out to be fairly unimportant for understanding the performance of the two layer model, we speculate that these may be much more important in larger and more complex transformers. We're also struck by them because they seem theoretically very elegant.

Recall that virtual attention heads were the terms of the form 

Where Q- and K-Composition affect the attention pattern, V-Composition creates these terms which really operate as a kind of independent unit which performs one head operation and then the other. This resulting object really is best thought of as the composition of the heads, 

There are two things worth noting regarding virtual attention heads.

Firstly, this kind of composition seems quite powerful. We often see heads whose attention pattern attends to the previous token, but not heads who attend two tokens back — this may be because any useful predictive power from the token two back is gained via virtual heads. Attention patterns can also implement more abstract things, such as attending to the start of the current clause, or the subject of the sentence — composition enables functions such as ‘attend to the subject of the previous clause’.

Secondly, there are a lot of virtual attention heads. The number of normal heads grows linearly in the number of layers, while the number of virtual heads based on the composition of two heads grows quadratically, on three heads grows cubically, etc. This means the model may, in theory, have a lot more space to gain useful predictive power via the virtual attention heads. This is particularly important because normal attention heads are, in some sense, “large”. The head has a single attention pattern determining which source tokens it attends to, and 

Over the last few sections, we've made progress on understanding one-layer and two-layer attention-only transformers. But our ultimate goal is to understand transformers in general. Has this work actually brought us any closer? Do these special, limited cases actually shed light on the general problem? We'll explore this issue in follow-up work, but our general sense is that, yes, these methods can be used to understand portions of general transformers, including large language models.

One reason is that normal transformers contain some circuits which appear to be primarily attentional. Even in the presence of MLP layers, attention heads still operate on the residual stream and can still interact directly with each other and with the embeddings. And in practice, we find instances of interpretable circuits involving only attention heads and the embeddings. Although we may not be able to understand the entire model, we're very well positioned to reverse engineer these portions.

In fact, we actually see some analogous attention heads and circuits in large models to those we analyzed in these toy models! In particular, we'll find that large models form many induction heads, and that the basic building block of their construction is K-composition with a previous token head, just as we saw here. This appears to be a central driver of in-context learning in language models of all sizes – a topic we'll discuss in our next paper.

That said, we can probably only understand a small portion of large language models this way. For one thing, MLP layers make up 2/3rds of a standard transformer's parameters. Clearly, there are large parts of a model’s behaviors we won’t understand without engaging with those parameters. And in fact the situation is likely worse: because many attention heads interact with the MLP layers, the fraction of parameters we can understand without considering MLP layers is even smaller than 1/3rd. More complete understanding will require progress on MLP layers. At a mechanistic level, their circuits actually have a very nice mathematical structure (see [additional intuition](https://transformer-circuits.pub#additional-intuition)). However, the clearest path forward would require individually interpretable neurons, which we've had limited success finding.

Ultimately, our goal in this initial paper is simply to establish a foothold for future efforts on this problem. Much future work remains to be done.

The [Distill Circuits thread](https://distill.pub/2020/circuits/) 

The Circuits approach needs to be significantly rethought in the context of language models. Attention heads are quite different from anything in conv nets and needed a new approach. The linear structure of the residual stream creates both new challenges (the lack of a privileged basis removes some options for studying it) but also creates opportunities (we can expand through it). Having circuits which are bilinear forms rather than just linear is also quite unusual (although it was touched on by Goh et al.

We've noticed several interesting, high-level differences between the original circuits work on InceptionV1 and studying circuits in attention-only transformer language models:

Previous work by the LessWrong user Nostalgebraist on a method they call the ["Logit Lens"](https://www.lesswrong.com/posts/AcKRB8wDpdaN6v6ru/interpreting-gpt-the-logit-lens)

Our approach could be seen as making a similar observation, but deciding that the residual stream isn't actually the fundamental object to study. Since it's the sum of linear projections of many attention heads and neurons, it's natural to simply multiply out the weights to look at how all the different parts contributing to the sum connect to the logits. One can then continue to exploit linear structure and try to push linearity as far back into the model as possible, which roughly brings one to our approach.

Our work follows the lead of several previous papers in investigating transformer attention heads. Investigation of attention patterns likely began with visualizations by Llion Jones 

The largest difference between these previous analyses of attention heads and our work really seems to be a matter of goals: we seek to provide an end-to-end mechanistic account, rather than empirically describe attention patterns. Of course, this preliminary paper is also different from this prior work in that we only studied very small toy models, and only really then to illustrate and support our theory. Finally, we focus on autoregressive transformers, rather than a denoising model like BERT 

Our investigations benefitted from these previous papers, and we have some miscellaneous thoughts on how our results relate to them:

An important line of work critiques the naive interpretation of attention weights as describing how important a given token is in effecting the model's output (see empirically 

Our framework might be thought of as offering — for the limited case of attention-only models — a typology of ways in which naive interpretation of attention patterns can be misleading, and a specific way in which they can be correct. When attention heads act in isolation, corresponding to the first order terms in our equations, they are indeed straightforwardly interpretable. (In fact, it's even better than that: as we saw for the one-layer model, we can also easily describe how these first order terms affect the logits!) However, there are three ways that attention heads can interact (Q-, K-, and V-Composition) to produce more complex behavior that doesn't map well to naive interpretation of the attention patterns (corresponding to an explosion of higher-order terms in the path expansion of the transformer). The question is how important these higher-order terms are, and we've observed cases where they seem very important!

Induction heads offer an object lesson in how naive interpretation of attention patterns can both be very informative and also misleading. On the one hand, the attention pattern of an induction head itself is very informative; in fact, for a non-trivial number of tokens, model behavior can be explained as "an induction head attended to this previous token and predicted it was going to reoccur." But the induction heads we found are totally reliant on a previous token head in the prior layer, through K-composition, in order to determine where to attend. Without understanding the K-composition effect, one would totally misunderstand the role of the previous token head, and also miss out on a deeper understanding of how the induction head decides where to attend. (This might also be a useful test case for thinking about gradient-based attribution methods; if an induction head confidently attends somewhere, its softmax will be saturated, leading to a small gradient on the key and thus low attribution to the corresponding token, obscuring the crucial role of the token preceding the one it attends to.)

The research on attention heads mentioned above is often grouped in a larger body of work called "Bertology", which studies the internal representations of transformer language model representations, especially BERT [fantastic review](https://arxiv.org/pdf/2002.12327.pdf) by Rogers et al. 

Our work in this paper primarily intersected with the attention head analysis aspects of Bertology for a few reasons, including our focus on attention-only models, our decision to avoid directly investigating the residual stream, and our focus on mechanistic over top-down probing approaches.

Our work leverages a number of mathematical observations about transformers to reverse engineer them. For the most part, these mathematical observations aren't in themselves novel. Many of them have been implicitly or explicitly noted by prior work. The most striking example of this is likely Dong et al. 

There are many additional approaches to neural network interpretability, including:

It seems to us that interpretability research is deeply linked with visualizations and interactive interfaces supporting model exploration. Without visualizations one is forced to rely on summary statistics, and understanding something as complicated as a neural network in such a low-dimensional way is very limiting. The right interfaces can allow researchers to rapidly explore various kinds of high-dimensional structure: examine attention patterns, activations, model weights, and more. When used to ask the right questions, interfaces support both exploration and rigor.

Machine learning has a rich history of using visualizations and interactive interfaces to explore models (e.g. 

Some recent proposed improvements to the transformer architecture have interesting interpretations from the perspective of our framework and findings:

Inspired by the original [Circuits Thread](https://distill.pub/2020/circuits/) and [Distill's Discussion Article experiment](https://distill.pub/2019/advex-bugs-discussion/), transformer circuits articles sometimes include comments and replications from other researchers, or updates from the original authors.

Chris Olah was one of the authors of the original paper.

Since the publication of this paper, a significant amount of follow-up work has greatly clarified and extended the preliminary ideas we attempted to explore. We briefly summarize several salient lines of research below, as of February 2023.

Understanding MLP Layers and Superposition. The biggest weakness of this paper has been that we have little traction on understanding MLP layers. We speculated this was due to the phenomenon of superposition. Since publication, more has been learned about MLP layer neurons, there has been significant elaboration on the theory of superposition, and alternative theories competing with superposition have been proposed.

Attention Head Composition and Circuits. A [preliminary investigation](https://www.youtube.com/watch?v=4O-JhroSAwA) by Turner explored the idea of attention head composition in more detail. A paper by [Wang](https://arxiv.org/pdf/2211.00593.pdf) [et al.](https://arxiv.org/pdf/2211.00593.pdf) described a complex circuit of attention heads (however it is only analyzed on a narrow sub-distribution).

Induction Heads. We published a [follow-up paper](https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html) exploring how much induction heads contribute to in-context learning. A number of researchers reproduced the general results about induction heads. [Chan](https://www.lesswrong.com/posts/j6s9H9SHrEhEfuJnq/causal-scrubbing-results-on-induction-heads) [et al.](https://www.lesswrong.com/posts/j6s9H9SHrEhEfuJnq/causal-scrubbing-results-on-induction-heads) use their method of "causal scrubbing" to more rigorously characterize induction heads. A paper by [von Oswald](https://arxiv.org/abs/2212.07677) [et al.](https://arxiv.org/abs/2212.07677) demonstrated that a repeated induction head-like mechanism could learn a linear model in-context by simulating gradient descent. Parallel to all this, induction heads have increasingly been cited in discussion of "whether neural networks understand", seemingly because they're an interesting concrete middle ground of neural networks "implementing an algorithm" (see e.g. Raphaël Millière's talk at [this workshop](https://compositionalintelligence.github.io/)).

Following the publication of this paper, we became aware of a bug in an underlying library we wrote. This only affected [one diagram](https://transformer-circuits.pub#composition-diagram-caption), but does impact on our interpretation of the "Two-Layer Attention Only Transformers" section in some ways. In particular, there is more attention head composition going on in that model than it seemed.


Technical Details

Our analysis required us to manipulate low-rank matrices efficiently (as discussed in the appendix section ["Working with Low-Rank Matrices"](https://transformer-circuits.pub#working-with-low-rank-matrices)). In order to do this, we wrote a library for manipulating "strands" of matrix multiplications which result in low-rank matrices. For several computations, there's an identity where one can transpose a matrix product to compute it more efficiently – for example for the trace (

As a result, instead of computing terms of the form 

It’s perhaps easiest to understand the consequence of this by considering the composition of an induction head and a previous token head. What we accidentally computed is a measure of the extent to which the induction head’s query is built from the same subspace that the previous token head moved data into. This is specifically what an induction head should do with the previous token’s K-composition term, and so it very strongly responds to it while filtering out other kinds of composition.

While this highlighted induction head composition (and implicitly computed something quite interesting), it is different from the attention head composition we meant to compute.

Effect of Bug

Only one diagram was affected by this bug. The original plot and corrected plot are shown below:

While the importance of K-composition with the previous token head remains unchanged, we see some additional attention head composition. The main addition is that two of the induction heads also rely on a head which attends to the last several tokens, rather than just the previous token. This additional composition is consistent with our discussion of how induction heads in other models will often use more complex circuits than the "minimal" K-composition with a previous token head (see ["How Induction Heads Work"](https://transformer-circuits.pub#how-induction-heads-work)).

This article has a companion [problem set](https://transformer-circuits.pub/2021/exercises/index.html) of problems we found helpful in building our intuition for attention heads.

We've released one of the libraries we made for our research, [PySvelte](https://github.com/anthropics/PySvelte) (which makes it convenient to use interactive visualizations from Python), and written an article describing another one called [Garcon](https://transformer-circuits.pub/2021/garcon/index.html) (our tooling for probing the internals of models, especially large ones).

In writing this paper, our thinking was greatly clarified and encouraged by correspondence with Martin Wattenberg, Vladimir Mikulik, Jeff Wu, Evan Hubinger, and Peter Hase; their generous and detailed feedback exceeded anything one might reasonably hope for.

We're also deeply grateful to Daniela Amodei, Jia Yuan Loke, Liane Lovitt, Timothy Telleen-Lawton, Kate Rudolph, Matt O’Brien, Jeffrey Ladish, Samuel Bowman, Geoffrey Irving, Tom McGrath, Michela Paganini, Paul Christiano, Allan Dafoe, Gabriel Goh, Nick Cammarata, Chelsea Voss, Katherine Lee, Beth Barnes, Jan Leike, Tristan Hume, Nate Thomas, Buck Shlegeris, Alex Tamkin, Quinn Tucker, and Rob Harries, for their support, for comments on this work, and for conversations that contributed to the background thinking on interpretability and safety this work is based on.

Neel Nanda would like to extend particular thanks to Jemima Jones for providing significant motivation, accountability and support as he navigated a difficult period while contributing to this paper.

We're grateful to several readers for reporting errata. Ken Kahn found several typos in the paper. Tuomas Oikarinen caught several typos in the [accompanying exercises](https://transformer-circuits.pub/2021/exercises/index.html). 

Matthew Rahtz reported several broken links.

Theoretical Framework: The framework described in this paper developed as part of ongoing conversations and empirical investigation between Nelson Elhage, Catherine Olsson, Neel Nanda, and Chris Olah over the last year as they've tried to understand language models. It's impossible to disentangle all the contributions that contributed to its development, but some examples of the kind of contributions involved follow. Nelson Elhage discovered evidence of neurons performing memory management in the residual stream, informing the "residual stream as a communication channel" framing. Catherine Olsson carefully characterized dozens of neurons and attention heads (with other authors also helping out), greatly informing our overall thinking on how to think about different parts of the model. Neel Nanda greatly improved our thinking on how attention heads compose. Chris Olah developed the foundational ideas of the framework, discovered induction heads, and guided theoretical progress. These examples are only a small slice of the contributions each mentioned author made.

In addition to these core research contributors, other members of Anthropic contributed important insights to the framework. Jared Kaplan, Sam McCandlish, Andy Jones, and Dario Amodei contributed especially important insights.

Analysis of Toy Models: The analysis of toy models presented in this paper was done by Chris Olah and Neel Nanda. However, they were heavily based on previous experiments, including many by Nelson Elhage and Catherine Olsson. The special attention-only toy models used for this article were created by Nelson Elhage.

Writing: This article was drafted by Chris Olah. Neel Nanda made major pedagogical improvements to the draft. Dario Amodei contributed heavily to the high-level framing. Nelson Elhage did careful editing to improve clarity. Many members of Anthropic, including Nicholas Joseph, Zac Hatfield-Dodds and Jared Kaplan provided valuable feedback throughout the writing process. Others, including Deep Ganguli, Jack Clark, Ben Mann, Danny Hernandez, Liane Lovitt, and Tom Conerly significantly improved exposition by testing and giving feedback on approaches to explaining these ideas.

Infrastructure for Analyzing Models: The crucial piece of software infrastructure which made our analysis of models possible is a library called Garcon (described in [another article](https://transformer-circuits.pub/2021/garcon/index.html)) created by Nelson Elhage with help from Tom Brown, Sam McCandlish, and Chris Olah. Garcon makes it easy to access model activations and parameters, regardless of scale. Nelson and Catherine Olsson also built an internal website, based on top of Garcon, which allows users to see dataset examples as well as interactively edit text and see model activations and attention patterns. Although not presented here, this website was very important to the evolution of our thinking.

Visualizations: Having interactive visualizations to explore model activations was important in enabling us to explore large amounts of data. Chris Olah created PySvelte (see [github](https://github.com/anthropics/PySvelte)) and many of our visualizations, with contributions from Nelson Elhage and Catherine Olsson. Conceptual illustrations of ideas in this article are by Chris Olah. Ben Mann set up infrastructure to allow convenient secure sharing of visualizations.

Model Training: All our interpretability research is enabled by having access to models to study, including large models. (Although the models used as examples in this particular paper are small, our framework was informed by experiments on large models.) Led by Tom Brown, Sam McCandlish, and Jared Kaplan, the majority of Anthropic's technical staff contributed to the development of our efficient distributed training infrastructure and the underlying machine learning. Core contributors include Nicholas Joseph, Tom Henighan, and Ben Mann. Nelson Elhage, Kamal Ndousse, Andy Jones, Zac Hatfield-Dodds, and Danny Hernandez also contributed to this infrastructure.

Cluster: Tom Henighan and Nova DasSarma, advised by Tom Brown and Sam McCandlish with contributions from many others at Anthropic, managed the research cluster our research depended on and maintained its stability. Nova provided important support when interpretability required unorthodox jobs to run on our cluster.

Other Contributions:

All staff at Anthropic provided valuable feedback throughout. Dario Amodei, Tom Brown, Jack Clark, Jared Kaplan, Sam McCandlish all provided essential high-level feedback.


Chris Olah led the project.

Please cite as:

Elhage, et al., "A Mathematical Framework for Transformer Circuits", Transformer Circuits Thread, 2021.

BibTeX Citation:

```
@article{elhage2021mathematical,
   title={A Mathematical Framework for Transformer Circuits},
   author={Elhage, Nelson and Nanda, Neel and Olsson, Catherine and Henighan, Tom and Joseph, Nicholas and Mann, Ben and Askell, Amanda and Bai, Yuntao and Chen, Anna and Conerly, Tom and DasSarma, Nova and Drain, Dawn and Ganguli, Deep and Hatfield-Dodds, Zac and Hernandez, Danny and Jones, Andy and Kernion, Jackson and Lovitt, Liane and Ndousse, Kamal and Amodei, Dario and Brown, Tom and Clark, Jack and Kaplan, Jared and McCandlish, Sam and Olah, Chris},
   year={2021},
   journal={Transformer Circuits Thread},
   note={https://transformer-circuits.pub/2021/framework/index.html}
}
```
This article has focused on attention-only transformers, without MLP layers. How can we extend this approach to understanding MLP models with MLP layers?

Recall that an MLP layer 

The presence of the GeLU activation means we can’t linearize through MLP layers as we did attention layers. Instead, we likely need to take the approach of the Circuits project in reverse engineering vision models

In theory, there’s a lot of reason to be optimistic about understanding these neurons. They have an activation function which should encourage features to align with the basis dimensions. They’re four times larger than the residual stream, and information doesn’t need to flow through them, which are both factors one might expect to reduce polysemanticity. Unfortunately, things are much more challenging in practice. We’ve found the neurons much harder to develop hypotheses for, with the exception of neurons at ~5% depth through the model, which often respond to clusters of short phrases with similar meanings. We’ve focused this paper on understanding attention heads because we got a lot more early traction on attention heads.

Despite this, it’s worth noting that there’s a fairly clean story for how to mechanistically reason about neurons, which we can use when we find interpretable neurons.

Path Expansion of MLP Layer in One-Layer Model: For simplicity, let’s consider a standard one layer transformer (ignoring layer norm and biases). The pre-activation values are a linear function of the residual stream. By applying our equation for attention heads, we get essentially the same equation we got for the [one-layer transformer logits](https://transformer-circuits.pub#onel-path-expansion):

This means we can study it with the same methods. [Virtual Weights and Convolutional-Like Structure](https://transformer-circuits.pub#virtual-weights-and-convolution-like-structure), will explore this connection in more detail.

What about the downstream effect of these neurons? 

This approach becomes more difficult as a network gets deeper. One begins to need to reason about how one MLP layer affects another downstream of it. But the general circuits approach seems tractable, were it not for neurons being particularly tricky to understand, and there being enormous numbers of neurons.

When we apply path expansion to various terms in a transformer, we generally get virtual weights of the form:

(In the general case, 

The main example we've seen in this paper is having 

Multiplying by 

Exact Equivalence to Standard Convolution in Limited Cases: We claim that all convolutions can be expressed as one of the tensor products above, and thus (1) attention is a generalization of convolution; (2) specific configurations of attention correspond exactly to convolution.

Consider the convolution 

where 

This equivalence is explored in depth by Cordonnier et al. [many 2D relative position heads](https://epfml.github.io/attention-cnn/), similar to the previous token head we've observed.

Analogy to Convolution Generalizations: The correspondence to standard convolution breaks down for "dynamic" attention heads which attend based on some pattern other than fixed relative positions. However, it seems like there are many cases where they still are spiritually very convolution-like. For example, a "previous verb" head isn't a literal relative position (at least if we parameterize our input based on token index), but it seems morally like a type of convolution.

One way to resolve the above might be to consider graph convolutions. If we treat each attention pattern as a different set of weights on a graph, we could treat the tensor product as a sum of graph convolutions. (For an informal discussion of analogies between transformers and graph neural networks, readers may wish to look at an article by Joshi 

But the fundamental thing is that when you see tensor products like the ones above, they can be thought of as something like a convolution with dynamic, soft, possibly semantic positions.

Why This is Useful: In reverse engineering convolutional neural networks, the Distill Circuits Thread 

In the above example, we see the weights between two curve detectors; along the tangent of the curve, the other curve excites it. This would be much harder to understand if one had to look at 25 different weights; we benefit from organization.

We expect that similar organization may be helpful in understanding transformer weights, especially as we begin considering MLP layers.

We often find it helpful to think differently about various activations in transformers based on whether they have the following properties:

Privileged Basis vs Basis Free: A privileged basis occurs when some aspect of a model’s architecture encourages neural network features to align with basis dimensions, for example because of a sparse activation function such as ReLU. In a transformer, the only vectors with privileged bases are tokens, attention patterns and MLP activations. See the appendix table of variables for a full list of which activations have or don’t have privileged activations.

Some types of interpretability only make sense for activations with a privileged basis. For example, it doesn't really make sense to look at the "neurons" (ie. basis dimensions) of activations like the residual stream, keys, queries or values, which don't have a privileged basis. This isn't to say that there aren't ways to study them; a lot of interesting work is done on word embeddings without assuming a privileged basis (e.g. 

Bottleneck Activations: We say that an activation is a bottleneck activation if it is a lower-dimensional intermediate between two higher dimensional activations. For example, the residual stream is a bottleneck activation because it is the only way to pass information between MLP activations, which are typically four times larger than it. (Additionally, in addition to it being a bottleneck for the communication between adjacent MLP layers, it’s also the only pathway by which arbitrary early MLP layers can communicate with arbitrary late MLP layers, so the stream may simultaneously be conveying different pieces of information between many different pairs of MLP layers, a much more extreme bottleneck than just 4x!) Similarly, a value vector is a bottleneck activation because it’s much lower dimensional than the residual stream, and it’s the only way to move information from the residual stream for one token position in the context to another (without the model dedicating an additional attention head to the same pattern, which it sometimes will). See above table for a list of which activations are or are not bottleneck activations.

Our models use a slightly unusual positional mechanism (similar to 

At their most basic level, positional embeddings are kind of like token addresses. Attention heads can use them to prefer to attend to tokens at certain relative positions, but they can also do much more:

It's worth noting that one could simplify all the math in this paper by introducing an "identity attention head," 

| Main Model Activations and Parameters |  |  | 
| Variable | Shape / Type | Description | 
|  |  | Transformer logits, for tokens  | 
|  |  | One-hot encoded tokens [activation, privileged basis] | 
|  |  | “Residual stream” or “embedding” vectors of model at layer  | 
|  |  | Token embedding [parameter] | 
|  |  | Positional embedding [parameter] | 
|  |  | Unembedding / softmax weights [parameter] | 
| Attention Heads Activations and Parameters |  |  | 
| Variable | Shape / Type | Description | 
|  | Set | Set of attention heads at layer  | 
|  |  | Output of attention head  | 
|  |  | Attention pattern of attention head  | 
|  |  | Query, key, value and result vectors of attention head  | 
|  |  | Query, key, and value weights of attention head  | 
|  |  | Output weights of attention head  | 
| MLP Layer Activations and Parameters |  |  | 
| Variable | Shape / Type | Description | 
|  |  | Output of MLP layer  | 
|  |  | Activations of MLP layer  | 
|  |  | Input weights for MLP layer  | 
|  |  | Output weights for MLP layer  | 
| Functions |  |  | 
| Variable | Shape / Type | Description | 
|  | Function | Gaussian Error Linear Units  | 
|  | Function | Softmax with autoregressive mask for attention distribution creation. | 

In machine learning, we often deal with matrices and tensors where we want to multiply "one side". In transformers specifically, our activations are often 2D arrays representing vectors at different context indices, and we often want to multiply "per position" or "across positions". Frequently, we want to do both! Tensor (or equivalently, Kronecker) products are a really clean way to denote this. We denote them with the 

Very pragmatically:

There are several completely equivalent ways to interpret these products. If the symbol is unfamiliar, you can pick whichever you feel most comfortable with:

`A[:,:,None,None] * W[None, None, :, :]` (although one wouldn’t computationally represent them in that form). More formally, The models used as examples in this paper are zero, one, and two layer decoder-only, attention-only transformers 

Models have a context size of 2048 tokens and use dense attention. (Dense attention was preferred over sparse attention for simplicity, but made a smaller context preferable.) We use a positional mechanism similar to Press et al. [pointer-arithmetic based algorithms](https://transformer-circuits.pub#pointer-arithmetic) without the distorted QK matrices like rotary.)

The training dataset is as described in Kaplan et al. 

Our transformer models apply layer normalization every time they read off from the residual stream. In practice, this is done to encourage healthy activation and gradient scales, making transformers easier to train. However, it also means that the "read" operation of each layer is more complicated than the idealized version we've described above. In this section, we describe how we work around this issue.

Before considering layer normalization, let's consider the simpler case of batch normalization. Transformers typically use layer normalization over batch normalization due to subtleties around the use of batch normalization in autoregressive models, but it's an easy case to work out and would make our theory exactly align with the model. Batch normalization tracks a moving estimate of the mean and variance of its inputs, and uses that to independently normalize them, before rescaling with learned parameters. At inference time, a fixed estimate of the mean and variance is used. This means that it's simply a fixed linear transformation and bias, which can be multiplied into the adjacent learned linear transformation and bias. As such, batch normalization in models can typically be absorbed into other operations and ignored for the purposes of interpretability on the inference time model.

Layer normalization is a little more subtle. For each residual stream vector, we subtract off the average activation, normalize by variance, and then multiply by a learned set of diagonal weights and add a learned bias vector. It turns out that subtracting off the average activation is a fixed linear transformation, since it just zeros out a single dimension in the vector space. This means that everything except for normalizing by variance, layer normalization applies a fixed affine transformation. Normalizing by variance multiplies the vector by a scalar, and multiplying by a scalar commutes with all the other operations in a path. As a result, we can fold everything but normalization into adjacent parameters, and then think of the normalization scaling as a variable reweighting of the set of path terms going through that layer normalization. (The main downside of this is that it means that paths going through different sets of layers aren't easily comparable; for example, we can't trivially compare the importance of virtual attention heads composed of heads from different layers.)

A final observation is that layer normalization in the first attention layer of the model can alternatively be applied to every vector of the embedding matrix and then ignored. In some cases, this is more convenient (for example, we do this for skip-trigrams in our one-layer models).

In the future, it might be worth trying to train models with batch normalization to avoid this as a pain point, despite the complexities it presents.

In this paper, we find ourselves dealing with very large, but extremely low-rank matrices. Picking the right algorithm is often the difference between painfully slow operations on the GPU, and everything being instantaneous on your CPU.

First, we recommend keeping matrices in factored form whenever possible. When multiplying a chain of matrices, identify the "smallest" bottleneck point and represent the matrix as a product 

Eigenvalues: Exploit the fact that 

SVD: The SVD can also be calculated efficiently, and a variant can be used to efficiently compute PCA:

## Correction

The following diagram has an error introduced by a bug in an underlying library we wrote to accelerate linear algebra on low-rank matrices. A detailed comment on this, along with a corrected figure, can be found below.
