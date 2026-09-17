Title: When Models Manipulate Manifolds: The Geometry of a Counting Task

URL Source: https://transformer-circuits.pub/2025/linebreaks

Markdown Content:
Intelligent systems need perception to understand, predict, and navigate their environment. These sensory capabilities reflect what's useful for survival in a specific environment: bats use echolocation, migratory birds sense magnetic fields, Arctic reindeer shift their UV vision seasonally. But when your world is made of text, what do you see? Language models encounter many text-based tasks that benefit from visual or spatial reasoning: parsing ASCII art, interpreting tables, or handling text wrapping constraints. Yet their only “sensory” input is a sequence of integers representing tokens. They must learn perceptual abilities from scratch, developing specialized mechanisms in the process.

In this work, we investigate the mechanisms that enable Claude 3.5 Haiku to perform a natural perceptual task which is common in pretraining corpora and involves tracking position in a document. We find learned representations of position that are in some ways quite similar to the biological neurons found in mammals who perform analogous tasks (“place cells” and “boundary cells” in mice), but in other ways unique to the constraints of the residual stream in language models. We study these representations and find dual interpretations: we can understand them as a family of discrete features or as a one-dimensional “feature manifold”/“multidimensional feature” [What is a Linear Representation? What is a Multidimensional Feature?](https://transformer-circuits.pub/2024/july-update/index.html#linear-representations)

The task we study is linebreaking in fixed-width text. When training on source code, chat logs, email archives, scanned articles, or judicial rulings that have line width constraints, how does the model learn to predict when to break a line? 

To orient ourselves to the stages of the computation, we first studied the model using discrete dictionary features. In this frame, we can understand computation as an “attribution graph” 

The attribution graph shows how the model performs this task by combining features that represent different concepts it needs to track:

The attribution graph provides a kind of execution trace of the algorithm, showing on this prompt which variables are computed and from what. After finding large feature families involved in representing these quantities across a diverse dataset, we suspected a simpler lens might be provided in terms of lower-dimensional feature manifolds interacting geometrically. We found geometric perspectives on the following questions:

How does the model represent different counts? The number of characters in a token, the number of characters in the current line, the overall line width constraint, and the number of characters remaining in the current line are each represented on 1-dimensional feature manifolds embedded with high curvature in low-dimensional subspaces of the residual stream. These manifolds have a dual interpretation in terms of discrete features, which tile the manifold in a canonical way, providing approximate local coordinates. Manifolds with similar geometry arise for a variety of ordinal concepts, and a ringing pattern we see in the embedded geometry in all these cases is optimal with respect to a simple physical model (§[Representing Character Count](https://transformer-circuits.pub#char-count)).

How does the model detect the boundary? To detect an approaching line boundary, the model must compare two quantities: the current character count and the line width. We find attention heads whose QK matrix rotates one counting manifold to align it with the other at a specific offset, creating a large inner product when the difference of the counts falls within a target range. Multiple heads with different offsets work together to precisely estimate the characters remaining (§[Sensing the Line Boundary](https://transformer-circuits.pub#boundary)).

How does the model know if the next word fits? The final decision — whether to predict a newline — requires combining the estimate of characters remaining with the length of the predicted next word. We discover that the model positions these counts on near-orthogonal subspaces, creating a geometric structure where the correct linebreak prediction is linearly separable (§[Predicting the Newline](https://transformer-circuits.pub#prediction)).

How does the model construct these curved geometries? The curvature in the character count representation manifold is produced by many attention heads working together, each contributing a piece of the overall curvature. This distributed algorithm is necessary because individual components cannot generate sufficient output variance to create the full representation (§[A Distributed Character Counting Algorithm](https://transformer-circuits.pub#count-algo)).

We validate these interpretations through targeted interventions, ablations, and “visual illusions” — character sequences that hijack specific attention mechanisms to disrupt spatial perception (§[Visual Illusions](https://transformer-circuits.pub#illusion)).

Zooming out, we take several broader lessons from this mechanistic case study:

When Models Manipulate Manifolds. For representing a scalar quantity (e.g., integer counts from 

Duality of Features and Geometry. Dictionary features provide an unsupervised entry point for discovering mechanisms, and attribution graphs surface the important features for any particular prediction. Sometimes, discrete features (and their interactions) can be equivalently described using continuous feature manifolds (and their transformations). In cases where it is possible to explicitly parameterize the manifold (as with the various integer counts we study), we can directly study the geometry, making some operations clearer (e.g., boundary detection). But this approach is expensive in researcher time and potentially limited in scope: it's straightforward when studying known continuous variables but becomes difficult to execute correctly for more complex, difficult-to-parametrize concepts.

Complexity Tax. While unsupervised discovery is a victory in and of itself, dictionary features fragment the model into a multitude of small pieces and interactions – a kind of complexity tax on the interpretation. In cases where a manifold parametrization exists, we can think of the geometric description as reducing this tax. In other cases, we will need additional tools to reduce the interpretation burden, like hierarchical representations 

Natural Tasks. The crispness of the representations and circuits we found was quite striking, and may be due to how well the model does the task. Linebreaking is an extremely natural behavior for a pretrained language model, and even tiny models are capable of it given enough context. Studying tasks which are natural for pretrained language models, instead of those of more theoretical interest to human investigators, may offer promising targets for finding general mechanisms.

To enable systematic analysis, we created a synthetic dataset using a text corpus of diverse prose where we (1) stripped out all newlines and (2) reinserted newlines every 

Four score and seven years ago our⏎

fathers brought forth on this continent,⏎

a new nation, conceived in Liberty, and⏎

dedicated to the proposition that all⏎

men are created equal.

Claude 3.5 Haiku is able to adapt to the line length for every value of [Appendix](https://transformer-circuits.pub#appendix-task-performance)).

All features in the main text of this paper are from a 10 million feature Weakly Causal Crosscoder (WCC) dictionary 

We define the line character count (or character count) at a given token in a prompt to be the total number of characters since the last newline, including the characters of the current token.

A natural thing to check is if the model linearly represents the character count as a quantitative variable: that is, can we predict character count with high accuracy via linear regression on the residual stream? Yes: a linear probe fit on the residual stream after layer 1 has an 

Instead, we find a multidimensional representation of the character count that we will analyze from four perspectives:

Each of these perspectives provides a complementary view of the same underlying object. The feature perspective is valuable for getting oriented, the subspace is perfect for causal intervention, the manifold is helpful for understanding how the representation is constructed and then manipulated to detect boundaries, and the logistic probes are useful for analyzing the OV and QK matrices of the individual attention heads involved.

We begin with the features. In layers one and two, we found features that seemed to activate based on a token’s character position within a line. For example, in the attribution graph for the aluminum prompt, there were two features active on the final word “called” that seemed to fire when the line character count was between 35–55 and 45–65, respectively. To find more such features, we computed the mean activation of each feature binned by line character count. There were ten features with smooth profiles and large between-character-count variance, shown below:

We find these features especially interesting as they are quite analogous to curve-detector features in vision models 

In the [Appendix](https://transformer-circuits.pub#appendix-feature-splitting), we show these features are universal across dictionaries of different sizes, but that some feature splitting occurs with respect to the line width constraint.

We observe that character count feature activations rise and fall at an offset, with two features being active at a time for most counts. This pattern suggests that the features are reconstructing a curved continuous manifold, locally parametrized by the activity of the two most active features. Given that their joint activation profiles follow a sinusoidal pattern, we expect reconstructions to lie on a curve between adjacent feature decoders.

To visualize this, we first compute the average layer 2 residual stream for each value of line character count on our synthetic dataset. We compute the PCA of these 150 vectors, and find that the top 6 components capture 95% of the variance; we project data to that 6 dimensional subspace which we call the “character count subspace” (top 3 PCs on the left below, next 3 PCs on the right). We observe the data form a twisting curve, resembling a helix from the perspective of PCs 1–3 and a more complex twist from the perspective of PCs 4–6.

We also reconstruct the residual stream for each datapoint using only the 10 character count features identified above, and compute the average reconstructed residual stream. We project the resulting curve, along with the feature decoders, into the same subspace. We find that the average line character count vectors are quite closely approximated by the feature reconstruction, though with mild kinks near the feature vectors themselves, reminiscent of a spline approximation of a smooth curve. While the 10 feature vectors discretize the curve, interpolating between the 2–3 neighboring features which are active at a time allows for a high-quality reconstruction of 150 data points.

To validate our interpretation of the character count subspace, we perform a coarse-grained ablation and a fine-grained intervention.

Ablation Experiment.  For our ablation experiment, we zero ablate (from a single early layer) a 

Intervention Experiment.  As a more surgical intervention, we perform an experiment to modify the perceived character count at the end of the aluminum prompt (originally 42 characters). Specifically, we sweep over character counts 

We also train supervised logistic regression probes to predict character count.

When we look at the average responses of each probe to tokens with different line character counts, we see a striking pattern. In addition to a diagonal band (probes, like the sparse features, have increasingly wide receptive fields), we see two faint off-diagonal bands on each side! The response curve of each probe is not monotonically decreasing away from its max, but rebounds. This “ringing” turns out to be a natural consequence of embedding a “rippled” manifold into low dimensions.

We note that the cosine similarities of the mean activation vector (which form the helix-like curve visualized in PCA space above), the linear probe vectors, and feature decoder vectors all exhibit similar ringing patterns to the above figure.[ringing](<https://en.wikipedia.org/wiki/Ringing_(signal)>)” in the sense of signal processing, a transient oscillation in response to a sharp peak, such as in the Gibbs Phenomenon.

This structure turns out to be a natural consequence of having the desired pattern of similarity, trivially achievable in 150 dimensions, projected down to low dimensions. As a toy model of this, suppose that we wish to have a discretized circle's worth of unit vectors, each similar to its neighbors but orthogonal to those further away. This can be realized by a symmetric set of unit vectors in 150 dimensions with cosine similarity matrix [appendix](https://transformer-circuits.pub#appendix-gibbs).

Alternatively, one can view the ringing from the perspective of sparse feature decoders as a kind of interference weight 

Finally, we also construct a simple physical model showing that the rippling and ringing arise even when the solution is found dynamically, whenever many vectors are packed into a small number of dimensions. Below, we show the result of a simulation in which 100 points confined to a 

Of particular interest is the result from setting the ambient dimension to 3[More Sensory and Counting Representations](https://transformer-circuits.pub#appendix-sensory) in the appendix.

We now study how the character counting representations are used to determine if the current line of text is approaching the line boundary. To detect the line boundary, the model needs to (1) determine the overall line width constraint and (2) compare the current character count with the line width to calculate the characters remaining.

We find that newline tokens have their own dedicated character [counting features](https://transformer-circuits.pub#appendix-width-features) that activate based on the width of the line, counting the number of characters between adjacent newlines.

To better understand how these representations are related, we train 150 probes for each possible value of “Line Width” like we did for “Character Count”. Using the attribution graph, we identify an attention head which activates boundary detection features. We visualize both sets of counting representations directly using the first 3 components of their joint PCA in the residual stream (left) and in the reduced QK space of this boundary head (right).

We find that this attention head “twists” the character count manifold such that character count 

This plot shows that

As a consequence of the ringing in the character count representations, we also observe ringing in the inner products (see [Rippled Representations are Optimal](https://transformer-circuits.pub#rippled-representations) above). The model is robust to these off-diagonal interference terms via the softmax applied to attention scores.

We find that the model actually uses multiple boundary heads, each twisting the manifolds by a different offset to implement a kind of “stereoscopic” algorithm for computing the number of characters remaining.[Appendix](https://transformer-circuits.pub#appendix-twisting).

To better understand each boundary head’s output, we train a set of probes for each value of characters remaining in the line (i.e., the line width 

As predicted by our weights based analysis, we observe that boundary heads have distinct but overlapping response curves that “tile” the possible values of characters remaining.

It's worth understanding why the model needs multiple boundary heads rather than just one. If the model relied only on boundary head 0, it couldn't distinguish between 5 characters remaining and 17 characters remaining—both would produce similar outputs. By having each head's output vary most significantly in different ranges, their sum achieves high resolution across the entire relevant range of “Characters Remaining” values.

We can see this more clearly by plotting each head's output in the first two principal components of the characters remaining space (which captures 92% of the variance). Head 0 shows large variance in the [0, 10] and [15, 20] ranges, Head 1 varies most in the [10, 20] range, and Head 2 varies most in the [5, 15] range. While no single head provides high resolution across the entire curve, their sum produces an evenly spaced representation that covers all values effectively.

We validate the causal importance of this two-dimensional subspace by performing an ablation and intervention experiment. Specifically, we conduct the same experiments as before: ablate the subspace and measure its effect on loss by token (left) and precisely modulate the characters remaining estimate on the last token in the aluminum prompt by substituting mean activation vectors.

We are now in a position to understand two distinct but related questions: (1) why these counting representations are multidimensional and (2) why multiple attention heads are required to compute these multidimensional representations.

Geometric Computations – A multi-dimensional representation enables the model to rotate position encodings using linear transformations—something impossible with one-dimensional representations. For instance, to detect an approaching line boundary, the model can rotate the position manifold to align with line width, then use a dot product to identify when only a few characters remain. With a 1D encoding, linear operations reduce to scaling and translation, so comparing position against line width would just multiply the two values, producing a monotonically increasing result with no natural threshold. Higher dimensions beyond 2D allow the manifold to pack more information through additional curvature.

Resolution – For character counting, the model must distinguish between adjacent counts for a large range of character positions, as this determines whether the next word fits. In a one-dimensional representation, positions would be arranged along a ray, with each position separated by some constant [Rippled Representations are Optimal](https://transformer-circuits.pub#rippled-representations) above.) For counting the characters remaining, the dynamic range is smaller, and so the model is able to embed the representation in a smaller subspace as a result.

To achieve the curvature for necessary high resolution, multiple attention heads are needed to cooperatively construct the curved geometry of the counting manifold. An individual attention head's output is a linear combination of its inputs (weighted by attention and transformed by the OV circuit), and thus is fundamentally constrained by the curvature already present in those inputs. In the absence of MLP contributions to the counting representation, if the output manifold needs to exhibit substantial curvature, multiple attention heads need to coordinate—each contributing a piece of the overall geometric structure. We will see another example of distributed head computation in the section on the [Distributed Character Counting Algorithm](https://transformer-circuits.pub#count-algo).

How did we originally find this boundary detection mechanism? When we first computed an attribution graph, we saw several edges from the previous newline features and embedding to predict-newline features. QK attributions showed that the top key feature was a “the previous line was 40–60 characters long” feature and the top query feature was “the current character count is 35–50” feature. At any one time there were often multiple counting features active at different strengths, suggesting that these features might be discretizing a manifold.

The boundary heads cause a family of boundary detecting features to activate in response to how close the current line is to the global line width. That is, they sense the approaching line boundary or the reverse index of the line count. Investigating these three sets of feature families led us to the count manifolds which they sparsely parametrize, and investigating the relevant attention heads let us find the boundary heads.

Finally, we note that these boundary-sensing representations parallel a well-studied phenomenon in neuroscience: boundary cells 

The final step of the linebreak task is to combine the estimate of the line boundary with the prediction of the next word to determine whether the next word will fit on the line, or if the line should be broken.

In the attribution graph for the aluminum prompt, we see exactly this merging of paths. The most influential feature

While the boundary detector activates regardless of the next token length, break predictor features activate only if the next token will exceed the length of the current line (as in the Aluminum prompt), and hence upweight the prediction of a newline.[Appendix](https://transformer-circuits.pub#appendix-break-predictors).

What is the geometry underlying the model’s ability to determine if the next token will fit on the line? Put another way, how is the break predictor feature above constructed from the boundary detector and next-word features?

To study this, we compute the average activations at the end of the model (~90% depth) across all tokens for all values of characters remaining 

Now consider the pairwise sum of each possible character-remaining vector 

When we use the separating hyperplane from the PCA of these average embeddings on real data, we achieve an AUC of 0.91 on the ground truth of whether the next token should be a newline. This reflects both the error of the three dimensional classifier and the error from Haiku’s estimates of the next token.

If the length of the most likely next word is linearly represented, this scheme would allow the model to predict newlines when that word is longer than the length remaining in the line. One could imagine a more general mechanism where the model comprehensively redirects the probability mass from all words that exceed the line limit to the newline. Claude 3.5 Haiku does not seem to leverage such a mechanism: when we compare the predicted distribution of tokens at the end of a line to the distribution on an identical prompt with the newlines stripped, we find them to be quite different.

Having described how the various character counting representations are used, the last big remaining question is: how are they computed?

We will show how Haiku uses many attention heads across multiple layers to cooperatively compute an increasingly accurate estimate of the character count. This turned out to be the most complicated mechanism we studied, though there are many similarities with the boundary detection mechanism.

To get an intuitive understanding of the behavior of the heads important for counting, we project their outputs into the PCA space of the line character count probes.

To understand how the character count is computed, we start at the very beginning: the embedding matrix.

As before, we can train probes or compute the average weights for every distinct token length in the embedding. We visualize the token character count probes for character length 1–14 and visualize their top principal components. Using the first 3 principal components, which capture 70% of the variance, we see that embedding character counts are arranged in a circular pattern (PC1 vs PC2) with an oscillating component (PC3). This pattern is consistent with the ones observed in [Rippled Representations are Optimal](https://transformer-circuits.pub#rippled-representations).

As with all of the counting manifolds, we also find [features](https://transformer-circuits.pub#appendix-token-lengths) that discretize this space into overlapping notions of short, medium, and long words.

To understand the counting mechanism, we will work backwards from the summed attention outputs to the embedding. Notably, we:

We can decompose the sum above into the contribution from the output of each individual head in layer 0.

How do individual heads implement this behavior? We can break down the behavior of an individual head by analyzing its QK circuit (where it attends) and OV circuit (the linear transformation from the embeddings to the output) 

QK Circuit.  Each head 

OV Circuit. The OV circuit coordinates with the QK circuit to create a heuristic estimate based on the number of tokens in the line multiplied by the average token length (

Below, we include a detailed walkthrough of L0H1.

For a more detailed analysis of each head, see [The Mechanics of Head Specialization](https://transformer-circuits.pub#appendix-mechanics). Layer 1 attention heads perform a similar operation, but additionally leverage the initial estimate of the character count (see [Layer 1 Head OVs](https://transformer-circuits.pub#appendix-l1-attn)).

To compute the line width, the model seems to use a similar distributed counting algorithm to count the characters between adjacent newlines. However, one subtlety that we do not address in this work is how the line width is actually aggregated. It is possible that the model computes a global line width by taking the max over all line lengths in the document or uses an exponentially weighted moving average of the last several line lengths. We do note that the line width uses a partially disjoint set of heads, likely because the “attend to previous newline as a sink” mechanism needs modification when the current token is also a newline.

Humans are susceptible to “visual illusions” in which contextual cues can modulate perception in seemingly unexpected ways. Famous examples include the Müller-Lyer illusion, in which arrows placed on the ends of a line can alter the perceived length of the line 

Can we use our understanding of the character counting mechanism to construct a “visual illusion” for language models?

To get started, we took the important attention heads for character counting and investigated what other roles they perform on a wider data distribution. We identified instances in which heads that normally attend from a newline to the previous newline would instead attend from a newline to the two-character string @@. This string occurs as a delimiter in git diffs, a circumstance in which you might want to start your line count at a location other than the newline:

⏎@@-14,30 +31,24 @@ export interface ClaudeCodeIAppTheme {⏎

But what happens when this sequence appears outside of a git diff context—for instance, if we insert @@ in the aluminum prompt without changing the line length?

We find that it does modulate the predicted next token, disrupting the newline prediction! As predicted, the relevant heads get distracted: whereas with the original prompt, the heads attend from newline to newline, in the altered prompt, the heads also attend to the @@.

How specific is this result: does any pair of letters nonsensically inserted into the prompt fully disrupt the newline prediction? We analyzed the impact of inserting (at the same two positions) 180 different two-character sequences, half of which were a repeated character. We found that while most inserted sequences moderately impact the probability of predicting a newline, newline usually remains the top prediction. There was also no clear difference between sequences consisting of the same or different characters. However, a few sequences substantially disrupted newline prediction, most of which appeared to be related to code or delimiters of some kind: `` >> }} ;| || `, @@.

We further analyzed the extent to which there was a relationship between ‘distraction’ of the important attention heads and the impact on the newline prediction. Indeed we found that many of the sequences with potent modulation of newline probability—and especially code-related character pairs—also exhibited substantial modulation of attention patterns.

While in the aluminum prompt the task is implicit, this illusion generalizes to settings where the comparison task is made explicit. These direct comparisons are perhaps more analogous to the Ponzo, Sander, and Müller-Lyer illusions, where the perception and comparison is more direct.

These effects are robust to multiple choice orderings. Moreover, if the length of the text following the @@ exceeds that of the alternative choice, the alternative choice is selected as being shorter.

While we are not claiming any direct analogy between illusions of human visual perception and this alteration of line character count estimates, the parallels are suggestive. In both cases we can see the broader phenomena of contextual cues, and the application of learned priors about those cues, modulating estimates of object properties of entities. In the human case, priors such as three-dimensional perspective can influence perception of object size, or color constancy can influence estimates of luminance (such as in the checker shadow illusion). Here, one possible interpretation of our results is that mis-application of a learned prior, including the role of cues such as @@ in git diffs, can also modulate estimates of properties such as line length.

Objective.  This work is at the intersection of LLM “biology” (making empirical observations about what is going on inside models; e.g. 

Linebreaking. Michaud et al. 

Position.  Prior interpretability work on positional mechanism has largely focused on token position (e.g., 

Others have also studied, even going back to LSTMs, the existence of mechanisms in language models for controlling the length of output responses 

Geometry and Feature Manifolds.   Beyond position, there has been extensive work in understanding the geometric representation of numbers, especially in toy models (e.g., 

Multidimensional features with clear geometric structure have been found in more natural contexts 

Perhaps most relevant is recent work from Modell et al. 

Biological Analogues.  The geometric and algorithmic patterns we observe have suggestive parallels to perception in biological neural systems. Our character count features are analogous to place cells on a 1-D track 

In this paper, we studied the steps involved in a large model performing a naturalistic behavior. The linebreaking task, frequently encountered in training, requires the model to represent and compute a number of scalar quantities involving position in character count units that are not explicit in its input or output

Naturalistic Behavior and Sensory Processing. Deep mechanistic case studies benefit from choosing behaviors that the model performs consistently well, as these are more likely to have crisper mechanisms. This means prioritizing tasks that are natural in pretraining over tasks that seem natural to human investigators, and ideally, that are easily supervisable. As in biological neuroscience, perceptual tasks are often both natural and easy to supervise for interpretability (e.g., it is easy to modify the input in a programmatic way). Although we sometimes describe the early layers of language models as responsible for “detokenizing” the input 

The Utility of Geometry.  Many of the representations and computations we studied had elegant geometric interpretations. For example, the counting manifolds are the result of an optimal tradeoff between capacity and resolution, with deep connections to space-filling curves and Fourier features. The boundary head twist was especially beautiful, and after discovering one such head, we were able to correctly predict that there would need to be additional heads to provide curvature in the output. The distributed character counting algorithm was more complex, but we were still able to clarify our view by studying linear actions on these manifolds. For other computations, like the final breaking decision, the linear separation was clearly a part of the story but there must be some additional complexity we were not able to see yet to handle multitoken outputs. For the more semantic operations, we purely relied on the feature view. Of course, describing any behavior in full is immensely complicated, and there is a long list of possible subtleties we did not study: how the model accounts for uncertainty in its counting, its mechanism for estimating the line width given multiple prior lines of text, how it adapts to documents with variable line width, how it handles multiple plausible output tokens of different lengths or multitoken words, or various special cases (e.g., a LaTeX \footnote{} or a markdown link). For the inspired, we share transcoder attribution graphs for a fixed-width line break prompt on [Gemma 2 2B](https://www.neuronpedia.org/gemma-2-2b/graph?slug=fourscoreandseve-1757368139332&pruningThreshold=0.8&densityThreshold=0.99&pinnedIds=14_19999_37&clerps=%5B%5B%2214_200290090_37%22%2C%22nearing+end+of+the+line%22%5D%5D) and [Qwen 3 4B](https://www.neuronpedia.org/qwen3-4b/graph?slug=fourscoreandseve-1757451285996&pruningThreshold=0.8&densityThreshold=0.99&clerps=%5B%5B%2230_117634760_39%22%2C%22nearing+end+of+line%22%5D%5D&pinnedIds=30_15307_39), using the new neuronpedia interactive interface.

Unsupervised Discovery.  It likely would not have been possible to develop this clarity if it were not for the unsupervised sparse features. In fact, when we started this project, we attempted to just probe and patch our way to understanding, but this turned out poorly. Specifically, we did not understand what we were looking for (e.g. we didn’t know to distinguish line width vs. character count), where to look for it (e.g., we didn’t expect line width to only be represented on the newline), or how to look for it (we started by training 1-D linear regression probes). However, after identifying some relevant features but before spending substantial effort systematically characterizing their activity profiles, we were also confused by what they were representing. We saw dozens of features that were vaguely about newlines and linebroken text, but their differences were not obvious from flipping through the activating examples. Only after we tested these features on synthetic datasets did their role in the graph and the underlying computation become clear. We suspect better automatic labels 

Feature-Manifold Duality. The discrete feature and geometric feature-manifold perspectives offer dual lenses on the same underlying object. For example, in this work the model's representation of character count can be completely described (modulo reconstruction error) by the activities of the features we identified, where the action of the boundary heads is described by virtual weights that expand out the feature interactions via attention head matrices. The same character count representation can be described by a 1-dimensional feature manifold – a curve in the residual stream parametrized by the character count variable – where linear action of the boundary heads is described by continuous “twisting” of the manifold. In general, geometric structures learned by the model will likely admit both global parametrizations and local discrete approximations.

The Complexity Tax. Despite this duality, the descriptions produced by the two perspectives differ in their simplicity. The discrete features shatter the model into many pieces, producing a complex understanding of the computation. This seems like a general lesson. It seems like discrete features and attribution graphs may provide a true description of model computation, which can be found in an automated way using dictionary learning. Getting any true, understandable description of the computation is a very non-trivial victory! However, if we stop there, and don't understand additional structure which is present, we pay a complexity tax, where we understand things in a needlessly complicated way. In the line breaking problem, constructing the manifold paid down this tax, but one could imagine other ways of reducing the interpretation burden.

A Call for Methodology. Armed with our feature understanding, we were able to directly search for the relevant geometric structures. This was an existence proof more than a general recipe, and we need methods that can automatically surface simpler structures to pay down the complexity tax. In our setting, this meant studying feature manifolds, and it would be nice to see unsupervised approaches to detecting them. In other cases we will need yet other tools to reduce the interpretation burden, like finding hierarchical representations 

A Call for Biology. The model must perform other elegant computations. We can find these by starting with a specific task the model performs well, study this from multiple perspectives, develop methodology to answer the remaining questions, and relentlessly attempt to simplify our explanations. Because the investigation is grounded in specific examples of a behavior, it provides a fast feedback loop, can shed light on weaknesses of existing methods and inspire new ones, and can sharpen our conceptual language for understanding neural networks. We would be excited to see more deep case studies that adopt this approach.

For attribution in academic contexts, please cite this work as

Gurnee, et al., "When Models Manipulate Manifolds: The Geometry of a Counting Task", Transformer Circuits, 2025.

BibTeX citation

```
@article{gurnee2025when,
  author={Gurnee, Wes and Ameisen, Emmanuel and Kauvar, Isaac and Tarng ,Julius and Pearce, Adam and Olah, Chris and Batson, Joshua},
  title={When Models Manipulate Manifolds: The Geometry of a Counting Task},
  journal={Transformer Circuits Thread},
  year={2025},
  url={https://transformer-circuits.pub/2025/linebreaks/index.html}
}
```
We would like to thank the following people who reviewed an early version of the manuscript and provided helpful feedback that we used to improve the final version: Owen Lewis, Tom McGrath, Eric Michaud, Alexander Modell, Patrick Rubin-Delanchy, Nicholas Sofroniew, and Martin Wattenberg. We are also thankful to all the members of the interpretability team for their helpful discussion and feedback, especially Doug Finkbeiner for discussions of rippling and ringing, Jack Lindsey on framing, Tom Henighan for feedback on clarity, Brian Chen for improving the design of the figures and line edits of the text, and the team who built the attribution graph 

It is natural to ask if the character counting features are fundamental, or simply one discretization of the space among many. We found that dictionaries of different sizes learn features with very similar receptive fields, so this featurization – including the slowly dilating widths – is in some sense canonical. We hypothesize that this canonical structure emerges from boundary constraint: positions near zero (start of line) create a natural anchoring point for feature development.

The geometry of the decoder directions is also fairly consistent between the dictionaries, showing characteristic ringing.

However, we do see some evidence of feature splitting. For example, below are three character count features which activate on the same interval (~20–45 characters in the line), but differentially activate for lines of different widths: LCC2.a activates on all line widths, LCC2.b preferentially activates on long line widths, and LCC2.c preferentially activates when close to the line width boundary.

Recent work has raised the possibility that feature dictionaries could behave pathologically where there exist feature manifolds 

Line width features tile the space similar to character count features.

We simulate 

We explore a deeper connection between the ringing observed in the character count feature manifold, and a connection to Fourier analysis in an analytical construction.

Suppose that we wish to have a discretized circle's worth of unit vectors, each similar to its neighbors but orthogonal to those further away. Then the cosine similarity matrix [colab notebook](https://colab.research.google.com/drive/13L51UzyNQ6SnjNZRyhWsx_QuyQ3LoosW#scrollTo=c1q8ozayknvl)). Finally, because the [discrete Fourier transform diagonalizes circulant matrices](https://en.wikipedia.org/wiki/Circulant_matrix), the Fourier coefficients of 

One essential feature of the representation of line character counts is that the “boundary head” twists the representation, enabling each count to pair with a count slightly larger, indicating that the boundary is close. That is, there is a linear map QK which slides the character count curve along itself. Such an action is not admitted by generic high-curvature embeddings of the circle or the interval like the ones in the physical model we constructed. But it is present in both the manifold we observe in Haiku and, as we now show, in the Fourier construction. First, note permutating the coordinates of 

We evaluate whether a Fourier decomposition of the character count curve is optimal, and find that it is quite close given that it does not account for dilation. Fourier components explain at most 10% less variance than an equivalent number of PCA components, which are optimal for capturing variance.

Finally, we note that as one moves through layers, the representation becomes more peaked. This sharpening of the receptive field is useful to the model to better estimate character counts, and corresponds to higher curvature in the embedding and, as predicted by the model above, more pronounced ringing. Below we show cross-sections (at character count 30, 60, 90, 120) of the cosine similarity matrix of probes trained after layers 0, 1, 2, and 3. With each subsequent layer, the graphs get more tightly peaked and secondary rings go higher.

Different heads access and manipulate the space in different ways. Below, we show the cosine similarity of both probe sets through QK for three heads: one which keeps them aligned, one which shifts character count to align better with later line widths, and one which does the opposite.

We can also look at this transformation by visualizing the Singular Value Decomposition of each set of probes in a joint basis after passing them through QK. Once more, the alignment, left offset, and right offset can be read directly from the components.

We can directly plot the first 3 components of the joint probe space after passing them through each QK. Doing so shows that one head keeps the representations aligned, while the others twist them either clockwise or counterclockwise.

Boundary detector features (at about ~⅓ model depth) do not take into account the length of the next token.

Later in the model, there exist features which incorporate both the number of characters remaining and the length of the most likely next token. These features only activate when the most likely next token is longer than the number of characters remaining (i.e. below the red diagonal below), as is the case in our aluminum prompt.

We also found features for the converse: features which suppress the newline because the predicted next token is shorter than the number of characters remaining in the line.

Both break prediction and suppression features sometimes also have interpretable logit effects on the output of all tokens, not just the newline. For instance, the features below respectively excite and suppress the newline as their top effect, but also systematically suppress tokens with more characters. This is because if the model is wrong about the value of the next token (and whether it's a newline), the token must at least be short enough to fit on the line.

We find layer 0 features that activate as a function of the character count of individual tokens.

These features are overlapping (e.g. there are tokens for which the long word and medium word features are both active) and non-exhaustive (none of them fire on some common tokens, where we suspect the representation of character length is partially absorbed 

Heads collaborate to generate the count manifold, but how does each head aggregate counts?

As a toy model, consider the following construction for character counting with a single attention head:

This produces a ray with total length proportional to the character count of the line.

In practice, we observe that individual attention heads do indeed use the newline as an attention sink, but at different offsets. As an example, we visualize the attention patterns of 4 important Layer 0 heads on several prompts with different line widths (starting from the first newline in the sequence).

To characterize the mechanism more precisely, we compute the average attention as a function of the number of tokens since the previous newline and also as a function of the character length of individual tokens.

Similar to boundary detection, individual attention heads specialize in particular offsets to tile the space. Moreover, we observe that most of these attention heads have a bias towards attending to longer tokens.

In addition to QK, a head can change its output based on the OV circuit 

The output of each head can be thought of as having two components: (1) a character offset from the newline driven by the attention pattern and (2) an adjustment based on the actual character length of the tokens. Note that the average character count of a token is approximately 4.5 (and the median is 4), so we can interpret these effects shifting from a mean response (i.e. the transition point is always around count 4).

To walk through a head in action, consider the perspective of L0H1, which attends to the newline for the first ~4 tokens and then spreads out attention over the previous ~4–8 tokens:

Other heads perform a similar operation, except with different offsets depending on their newline sink behavior. Layer 1 heads also perform a similar operation, though they also can leverage the character count estimate of the Layer 0 heads (see [Layer 1 Head OVs](https://transformer-circuits.pub#appendix-l1-attn)).

Similar to the OVs of the Layer 0 attention heads, Layer 1 heads write to the character count features in accordance to how long the tokens they attend to are.

However, in addition to the token character length, Layer 1 heads also use the initial line length estimate constructed in Layer 0 to create a more refined estimate of the character count.

These repeated computations appear responsible for implementing the sharpening of representations.

Below, we show head sums for 3 different prompts with different line widths.

As before, we can look at their decomposition.

While in this work we carefully studied the perception of line lengths and fixed width text, there are many tasks which language models must perform which benefit from a positional, visual, or spatial representation of the underlying text. In the course of our investigation, we came across several other feature families and representations for these behaviors and report several below.

In addition to line width for tracking the absolute character length of a full line of text, there also exist features that are sensitive to lines which have ended early (i.e., lines where the character count is substantially shorter than the line width 

It is worth emphasizing that the role of these features, like others in this work, is not obvious from a typical workflow of quickly looking at dataset examples. It might be tempting to ignore these as "newline" features, but careful analysis yields quite clear behavior.

In addition to prose, language models must parse other kinds of more structured data like tables. Accurate prediction of a table’s content requires careful integration of row and column information (e.g. is this a column of text or numbers?). To facilitate this, we use a synthetic dataset of 20 markdown tables to find feature families which activate on separator tokens, specialized to particular rows or columns. Visualizing feature activations on each of these 20 tables (arranged by location in the table) showed clear patterns.

On a synthetic dataset of larger tables, we also observe counting representations for the column and row index that resemble the character counting representations. Specifically, we see ringing in the pairwise probe cosine similarities and the characteristic “baseball seam” in the PCA basis.
