Title: A Toy Model of Mechanistic (Un)Faithfulness

URL Source: https://transformer-circuits.pub/2025/faithfulness-toy-model

Markdown Content:
Mechanistic faithfulness is the concern that when we replace model components with sparse approximations (such as transcoders), those replacements might implement different mechanisms than the original model. Every computer science student knows that there are many different algorithms which can achieve the same effect – sorting algorithms are a classic example of this! Why should we believe our transcoders learn the same mechanisms as the original underlying model?

The goal of this update is to describe mechanistic faithfulness, and to concretely illustrate it in a toy model. Although we've discussed [this briefly](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#limitations-faithfulness) in recent papers, we felt it deserved more depth. We'll also briefly explore the idea that "Jacobian matching" might help with this. Finally, we'll conclude with a discussion of how mechanistic faithfulness relates to the broader universe of concerns around SAEs and transcoders.

In 2024, a number of researchers introduced what we now call transcoders 

At the time, I think we underappreciated what a radical change this was. Certainly, I underappreciated this. We were switching from using SAEs to model representation to modeling computation.

When one models representation, much can be forgiven. Features are just a basis for the vector space of activations, and the primary concern is whether they make the activations understandable. As long as your features are monosemantic and explain the activations, it's a valid way to understand what the activations contain. Of course, it might prove hard to understand the model's computation in that basis, but you won't be misled.

However, when one models computation, it becomes possible to have a setup that is monosemantic, and has low MSE on distribution, but achieves this through a different computational mechanism. This is particularly concerning since the alternate mechanisms may generalize differently off distribution. This attacks one of the deepest hopes for mech interp: explanations we can really deeply trust.

This lack of mechanistic faithfulness is somewhat bounded by the fact that the mechanism can only diverge so far, since we're only replacing the model "one step at a time" – that is, every transcoder step predicts the ground-truth residual stream, after a single transcoder non-linearity, greatly limiting the space of mechanistic divergences. (If you force sorting algorithms to match their memory state after every computational step, that likely forces them to be the same!)

But for transcoders, we can clearly construct examples where they can and do mechanistically diverge despite this.

Our goal is to construct a toy model illustrating mechanistic (un)faithfulness. This means we need to have a toy model which studies computation, rather than representation, such that the computation can potentially be unfaithful!

One such toy problem is the absolute value problem. We can have a one layer model try to map 

This is exactly identical to 

One question we could ask is "if we don't give a model enough neurons to implement this solution, how does it do computation in superposition"? We're going to leave that question to another day, and instead ask "under what circumstances can a transcoder recover these features, given a perfect model (i.e. 

It turns out that our transcoders easily discover this solution by default.

