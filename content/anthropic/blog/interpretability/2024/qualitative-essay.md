Title: Reflections on Qualitative Research

URL Source: https://transformer-circuits.pub/2024/qualitative-essay

Markdown Content:
This note offers some opinionated thoughts on why interpretability research may have qualitative aspects be more central than we're used to in other fields. It also aims to describe some heuristics for research taste in qualitative work.

Early scientific fields are often quite qualitative and become more quantitative as they mature. For example, discovering cells is a qualitative result, which can then mature (over many decades) into quantitative tools like counting white blood cells in cancer research. Discovering chemical spectral lines was a qualitative result, which only really became quantitative when Bohr realized that the "butterfly wings of atoms" gave insight into electron orbitals.

The vast majority of researchers are trained in mature disciplines, because genuinely new scientific fields are rare. These mature disciplines have established paradigms, with established quantitative measures and methods. But interpretability is not a mature field. It doesn't have an established paradigm. Even the most basic abstractions (does it make sense to think of a model in terms of "features"?) are up for debate.

There's a risk that our training from mature fields may give us the wrong instincts if we translate them into such an early, messy, unestablished science. In particular, we should expect to need to be guided a lot more by qualitative results.

To be clear, this isn't saying we should not do quantitative research when appropriate! And in fact, often these can be synergistic, with qualitative research helping us be confident we're using the right quantitative tools. (The line between them can also be blurry!) Rather, the goal of this note is simply to argue that qualitative results should genuinely be seen as first class citizens, and something we want to keep returning to as a touchstone to avoid becoming lost or fooling ourselves.

