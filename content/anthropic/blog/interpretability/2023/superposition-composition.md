Title: Distributed Representations: Composition & Superposition

URL Source: https://transformer-circuits.pub/2023/superposition-composition

Markdown Content:
Distributed representations are a classic idea in both neuroscience and connectionist approaches to AI. We're often asked how our work on superposition relates to it. Since publishing our [original paper](https://transformer-circuits.pub/2022/toy_model/index.html) on superposition, we've had more time to reflect on the relationship between the topics and discuss it with people, and wanted to expand on our [earlier discussion](https://transformer-circuits.pub/2022/toy_model/index.html#related-codes) in the related work section and share a few thoughts. (We care a lot about superposition and the structure of distributed representations because decomposing representations into independent components [is necessary to escape the curse of dimensionality](https://transformer-circuits.pub/2022/mech-interp-essay/index.html) and understand neural networks.)

It seems to us that "distributed representations" might be understood as containing two different ideas, which we'll call "composition" and "superposition".

To make this concrete, we'll consider a few potential ways neurons might represent shapes of different colors. These lovely examples are borrowed from [Thorpe (1989)](https://persee.fr/doc/intel_0769-4113_1989_num_8_2_873), who created them to demonstrate various possibilities between the idea of a "local code" and a "distributed code" in neuroscience. Thorpe provides four example codes – "local", "semi-local", "semi-distributed", and "high-distributed". These might traditionally be seen as being on a spectrum between being "local" and "distributed". We'll consider these examples again and offer an alternative view in which the examples instead vary on two different dimensions of superposition and composition.

Following Thorpe, this note will focus on examples where neurons have binary activations. This significantly simplifies the space of possibilities, but it's still a rich enough space to have interesting questions.[linear representation](https://transformer-circuits.pub/2022/toy_model/index.html#motivation-directions) where features correspond to directions.

We'll start with what Thorpe calls a pure "local code". In this setup, there's a neuron for each (shape, color) pair. In our language, this model is "monosemantic" and doesn't have any superposition.

This example might seem silly and wasteful, but it is worth noting that this code actually has quite a few interesting properties! For example, it can be used to represent arbitrary sets of stimuli – for example, a red circle and a blue square. Alternatively, if another neural network layer is built on top of it, it's very interesting to ask what can be "linearly selected" from it, and this is also extremely flexible. A future neuron could use a linear function to do something like fire for black circles and white triangles, but not black triangles or white circles. This won't be possible with other codes!

This example is also an interesting example for thinking about the linear probes research methodology. In our toy example, it would be very reasonable to probe for the existence of a feature like "red", and in the above example, a linear probe would certainly be able to predict it! However, despite this, it doesn't seem like the above example should be understood as having a "red" feature.

Another natural option is to represent independent features (eg. "Green", "Square") with neurons and use composition to represent objects. Thorpe calls this a "semi-local code" and others might call it a "sparse code".

We see it as using a different set of features – colors and shapes, rather than (color, shape) pairs – which can compose together to represent (color, shape) stimuli.

This representation is "distributed" in the sense that representing an object is distributed over independent features composing together. This is often what people mean when they talk about distributed representations in machine learning. A classic example is Mikolov (2013) who found – to great excitement of the community! – that word embeddings represent properties of words like "gender" or "plural" as different direction vectors in embedding space.

There are several advantages for neural networks using "compositional" distributed representations:

Compositional representations are also extremely favorable from an interpretability perspective – we can [understand them in terms of individual components](https://transformer-circuits.pub/2022/mech-interp-essay/index.html), despite the potentially exponential space of composition.

The previous code needed fewer neurons than the "local" code, but there's another kind of distributed representation that can represent all colored shapes with only 4 neurons (just as 4 bits can represent 16 numbers). Such codes are sometimes called "dense". In our framing, it's maximal superposition.

Dense codes like this have a very cool advantage: they can represent far more, totally unrelated objects than there are neurons. But note that there are also lots of costs to this. It isn't possible to generically select most colors or shapes with a linear probe, let alone sets. You also generally can't represent objects "between" these features (eg. "purple square with rounded corners").

It's also worth noting that there's been a subtle change in this code, besides the use of superposition. It switched back to using the (color,shape) combination features from the "local" code, rather than the independent color and shape features of the "semi-local" compositional code. This is because, as we'll see in the next section, maximal superposition is only possible if there isn't composition. Thus, this code needs to use a non-composing set of features.

It's tempting to see these codes as being on a continuous spectrum in terms of how "distributed" they are. The "local" code activated one neuron at a time and needed many neurons. The "semi-local" code activated two neurons at a time and needed fewer neurons. The "highly-distributed" code seems to just take this trend further – it sometimes activates four neurons at a time, and needs the fewest neurons.

But we see the "semi-local" code and the "highly-distributed" code as actually taking two totally different, orthogonal strategies of composition and superposition. (Thorpe also has a "semi-distributed" code which we see as using a mixture of both strategies; we'll discuss it shortly.)

In fact, there's a sense in which superposition and composition are fundamentally competing as strategies. If one has 

That is, composition and superposition are two different ways to allocate the exponential volume. (The Toy Models of Superposition paper might be seen as fundamentally exploring this trade-off, although it casts things in terms of an equivalent idea of feature sparsity rather than composition.)

If it's true that "distributed representations" actually consist of two different, competing strategies with very different properties, it seems important to distinguish them. For example, sometimes distributed representations are described as providing both the benefits of non-local generalization and representing more features than neurons, but these properties seem to actually trade-off because one is associated with composition and the other with superposition.

Composition and superposition are competing strategies, but it turns out they aren't as completely incompatible as it might sound. It's often the case that these compositional representations will only use a small number of features at a time, and many features are mutually exclusive. There's "limited composition" in some sense. That is to say, features are sparse, typically having a value of 0.

When features are dense (ie. combinations are common), networks can only represent one feature per neuron. But when features are sparse (combinations of many features are rare) superposition [becomes possible](https://transformer-circuits.pub/2022/toy_model/index.html#demonstrating-basic-results).

One way you can think about this is that while 

Thorpe's final "semi-distributed" code might be seen as an example of this. It still represents things in terms of composing color and shape features, but puts the color features in mutual superposition, and the shapes in mutual superposition.

Although these units begin to have some meaning – they're polysemantic, but only respond to a few features – it seems more natural from our perspective to think of this code from the perspective of linear combinations of neurons (ie. directions) as corresponding to features. For example, A+B=Green and D+E=Square. This is what we mean by this code representing composable shape and color features in superposition.

It seems to us that there are two different things people call "distributed representations": superposition and composition. While it's possible to mix these, they're fundamentally in tension. Distinguishing them seems important because they have very different properties.

Understanding the structure of representations is important because they're fundamentally at the heart of efforts to reverse engineer and mechanistically understand neural networks. When neural network representations are distributed in the sense we call "composition", it is possible to understand them in terms of a linear number of independent features rather than an exponential amount of volume. This [seems critical](https://transformer-circuits.pub/2022/mech-interp-essay/index.html) if we hope to truly understand neural networks.

This is an informal note written by Chris Olah and published on May 4, 2023.

Informal notes allow us to publish ideas that aren't as refined and developed as a full paper. Mistakes are very possible! We'd ask you to treat this note as more like some ideas a colleague is informally sharing with you, or a typical internet blog post, rather than a paper.

This article was greatly improved by the editing of Joshua Batson, and more generally by the insightful feedback of Adam Jermyn, Martin Wattenberg, Neel Nanda, Samuel Marks, Trenton Bricken, Thomas Henighan, Shan Carter, Adly Templeton, Thomas Conerly, Brian Chen, Tristan Hume, and Robert Lasenby.

The ideas in this essay were influenced by conversations with too many colleagues and other researchers to have any hope of enumerating, but to whom I'm indebted.