(One important detail is that we're going to use tanh L1 regularization, in order to achieve closer results to L0 regularization. If we use standard L1 regularization, our transcoders essentially learn duplicate features with extra capacity, and are generally messier.)

In the basic problem setup, we'll use the following data distribution:

We're now going to add a special "memorization data point" to the mix. We'll define a special data point p which is 1 on the first 3 dimensions and 0 on the others:

And then have `repeat_frac` of the time.

If we repeat a data point like this, we begin to see a transcoder "memorization feature" circuit:

Notice how this feature specifically activates on p and then produces p!

We'll call this a "[datapoint feature](https://transformer-circuits.pub/2023/toy-double-descent/index.html)". 

Datapoint features like this are fine if they reflect the underlying model. Models very likely do memorize some datapoints, in which case we want features like this. But in this case, the model 

We now have an example of mechanistic unfaithfulness to explore. In the following sections, we'll investigate when these datapoint features form, and how we can get rid of them, with the goal of more generally resolving mechanistic faithfulness.

Before we run more experiments, it is useful to understand exactly when we should expect these datapoint features to form in a transcoder, so that we can calibrate our experiments. When they form from the transcoder memorizing a datapoint (rather than the underlying model doing so and the transcoder faithfully mimicking it) there are two necessary conditions:

If we do a scan over these two hyperparameters and measure the presence of datapoint features, we see that in fact it's the product of these two properties that controls whether datapoint features form (our measure of memorization is defined in the next section):

For any feature 

We'll define the memorization of a feature 

where 

In this section, we'll investigate our toy model in two regimes, one without repeated data points, and one with them. Our goal will be to mechanistically understand the resulting models, and directly observe the induction of data point features.

To aid us in this, we'll introduce a new visualization which will allow us to see how each transcoder feature maps to the features we think it might be natural to learn (

Below, you can see this visualization applied to a baseline setup, where we train a transcoder on the absolute value problem without any repetition.

This visualization is explained in detail below, but note for now that it shows a map between transcoder neurons, and the features we might expect to learn. On the x-axis we have transcoder neurons, and on the y-axis potential features/circuits. Without repetition, we see that the transcoder learns almost exactly the expected features 

The key idea is that we have a "library" of ideal circuits we think the transcoder might learn (the regular ReLU ones and memorization). We consider the "contribution" 

We can now ask what happens if we introduce a repeated data point! (We only use a relatively small amount of repeated data for this setup, 5%, such that memorization is marginal).

We can now see a data point feature (see bottom right), as expected. We also have features corresponding to the three individual features which are used in the repeated data point, but they are rotated so that they don't activate on the repeated data point.

This demonstrates that mechanistic faithfulness can be a real problem, even in very simple setups!

Mechanistic faithfulness is a serious problem, but we don't think the issue is hopeless. In this section, we propose a connection between mechanistic faithfulness and a layer's Jacobian, and then demonstrate that we can exploit this to solve mechanistic faithfulness, at least in some toy problems. (We don't claim this is the ideal solution, and indeed we'll note some issues with it. Our goal is simply to show that there are productive paths forward to explore.)

The connection to Jacobians may not be obvious at first. One way to see it is to consider how the features learned by transcoders in our previous two examples produce very different Jacobians on the memorized data point:

In the baseline case, we see a diagonal matrix (telling us the features are independent). But in the repeated data case, with the datapoint feature, we see that the "reason" each feature is active locally is due to all the other features (which should be irrelevant!).

There's a deeper technical reason this is principled. For a single layer, there's a very important way in which the Jacobian (as a function across data points) hits upon the nature of its mechanisms. Intuitively, if you want to understand the mechanisms, you should look at the weights – this is the motivation behind circuits work, and also behind recent "weight-based interpretability" (e.g. 

Given this observation, a natural response is what we'll call "Jacobian matching". We penalize the difference between our transcoder’s jacobian and the true jacobian:

In practice, this objective would be extremely expensive. Instead, we exploit that this is equivalent in expectation to:

It turns out that adding this to our loss dramatically helps our datapoint feature memorization metric.

Let's return to our repeated data point example, which produced a data point feature above. What happens if we regularize it to match the ground truth Jacobian? It turns out to basically fix the problem!

We see that this gets rid of the memorization feature, but that our transcoder features for the features used on the repeated point are a bit rotated (see residual at bottom left).

In the adversarial attack robustness literature, there's a problem called "gradient masking", where models will learn to manipulate their gradients. The general trick we observe is that they can create features with large weights and thus large gradients, but very negative biases so that they're barely active and don't contribute much to the L1 objective.

Note that this is exploiting a gap between our L1 objective, and the L0 objective we wish to optimize. Using tanh L1 helps significantly here, since it moves us closer to L0.

In principle, it seems like there are other optimization terms we could add to push back on this if it was an issue. For example, it might be interesting to penalize the L1 of different transcoder features contributions to the Jacobian, to penalize attempts to manipulate the gradient like this. This can be cheaply approximated, since it's just the norms of the input and output weights multiplied together, masked by the Jacobian of the transcoder activation function.

Surprisingly, we may not want to perfectly match the Jacobian. This is because we believe the models we study are in superposition, and we believe that doing superposition will "blur" the Jacobian.

One way to think about this is to ask what we expect the spectrum of the Jacobian to look like. Very naively, we might expect something like this:

The tail is an artifact of superposition, and we shouldn't care about modeling it. Concretely, it corresponds to features spread over multiple neurons, which also represent other features. If a single feature is active and it is spread over 10 neurons, we should expect one large singular value in the Jacobian corresponding to that feature, and 9 small singular values corresponding to other things those neurons help represent.

This intuition makes us believe that, while we want to push towards better Jacobian matching, we shouldn't expect to match it perfectly.

We've seen that mechanistic faithfulness can be a real concern!

In our thinking, it's become more than this. In the ongoing discourse on SAEs/transcoders and their potential weaknesses, we've found it helpful to refactor concerns into two broad categories:

In practice, many concerns people raise speak to both of these categories to some extent.

Both of these are major concerns, but we're presently most concerned with mechanistic faithfulness. Although success on it is probably more of a spectrum than binary, we see it as cutting at the very heart of why we care about mechanistic interpretability. Thankfully, it seems like there are promising paths forward.

The ideas described in this note were greatly shaped and informed by discussions with Josh Batson, Michael Sklar, Jack Lindsey, Adly Templeton, and Siddharth Mishra-Sharma. In particular, many of these individuals did very significant thinking expanding the ideas in this post, and pursuing the much more ambitious project of how they might relate to real models. The only reason they are not authors is that we scoped this post quite narrowly and didn't include various extensions, in the interests of publishing this earlier.

We're more broadly grateful for comments from the entire Anthropic interpretability team.