[Anscombe's Quartet](https://en.wikipedia.org/wiki/Anscombe%27s_quartet) is a famous example of how several radically different datasets can have the same mean, standard deviation, and correlation:

This reflects a more general lesson: summary statistics which boil rich high dimensional data into a single number will always blind you to most of what's going on. And so, you need to be very careful when you do so. And in particular, you need to be very careful that you know and trust what you're measuring if you're going to rely on it.

In established fields, there may be standard measurements and quantitative values that are very well understood – in what they mean, in how to think about them, and so on. But in interpretability we don't have that benefit.

A pattern we see in some interpretability and interpretability-adjacent ML papers is defining some metric which is claimed to correspond to some property of interest, and then very rigorously measuring this metric. We see this as a kind of [Cargo-Cult Science](https://calteches.library.caltech.edu/51/2/CargoCult.htm). It can seem very rigorous with lots of line plots with standard deviation bars and such. But it often isn't, because the critical weakness is whether the metric actually, reliably tracks the property of interest, not the rigor with which the metric is evaluated. A recent example of this in our own work was our study of [tanh-regularization in dictionary learning](https://transformer-circuits.pub/2024/feb-update/index.html#dict-learning-tanh), where summary statistics initially indicated a lot of promise and it was only later qualitative inspection of features that revealed that we had been led astray.

We suspect that often, in early stage scientific fields like interpretability, rigor involves much more qualitative work, with quantitative metrics growing out of that and initially being treated quite skeptically.

One of the reasons mature fields can rely so much on summary statistics is that they have a constrained hypothesis space.

Consider how mature fields often frame experiments as testing one hypothesis against another. For instance, a gravitational wave experiment might be set up to test General Relativity against Chern-Simons gravity. This makes sense when the hypothesis space is narrow and well-understood, where the bulk of the probability mass truly is on the few hypotheses being tested. In these cases it's often relatively straightforward to come up with a single measure that should be different between two hypotheses!

But in pre-paradigmatic fields we don’t know what hypotheses to consider! A physicist in the 1800s coming up with explanations for energy production in the Sun would have entirely missed nuclear physics.

So the goal of science in pre-paradigmatic fields is to first figure out what hypotheses we should be considering! And this means working in a rather different way. Whereas summary statistics can be very good for discriminating between a small number of hypotheses, individual numbers don’t provide a rich enough signal to orient us in a vast space of possibilities.

Why are summary statistics so popular? One reason involves the ecosystem of interfaces for scientific research. We don't often think of things this way, but scientific research implicitly involves interfaces for thinking about data. For example equations, line plots, and terminology are all interfaces. [Media for Thinking the Unthinkable](https://www.youtube.com/watch?v=oUaOucZRlmE) (for compelling discussion of interfaces in scientific thinking) and [Drawing Theories Apart](https://www.amazon.com/Drawing-Theories-Apart-Dispersion-Diagrams/dp/0226422674) (for discussion of "paper tool" interfaces).

The most common and reusable interfaces (eg. line plots) often require a scalar summary statistic. These simple datatypes are a kind of lingua franca of research because they're so common that they can be reused across almost scientific fields and have standardized practices. But they require us to reduce our results to a summary statistic.

Presenting more complex (and simply more massive) data requires custom interfaces. In fact, working with the scale of data we do in interpretability without reducing to summary statistics is almost impossible without custom and often interactive interfaces. This is why interpretability has been so intertwined with data visualization. Just as early chemistry depended on custom glassware – and indeed, many scientists did their own glasswork! – so too does interpretability depend on data visualization.

Another reason summary statistics are popular is that they are defensible. There is a broader scientific community and culture that implicitly tells us “If you present your data in one of several common formats, you are in the very worst case still doing science.”

In mature fields this makes sense: there has been convergence on the right summary statistics to use, their pitfalls are known, and the community knows how to interpret them. But in young fields we don’t know what summary statistics to use. And we can easily be led astray by numbers that don’t mean what we think, or that hide the core complexity we’re trying to study.

So while there are good reasons that mature fields lean heavily on summary statistics, and develop a culture that encourages them, that culture can be misleading in pre-paradigmatic fields, and in interpretability we need to be mindful of this.

How do we know if qualitative results are "real"? What does rigorous qualitative research look like?

We don't have tools like "statistical significance" to fall back on, and so it's easy for this research to seem non-rigorous. And yet the discovery of cells under a microscope was certainly rigorous! And likewise the discovery of stellar spectra, of superconductivity (“Is the resistance zero?”), and so on. Some of the most striking and world-changing discoveries in the history of science came in the form of qualitative results that didn’t need error bars or summary statistics because they were so striking.

In certain well established topics there might be known qualitative methods – for example noticing a new species in zoology – which leverage the fact that we really understand what we're observing to make a rigorous qualitative observation. But this, also, isn't applicable to interpretability.

So this brings us back to the original question – how can we know if qualitative results are real? We suspect that one of the most reliable ways to know that a qualitative result is trustworthy is what we'll call the signal of structure:

The signal of structure is any structure in one's qualitative observations which cannot be an artifact of measurement or have come from another source, but instead must reflect some kind of structure in the object of inquiry, even if we don't understand it.

You might think of this as the informal, "unsupervised" version of statistical significance. Whereas statistical significance tests a particular (hopefully pre-registered) hypothesis against a null hypothesis, the signal of structure observes an unpredicted high-dimensional pattern and rejects the hypothesis it was noise or an artifact, typically because the structure is so compelling and complex that it's clearly orders of magnitude past the bar.

The observation of cells can't be an artifact of the microscope – they're too complex! And it can’t be a noise – they’re too structured! An artifact like a lens flare can produce distortions, but not ones like this:

Likewise, [DeepDream](https://en.wikipedia.org/wiki/DeepDream) is too complex (and has no other source structure could come from) to be anything other than a reflection of some structure inside the network. Even if you don't know what the structure is, it can't be random noise!

Another example of this is the weights between [curve detectors](https://distill.pub/2020/circuits/curve-circuits/) in InceptionV1. The pattern is too complex to be noise. We could argue over interpretation, but there's clearly something there!

Note that all of these involve structure with overwhelming complexity and detail which is just far beyond anything that could happen by chance. They're so striking that they hold up as notable even if they're cherry picked: Hooke seeing cells through a microscope is just obviously showing something real, even if he looked at hundreds of other things that were uninteresting and didn't report on them.

A big part of the reason one can find this is because there's low hanging fruit – glaringly obvious patterns once one sees them. In mature fields, the bright beacons of the signal of structure will generally have already been found, and remaining lessons will involve trying to carefully tease things apart. But for interpretability, there's an incredible amount of low hanging fruit.

Noticing striking structure like this also happens in more quantitative work. For example, noticing that neural networks have such clean scaling laws when you look at a log-log plot is an example of the "signal of structure" – the structure must be telling you something! And in particular, such structure is often telling you about a new abstraction.

The critical thing is that you need to know the structure isn't coming from some other source. For example, certain saliency map methods can produce striking, seemingly highly structured images – but they're mostly reflecting the structure of the original image, so it's not clear they're really showing you striking structure about the network. This isn't to say that saliency maps are necessarily not showing something, just that the signal of structure can't give us confidence in this: we need some other principled argument or experiment. More generally, many interesting phenomena involve cases where there are multiple interacting objects (such as a neural network applied to a particular input), and these are very interesting, but the structure needs to be evaluated with respect to every source and whether some less interesting explanation is possible, before we can invoke the signal of structure.

It's worth noting that it's also exceedingly easy to fool oneself with qualitative research. (This is likely another reason why scientists are often skeptical of it!) The signal of structure is the easiest way we know of to find real phenomena, but it's not the only one, and when it can combine with others, it's even more compelling.

Some signs of good qualitative work include:

We usually want to see qualitative results shine on at least one of these (more is better of course!). Conversely, the following traits make us suspicious of qualitative work:

[The Structure of Scientific Revolutions](https://en.wikipedia.org/wiki/The_Structure_of_Scientific_Revolutions) – Especially the discussion of particular competing "schools" in research into electricity and how they focused on different phenomena and abstractions; and also early atomic physics and visualizing particles in cloud chambers. Note the general pattern of how this often starts with relatively qualitative observations, how the choice of what to focus attention on and reify into an abstraction is very central and enables more quantitative work. Once you recognize cells, you can count them.

[Drawing Theories Apart](https://www.amazon.com/Drawing-Theories-Apart-Dispersion-Diagrams/dp/0226422674) – Especially chapter 1 discussion of "paper tools". 

[Media for Thinking the Unthinkable](https://www.youtube.com/watch?v=oUaOucZRlmE) – If the idea of "interfaces" and research being linked is foreign, watch this talk! If you like this, consider also looking at [Up and Down the Ladder of Abstraction](http://worrydream.com/LadderOfAbstraction/).

[Proofs and Refutations](https://math.berkeley.edu/~kpmann/Lakatos.pdf) – An important part of research is stumbling around in the dark for the right definitions and abstractions. This play by Lakatos beautifully gets at this. You might see the interplay between concrete examples and definitions as analogous to qualitative investigation and the creation of summary statistics. We need to engage with examples to find the right definitions / statistics.

[Cargo Cult Science](https://calteches.library.caltech.edu/51/2/CargoCult.htm) – A classic essay by Richard Feynman about pro forma rigor in science.
