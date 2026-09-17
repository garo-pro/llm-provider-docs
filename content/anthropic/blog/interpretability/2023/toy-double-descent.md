Title: Superposition, Memorization, and Double Descent

URL Source: https://transformer-circuits.pub/2023/toy-double-descent

Markdown Content:
In a [recent paper](https://transformer-circuits.pub/2022/toy_model/index.html) 

Why should mechanistic interpretability care about overfitting? Despite overfitting being a central problem in machine learning, we have little mechanistic understanding of what exactly is going on when deep learning models overfit or memorize examples. Additionally, previous work has hinted that there may be an important link between overfitting and learning interpretable features 

So understanding overfitting is important, but why should it be relevant to superposition? Consider the case of a language model which verbatim memorizes text. How can it do this? One naive idea is that it might use neurons to create a lookup table mapping sequences to arbitrary continuations. For every sequence of tokens it wishes to memorize, it could dedicate one neuron to detecting that sequence, and then implement arbitrary behavior when it fires. The problem with this approach is that it's extremely inefficient – but it seems like a perfect candidate for superposition, since each case is mutually exclusive and can't interfere.

In this note, we offer a very preliminary investigation of training the same toy models in our previous paper on limited datasets. Despite being extremely simple, the toy model turns out to be a surprisingly rich case study for overfitting. In particular, we find the following:

We hypothesize that real neural networks perform operations in a sparse, high-dimensional “feature” space, but these features are difficult for us to see directly because they’re stored in superposition. Motivated by this, we attempt to simulate this feature space using synthetic input vectors [previous paper](https://transformer-circuits.pub/2022/toy_model/index.html#demonstrating-setup-x)). Concretely, 

We consider the ["](https://transformer-circuits.pub/2022/toy_model/index.html#demonstrating-setup-model)[ReLU Output](https://transformer-circuits.pub/2022/toy_model/index.html#demonstrating-setup-model)[" toy model](https://transformer-circuits.pub/2022/toy_model/index.html#demonstrating-setup-model), defined as

where 

In this work we limit ourselves to uniform importance 

We use 50,000 full-batch updates, as opposed to mini-batch, using the AdamW 

Unless otherwise specified, we use a weight decay of 

In the "normal superposition" we described in our previous paper, we found that the model embeds more features than it has dimensions, often mapping them to regular polytopes. For example, if the model has a two dimensional hidden space, sparse features will be organized as a pentagon:

But what happens if we train models on finite datasets instead? It turns out that the models we find will often look very messy and confusing if you try to look at them from the perspective of features, but very simple and clean if you look at them from the perspective of data point activations.

Let's visualize a few ReLU-output models trained on datasets of different sizes, with many sparse features. We'll focus on models with [Colab notebook](https://colab.research.google.com/drive/1AREdeODhgsQ_ukqPKnhWQ4bijVqTa4eW?usp=sharing) for more details). Highly sparse features increase the probability of orthogonal data points. Having lots of features avoids the trivial case where data points are just a single feature activating. Ideally, we'd have preferred to just use enough features that the central limit theorem causes data points to have similar norms, but achieving this with high sparsity would require annoyingly large vectors.

The data points – rather than the features – are being represented as polytopes!

What happens as we make the dataset larger? Clearly our toy models behave very differently in the small data regime where they "use data points as features" and the infinite data regime where they learn the real, generalizing features. What happens in between?

In the original paper, the notion of "feature dimensionality" was helpful for studying how the geometry of features varies as we change models. For this note, we'll extend our notion of feature dimensionality (which we will denote as 

We can now visualize how the geometry of features and data points changes as we vary the size of the dataset. In the middle pane below is a scatter plot of both feature and training-example dimensionalities for varying the dataset size (we will discuss the test loss in the top pane in a later section).

In the small data regime on the left, we see that while the feature dimensionalities are small, the training-example dimensionalities follow [Colab notebook](https://colab.research.google.com/drive/1AREdeODhgsQ_ukqPKnhWQ4bijVqTa4eW?usp=sharing) for more details.

In the large data regime on the right, we see 5 features whose dimensionalities are large, while the rest of the feature and training-example dimensionalities are small. The blue vector plots show that those 5 features are represented in a pentagon, while the rest are largely ignored. We provide some intuition as to why one should expect this ~5 feature solution in this [colab](https://colab.research.google.com/drive/1PTGgQt6OuWfAPi2iNn_myB4gQo-8ORAI?usp=sharing). The fractional dimension of the pentagon features is notably less than the expected 2/5. We believe this is due to there being many other features (9,995) whose individually small contributions add-up to a significant fraction of the denominator in 

Most data examples have nonzero values for only zero or one of the 5 pentagon features, causing the hidden-vectors to also trace out a pentagon in the bottom-right red subfigure. The outliers represent rare cases with >1 nonzero values.

In between these two extremes, things are messier and harder to interpret.

We did not use a consistent scale for the red and blue vector plots in the previous figure. Using a consistent scale (see below figure) reveals that lengths of both hidden and feature vectors vary widely with dataset size, peaking around 

A few comments on these trends:

The phenomenon of models behaving very differently in two different regimes, with strange behavior in between, is eerily reminiscent of double descent 

For a given 

It’s interesting to note that we’re observing double-descent in the absence of label noise. That is to say: the inputs and targets are exactly the same. Here, the “noise” arises from the lossy compression happening in the downprojection. It is impossible to encode 10,000 features into 2 neurons with a linear projection, even in the sparse limit. Thus the reconstruction is necessarily imperfect, giving rise to unavoidable reconstruction error and consequently, double-descent 

At this point, it's natural to wonder whether the double descent might be an artifact of only having 

We visualize double descent as a two-dimensional function varying both the number of training examples, 

There are clearly regions where "double descent" occurs – regions where bigger models or more data hurt performance.

Consistent with prior work on double descent, these results are sensitive to weight decay and the number of training epochs. [In the appendix](https://transformer-circuits.pub#wd-epochs-double-descent), we show that for 

We find that, in toy models, memorization can be understood as models learning "single data point features" in superposition. These models exhibit double descent as they transition from this strategy of representing data points to representing features.

There is much more to explore. The most obvious question is whether the naive mechanistic theory of overfitting that these results suggest generalizes at all to real models. But there's also a lot to ask in the context of toy models:

Inspired by the original [Circuits Thread](https://distill.pub/2020/circuits/) and [Distill's Discussion Article experiment](https://distill.pub/2019/advex-bugs-discussion/), the authors invited several external researchers who we had previously discussed our preliminary results with to comment on this work. Their comments are included below.

[Marius Hobbhahn](https://www.mariushobbhahn.com/) is a PhD student at the University of Tuebingen.

I replicated most findings in the “Superposition, Memorization, and Double Descent” paper. I changed the setup by reducing the sparsity and the number of features by 10× respectively. I still find the double descent phenomenon as described in the paper with very similar constellations for features and hidden vectors. I also found double descent in multiple other settings, e.g. with different loss functions or when adding a ReLU activation between the layers. My preliminary takeaway from these findings is that the double descent is a fairly regular phenomenon that we should expect to happen in many settings. (Details can be found in my post [More Findings on Memorization and Double Descent](https://www.lesswrong.com/posts/KzwB4ovzrZ8DYWgpw/more-findings-on-memorization-and-double-descent).)

[Adam Jermyn](https://adamjermyn.com/) is an independent researcher focused on AI alignment and interpretability.

One question I had reading this paper is: what sets the scale at which models learn generalizing features? When I asked this, the authors proposed two potential hypotheses:

The first hypothesis predicts that increasing the weight decay rate should decrease the generalizing scale.

The figure below shows the dimensionalities of features for models trained with different weight decay rates. Lines show the maximum feature dimension and points and lines are colored by the weight decay rate.

The generalizing scale corresponds to a jump in the dimensionalities. Importantly this scale does not appear to change with the weight decay rate, which is evidence against the first hypothesis.

The second hypothesis predicts that the generalizing scale occurs once the dataset is large enough that it contains multiple instances of each feature. That is, it occurs at 

The figure below shows the dimensionalities of features for models trained with different weight decay rates. Lines show the maximum feature dimension and points and lines are colored by the feature frequency (

Indeed that appears to be the case! Models trained with very different sparsities learn generalizing features once datasets are large enough to see each feature roughly 10 times.

While this is suggestive, it is not clear that this is the whole story. For instance, for models with more hidden dimensions the dimensionality curves don’t lie as cleanly on top of each other (see below), and there are other trends that are puzzling (e.g. the peak feature dimensions decrease as the datasets grow post-generalization), so it seems possible that there is more going on.

When the authors shared a preliminary draft, they suggested it might be interesting to look at what happens when individual datapoints are repeated in the dataset.

When a datapoint appears a small number of times (2–3) the phenomenology is the same as in this paper, but with more repeats models switch to learning a combination of datapoints and generalizing features.

The figure below shows training histories of the feature and sample dimensions (left panels) as well as the final feature and sample embeddings (right panels) for a model with T=30,000 and a single feature (black) appearing 5 times. The repeated feature is embedded alongside four generalizing features and suppresses the fifth, effectively replacing one of the generalizing features that would ordinarily be learned.

When there are multiple repeated datapoints the model preferentially learns these, and each replaces a feature in the embedding space:

When there are more than five repeated datapoints, the model embeds five of them and all five features it would have learned are suppressed:

One question that arises here is: how many times does a datapoint need to appear before it is memorized? One way to think about this is in terms of the benefits of memorizing a datapoint versus learning a feature. Very roughly:

The figure below shows results from models trained with a single repeated feature and hidden dimension 

The data are somewhat noisy, which could in part reflect the difficulty optimizing in 

As one final observation, in models on the edge of memorizing datapoints we see a number of “near misses”, where the model memorizes a datapoint and then “decides” against it!

Interestingly, this phenomenon is mirrored by a phenomenon in models with no repeated datapoints in the intermediate dataset regime, where some models briefly learn generalizing features and then forget them by the end of training. This is shown in the movie below for a model trained with T=10,000:

Chris Olah is one of the authors of the original paper. This comment describes a small extension we may or may not expand on.

A natural question based on these results is whether we can detect overfit, memorized examples in real neural networks with data dimensionality. As a preliminary investigation, let's look at a model which is only slightly less of a toy – a one hidden layer MNIST model with 512 hidden units. Ordinarily, it's difficult to study superposition in real models because we don't know what the features are. But studying data points in superposition is an exception to this, since we do know what the data points are!

In practice, we expect that features in real models are not orthogonal and that even when a dataset example is memorized, it likely activates some “generalizing” features. To account for this, we'll slightly change our notion of data dimensionality to maximal data dimensionality:

The intuition is that if an example activates multiple features, the supremum can pick the one with the highest data dimensionality. (It turns out that this is closely related to the log-likelihood of data points if you fit a Gaussian to the network activations.)

Below, we plot the data dimensionality of all training examples. We see that most examples have roughly the same dimensionality, but there are a few outliers in both tails. Strikingly, the examples with unusually high data dimensionality – almost 100× higher than typical examples! – tend to be weird outliers. While far from conclusive, it's tempting to believe these examples are "memorized" examples which the model has "special cased".

In addition to detecting overfitting, one might also see this as an example of [mechanistic anomaly detection](https://ai-alignment.com/mechanistic-anomaly-detection-and-elk-fb84f4c6d0dc) – detecting that a model is making decisions for a different reason than it normally does. Of course, we don't mean to suggest that all cases of a model "triggering a special case" can be so easily detected. If anything, it may hint that mechanistic anomaly detection will be even harder than one might think, since it could be hidden by superposition.

I can replicate these results on maximal data dimensionality up to small deviations. I tested the properties of [More Findings on Maximal Data Dimension](https://www.lesswrong.com/posts/WfdxXhszxFc3BxZ8r/more-findings-on-maximal-data-dimension).)

Chris Olah and Tom Henighan are authors of the original paper.

When training 

Experiments – The experiments in this paper were conducted by Tom Henighan, with help from Tristan Hume and Nelson Elhage. This was based on a prediction by Chris Olah that this behavior could be observed in roughly this experimental setup. Actually eliciting this behavior required careful tuning of hyperparameters, which was done by Tom Henighan. Robert Lasenby contributed significantly to our theoretical understanding of why features organize into pentagons. Nicholas Schiefer helped clarify our interpretation of the experimental results. Stanislav Fort did an independent reproduction of some of the experiments in this paper.

Diagrams – Diagrams were made by Tom Henighan and Shan Carter, with some help from Chris Olah.

Writing – This paper was drafted by Tom Henighan, with significant editing from Chris Olah and Shan Carter. Other authors also contributed to editing.

We’re grateful to Sheer El Showk, James Sully, Adam Jermyn, Trenton Bricken, Neel Nanda, Eric Winsor, Ilya Sutskever, Gabriel Goh, William Saunders, Martin Wattenberg, Jacob Hilton, Sam Marks, Eric Neyman, Drake Thomas for useful comments and suggestions on early drafts.

We're also grateful to all our colleagues for creating a supportive environment for us to do our work: Jared Kaplan, Dario Amodei, Daniela Amodei, Jack Clark, Tom Brown, Sam McCandlish, Ben Mann, Nick Joseph, Catherine Olsson, Danny Hernandez, Amanda Askell, Kamal Ndousse, Andy Jones, Dawn Drain, Timothy Telleen-Lawton, Anna Chen, Yuntao Bai, Deep Ganguli, Liane Lovitt, Zac Hatfield-Dodds, Nova DasSarma, Jia Yuan Loke, Jackson Kernion, Tom Conerly, Scott Johnston, Jamie Kerr, Sheer El Showk, Shauna Kravec, Stanislav Fort, Rebecca Raible, Saurav Kadavath, Rune Kvist, Eli Tran-Johnson, Rob Gilson, Guro Khundadze, Ethan Perez, Sam Bowman, Sam Ringer, Jeeyoon Hyun, Michael Sellitto, Jared Mueller, Joshua Landau, Cameron McKinnon, Sandipan Kundu, Carol Chen, Roger Grosse, Robin Larson, Noemí Mercado, Anna Goldie, Azalia Mirhoseini, Jennifer Zhou, Erick Galankin, Dustin Li, James Landis, Neerav Kingsland, Tamera Lanham, Miranda Zhang, Bryan Seethor, Landon Goldberg, Brian Israel, Newton Cheng, Mike Lambert, Oliver Rausch, Matt Bell, Hongbin Chen, Kamile Lukosiute, Martin Lucas, Ivan Vendrov, Karina Nguyen, Peter Lofgren, Orowa Sikder, Logan Graham, Thomas Liao, and Da Yan.

Here we use 

Keeping other hyperparameters the same as the text, we find that decreasing weight decay increases the test loss bump, whereas increasing it all the way up to 1.0 removes it entirely. This is consistent with prior double-descent work.

Here we stick to a weight-decay of 1e-2, but vary the number of training epochs. We still use a linear warmup for the first 5% of training, followed by a cosine decay for the learning rate. Again we find results in-line with prior double-descent work: more epochs leads to a larger bump in test loss.

## Replication

Adam Jermyn is an independent researcher focused on AI alignment and interpretability.

After seeing preliminary results, I replicated the results in the section “How Do Models Change with Dataset Size?” for models with hidden dimensionm=2 . Overall I found good qualitative agreement. There are some quantitative differences between my results and those shown in the paper, but nothing that I expect to affect any of the conclusions. 

The figure below corresponds to the first figure in that section, and shows qualitatively similar features:

In particular, this replication shows the same division into three regimes, of memorizing samples from small datasets, learning generalizing features from large datasets, and doing something more complicated in between, and the sample and feature embeddings look qualitatively similar between my models and the ones shown in the paper.

There are three differences between this and the corresponding figure in the paper that I can see, and I think they may be related:

I ran my models multiple times and verified that the different instances replicate these differences. I have not been able to pin down where these differences come from, and as far as I can tell I have trained my models precisely as described in the text, though it is certainly possible that I have missed something!

I also reproduced the second figure of the same section:

The general trends are very similar. In particular:

There are again differences, though these are quantitative rather than qualitative. In particular, the peak bias norms in my models are roughly 3 times larger than those in the paper, and I see a rise in the weight norms over the range T=100–1000 whereas the figure in the paper shows more of a plateau.

Original Authors' Response: Thanks for replicating this! It's really nice to see that everything qualitatively reproduced. We're uncertain what caused the shift in the dataset size at which the transition occurs. It seems like there must be some hyperparameter difference between our setups, but we're uncertain what it is! However, since we only really care about the existence of the transition, and not exactly where it falls for this toy problem, we're not that concerned about identifying the exact difference.
