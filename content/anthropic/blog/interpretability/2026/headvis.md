Title: HeadVis: An Interactive Tool For Investigating Attention Heads

URL Source: https://transformer-circuits.pub/2026/headvis

Markdown Content:
We introduce HeadVis, an interactive tool for investigating attention heads in large language models. Visualizing how individual computational units activate across the full data distribution has been useful in previous work — for example, [feature vis](https://transformer-circuits.pub/2023/monosemantic-features/index.html#setup-interface) has been an essential tool for investigating residual stream features, MLP features, and MLP neurons. HeadVis is a conceptually similar tool but for attention heads. However, attention heads are harder to visualize than neurons, since they are high rank, compose two different circuits, and activate across a full context window. We present a few case studies using HeadVis to interpret attention heads from Claude Haiku 3.5. We also open-source [HeadVis](https://github.com/anthropics/headvis) code, and present a demo of HeadVis on heads from [Gemma 3](https://transformer-circuits.pub/gemma3/index.html) and a subset of heads from [Haiku 3.5](https://transformer-circuits.pub/haiku/index.html).

Prior work has shown that the behavior of attention heads on a broad data distribution can look quite different than their behavior on specific datasets. For example:

We built HeadVis to easily generate and test hypotheses about what an attention head does, using visualizations of attention patterns and head outputs, quantitative distribution metrics, low-rank component projections, and SAE feature attributions through QK and OV circuits.

In this paper, we:

One theme emerges across both prior work and our own: a head's behavior on the full distribution rarely matches what a narrow task suggests, and the difference isn't always easy to characterize.

We begin by studying induction to get familiar with using HeadVis. Induction heads`[A][B] ... [A] -> [B]`, the head attends from the second `A` back to `B` to predict its recurrence. A only needs to match fuzzily.

The first component of the HeadVis UI is a scatter plot of heads, with both axes and point color configurable to any per-head quantity, like its layer, head index, or one of several computed metrics. Heads that sit at the extremes of a metric are often the ones doing something interpretable, so this provides a simple way to find interesting heads quickly.

We can find induction heads by using a proxy metric called the induction score`[A][B]…[A]->[B]`, an induction head would attend from the second A to the first B. This is the same as the prefix score from 

Below is the attention pattern for the fuzzy induction head on a top-activating dataset example. This is the central UI of HeadVis: attention patterns bucketed by max attention pattern. On first view we noticed cases of standard induction. Hovering over `‘m` (in `I’m`) we see attention to the token after `I’m` earlier in the context. We also noticed that `gleam` has the highest max attention pattern, but there isn’t a previous instance of `gleam` so this can’t be standard induction. Hovering over either token in `gleam` we see attention to the new line after “dream”. Try guessing where tokens attend back to, then hovering over them to check. We've found fuzzy induction heads that match words across languages, across antonyms, across structural positions (the first item of one list matching with the first item of another list), and across kinship terms (aunt matching with uncle).

`t` to instead color tokens by their maximum attention pattern as a key.
To understand what makes this induction fuzzy, we run QK and OV attributions `gleam` back to the newline after `dream`. QK attribution writes the attention score as a sum of (query feature, key feature) interactions. In a copying induction head the top feature interactions would be `current token is X` on the query side with `token after X` on the key side. In this case we see `-ea sound at line end` on the query side interact with `newline after an “ee” sound` on the key side. This is analogous to the copying induction case except instead of matching on a specific token it’s matching on `”ee” sound`. 

OV attribution writes the head's output as a sum of (value feature, output feature) interactions: value features at the attended token, output features at the query token. Here the top OV term is identical to literal induction: `current token is \n` on the value side and `say \n` on the output side. The fuzziness lives entirely in QK; once the head has decided where to attend, it copies just like a literal induction head. Figure 4 lays these four features out as a circuit; the full QK and OV attribution tables are in the HeadVis UI.

QK and OV attributions are a core capability of HeadVis and can be generated in real time for any (query, key) pair; the features come from the same 10M-feature [weakly causal crosscoder](https://transformer-circuits.pub/2024/crosscoders/index.html) (WCC) used in 

This is a typical HeadVis workflow: pick a head that's extreme on some metric, browse its attention patterns across dataset examples, and use QK and OV attributions to understand what's happening. Induction heads are a clean case where the workflow goes smoothly.

When visualizing the attention heads from 

- Years: the tokenizer splits years 1000–1999 into two tokens, and the head attends between them. For example, `1776` becomes `1` + `776` and the head attends from `776` to `1`.

- Multi-token words: the tokenizer splits long words into multiple tokens, and the head frequently attends between them. For example, `interpretability` becomes `interpret` + `ability` and the head attends from `ability` to `interpret`.

- Newlines: the head attends from a newline token to a previous newline token. This was the previously studied behavior — in this case, the head computes the current line's width with help from character-counting heads in the previous layer.

We identified these behaviors by inspecting which tokens attend to which. HeadVis pre-computes a PCA view for every head as a cheap first check on whether token-level distinctions like these also hold in the head's Q, K, and O activations — different-looking tokens can produce similar activations, in which case what looks like three behaviors might be one to the head. We take the (query, key) token pairs with highest attention output norm, and collect the head's Q, K, and O activations at those pairs

For this head, all three PCAs separate the behaviors into distinct clusters in the first few principal components.

`1`, query is a 3-digit number, distance 1), independent of the PCA.
The clean separation across Q, K, and O — together with the token-level patterns — makes us fairly confident this head is polysemantic: a single head implementing several unrelated behaviors. Browsing more dataset examples in HeadVis, the three behaviors account for most of what we see; there aren't many examples we couldn't classify as one of them. A more direct test of polysemanticity would split the head into separate attention heads, each isolating one behavior, and show that together they reproduce the original; we return to this in the [Discussion](https://transformer-circuits.pub#discussion) section.

Knowing a behavior exists isn't the same as fully characterizing the behavior. For the years behavior, QK attributions on individual examples pair a "current token is 1" key feature with a "current token is [the 3-digit suffix]" query feature — consistent with what attention patterns suggested, and consistent with the behavior covering all of 1000–1999. But per-example attributions don't give a rule that predicts the circuit on arbitrary inputs: sweeping the year in a fixed prompt shows the head attends strongly only for 1000–1986, a constraint neither the attention patterns nor the attributions surfaced.

This is one of the two outcomes the introduction described: a head studied for one task turns out to have several unrelated behaviors on the full distribution. It's about as clean an example as we've found — an early-layer head, picked because its behaviors separated cleanly, with three behaviors visible from token patterns, confirmed by PCA, and attributions that mostly make sense. The answer selection head we study next is the other outcome, and it has none of those advantages.

We took a closer look at the answer selection head from 

We start with the same attention pattern view that surfaced the line width head's three behaviors. None of the top-activating examples are multiple choice questions. This isn't surprising, since multiple choice questions are a small percentage of the full dataset, and the model is unlikely to dedicate a full head to it.

The attention patterns give only a vague sense of what the head is doing. We find this is typical of mid-layer heads: token-level patterns are less readable than in early-layer heads. Applying the same PCA view that separated the line width head's behaviors, we get a different result: the Q, K, and O activations form one continuous cloud with no clusters.

In this head, the PCA does not cluster activations, so it's not useful for understanding if this head implements multiple behaviors. Instead, we turn to QK attributions, where our prior work already mapped the multiple choice case: a query feature for "about to say an answer to a multiple choice question" interacted with a key feature for "correct answer," and the OV circuit copied the answer label to the output. Running QK and OV attributions on examples from the full distribution, a pattern emerges. The OV circuit always does the same thing: it copies content stored at the attended token, much like induction. The QK circuit is where the complexity lives: across examples it matches "content the model has marked as relevant" features on the key side, and "about to produce content matching that mark" features on the query side.

These examples look structurally similar, but disjoint sets of features activate on each — for example, the multiple choice query feature never activates with the sentiment key feature. This raises the question of whether the head really implements one behavior, or whether it implements many behaviors that share a common template.

Following [virtual weights](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#global-weights) between features in the head's QK circuit. We project the "correct answer" feature through the head's key weights, and ask which other features land nearby in that projected space. Studying head behavior in the overcomplete feature basis could show us relations between keys and queries that low-rank projections like PCA could not discover.

The features that align most strongly are not specifically about multiple choice, but they are other instances of "this is the relevant content" — these features are near-orthogonal in the residual stream and the head's K projection maps them close together. These features fire on contexts broader than just multiple choice questions. The same holds on the query side.

For contrast, the same analysis on the line width head's years behavior shows the opposite: the features with the highest virtual weights to a years feature are all about years.

Our best guess is that this head is monosemantic, but we aren't fully convinced. The attributions and virtual weights above cover only a handful of examples. They look consistent, but we haven't surveyed the head systematically. A deeper issue is that we don't have a crisp enough hypothesis for this head's behavior to validate it. If we had a text description of what the head does, we could test every dataset example against it and notice (or have a grader model notice) when something doesn't fit. Without one, we're looking at a small sample and observing that everything so far matches the same loose template.

We close the case studies with a mid-layer head we only partially understand. We isolated one mechanism in its QK geometry — enough to predict its behavior on controlled inputs — then traced why that mechanism is hard to see on real data.

In HeadVis, you can view the head's top query and key tokens by the average attention that the head pays them. For this head, both lists are dominated by places, with some people and team names.

Dataset examples show the head attending between places and people, but it was hard to say more than that from attention patterns alone. In the first sequence below, from `Ivory` (in "Ivory Coast") it attends to an unrelated `a` in "we've had `a` real laugh". In the second, the head attends from `Knicks` to `Kidd` but not from `Nets` to `Kidd` — even though Jason Kidd is best known as a Net.

That second example was a hint that the obvious relation — a team attending to its own player — didn't hold. If you set up HeadVis with a server, it lets you type any sequence and see the head's attention on it[Open Source HeadVis](https://transformer-circuits.pub#open-source-headvis).`Madrid` the head attends to `Messi` rather than `Modrić`, even though Modrić played for Real Madrid and not Messi. From `Europe` the head attends to `Zambia`, the one country in the list that isn't European.

To test this systematically we generated 185 sentences of the form "{five cities} are all cities in {country}", where four cities belong to that country and one is foreign. The country token attends more to the foreign city than to any of its own 98% of the time.

We find this relationship explicitly formed in the QK circuit's geometry. For each country, we compute the head's query vector on "The country {X}", subtract the mean across countries, and do the same for city key vectors on "The city {X} is well known". A country's query vector has strongly negative cosine similarity (mean −0.47) with the key vectors of its own cities and near-zero (mean +0.01) with cities elsewhere. The W_Q and W_K projections create this geometry: in the residual stream a country and its own cities have positive cosine similarity (mean +0.17), so the head flips the sign.

The geometry and the 98% result on custom sentences are evidence that same-set suppression is real. But going back to this head's dataset examples in HeadVis with the pattern in mind, it isn't clean. For the polysemantic head earlier, once we knew the three behaviors we could pick them out in example after example. Here, some examples aren't about places at all; some have attention between places all from the same region, with no odd one out; for many we'd have to look harder to say whether they fit. We think what we've isolated is part of what this head does with places — not cleanly separable from the rest the way the polysemantic head's behaviors were, and not a full account of the place examples either.

We returned to the `Ivory`→`a` example mentioned earlier, which had a place in the query position but not a related token in the key position. QK attributions from Ivory to `a` show the top interacting feature pair is a Zambian feature on the key side and an Ivory Coast feature on the query side, contributing positively to the attention score.

`Ivory` to `a`. The top feature pair is an Ivory Coast feature on the query side and a Zambian feature on the key side.
Although the attention edge we're studying is from `Ivory` to a random token, the rest of the sequence seemed similar to our country-and-city results. The sequence is about a trip to Zambia — [chikumbuso](https://www.chikumbuso.com/), mentioned earlier in the sequence, is a Zambian nonprofit — and Ivory Coast is the one mention of a different country. The head attending from that out-of-place country to `a` partially because of a Zambian feature has the same shape: attention between something that fits and something that doesn't. 

To test this we replaced `Ivory Coast` with `Zambia` in the sequence, and attention to `a` dropped from 0.87 to 0.08; the same happened with `Lusaka`, the Zambian capital (0.10). Swapping in the other 51 African countries, attention mostly stays well above that (median 0.27). So as with our narrow prompts, a different country at the query attends and the same country doesn't

The remaining question was how a Zambian feature came to be active at `a`. Within HeadVis you can pick any (query, key) pair on a sequence and rank all heads by their attention pattern on it[Open Source HeadVis](https://transformer-circuits.pub#open-source-headvis).`chikumbuso` was the only Zambia-related token earlier in the sequence, so we looked for a head that attended between `a` and `chikumbuso`, and checked its OV circuit. That head writes exactly the Zambian features our head's key side reads! This is K-composition 

`chikumbuso` and writes the Zambian features this head reads.
This example partially explains why this head's dataset attention patterns aren't easy to read: sometimes what matters at a position is the features active there, not the token. Here the head attends to `a` because of a Zambian feature an earlier head put there; from the tokens alone there's no way to see that.

These four heads span how far the workflow gets you: induction ran cleanly end to end; the line width head's three behaviors were visible from token patterns and confirmed by PCA; the answer selection head needed feature-level analysis to even form a hypothesis; and here, controlled inputs and QK geometry recovered one mechanism we couldn't see from dataset examples. Each step in this section used a different HeadVis view — top-token rankings to see what dominates, custom sequences to test a guess, QK attributions to read a confusing example, head-ranking on a (query, key) pair to trace composition. We open-sourced the tool so others can run these investigations on their own models.

HeadVis is built from three pieces: a frontend for browsing and interacting with heads, an offline script that precomputes per-head metrics and attention patterns over a dataset, and a server that handles real-time queries like QK and OV attributions. We open-source the frontend along with a specification of the interface with the two backends. You can find the repository [here](https://github.com/anthropics/headvis) along with instructions for connecting the frontend to your own model. We suggest pointing Claude at the repository to implement the backend pieces for your setup. A hosted demo on a subset of Haiku 3.5 heads is [here](https://transformer-circuits.pub/haiku/index.html)[here](https://transformer-circuits.pub/gemma3/index.html).

Some features we found useful internally are not included in this open source release:

There are likely many more extensions that would make this tool more useful.

Interactive Attention Head Visualization. 

HeadVis is inspired by these works, but extended them to both understand how a single head functions across the full distribution of data, and develop extensible tools that simplified understanding complex attention behaviors. For instance, it would have been much more difficult to understand the general behavior of the answer selection head, or the composition of the same-set suppression head with other heads, without extending HeadVis to support QK/OV feature attributions and head composition visualizations.

Cataloging Attention Head Behaviors. Many prior works characterize the behaviors of individual attention heads; see 

We view HeadVis as a tool to discover more of these attention behaviors and pathologies. We expect that understanding individual examples of attention head biology will continue to be very insightful and guide future progress on fully decomposing the mechanisms of attention, such as with methods from 

It's useful to frame the open problems in the study of attention by contrast with MLPs. MLP layers are made up of many polysemantic neurons in superposition. For MLPs, we can learn monosemantic, sparse features that faithfully reconstruct much of the layer's computation with transcoders

We don’t know how to do the analogous thing for an attention layer. The goal is spiritually the same: learn monosemantic, sparse "attentional features" that faithfully reconstruct an attention layer. We expect an attentional feature to look roughly like an attention head, perhaps with a modified nonlinearity or a different head dimension, but close enough that a standard head is a reasonable mental model[mechanistic faithfulness](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#limitations-faithfulness) when we choose arbitrary decompositions of attention layers

Four obstacles stand in the way:

We don't yet know what attentional features look like in general, so we study individual heads as a stand-in. Heads are probably harder to interpret than attentional features — a head can be polysemantic or in superposition — but HeadVis gets us far enough to learn from them. Two of the heads from this paper gave us concrete examples of what a decomposition would need to handle: the line width head showed a single head cleanly implementing several unrelated behaviors, and the answer selection head pointed toward a monosemantic feature that may be irreducibly high-rank. Although we don't understand these mechanisms fully, knowing that they even exist is a very useful step towards characterizing the attentional feature they are a part of.

We'd be excited to see work on HeadVis for long contexts. In particular, heads that frequently attend over long distances have been less studied in the literature. Understanding how attention works over long transcripts could be very fruitful for studying advanced model capabilities, such as the role of the system prompt in guiding an Assistant throughout a conversation.

We'd also like to know how often heads use softmax competition for priority logic. A head can implement "attend to `A` if present, otherwise `B`" just by scoring `A`'s keys higher than `B`'s. Is this a fundamental part of the attention mechanism? So far we have only studied individual attention edges (a single key and query token) at a time, so any logic that depends on competition among keys would be invisible to us.

There's more attention biology to find, though it's unclear whether the obstacles below need to come first. Either way, we'd encourage anyone researching attention to spend a few hours in HeadVis on an open model. There's intuition that only comes from looking, and often a few hours is enough to find a head that crisply exemplifies the theoretical problem you want to study.

Our line width head made this obstacle feel approachable. Polysemanticity was known to occur, but our example is unusually simple: three behaviors, separable by PCA, in an early-layer head where attention patterns are readable on their own. We found it without much searching and we expect more heads like it. Such heads are natural test cases for decomposition methods.

We also want to make a theoretical point explicit: an attention head can be polysemantic without superposition. A head's output is high-rank, so it can write unrelated behaviors to distinguishable directions that downstream components can read from this head alone. An MLP neuron cannot do this. Its output is a single direction, so a polysemantic neuron needs other neurons to disambiguate its meanings, and that is superposition by definition. Head polysemanticity therefore comes in two flavors, with and without superposition. We have not determined which our line width head is, but a decomposition method must handle both.

We'd be excited to see a single polysemantic head split into monosemantic units. Take a head like our line width head, find weights for three attention heads that each isolate one behavior, and check that together they replicate the original head's output. The loftier goal is a procedure that does this automatically for any head

We suspect some attentional features are spread across multiple heads, but we have no confirmed example in a real LLM. We'd be excited to see a single example: a behavior that is difficult to interpret when you look at any head individually, but emerges when you view a few heads that collectively attend between tokens

Polysemanticity and superposition are both reasons a single head is the wrong unit of analysis; solving them means finding better units. This obstacle is different: even the right unit may be high-rank, and the field has little experience interpreting high-rank objects.

MLP layers have two properties which make them simpler to study:

(i) They are composed of neurons, and we can construct a transcoder with exactly the same architecture to study them. We believe that the features we learn can be [mechanistically faithful](https://transformer-circuits.pub/2025/attribution-graphs/methods.html#limitations-faithfulness) to the original MLP.[not true](https://transformer-circuits.pub/2025/faithfulness-toy-model/index.html)

(ii) Each transcoder neuron is a feature, and is a rank-1 object, which is simple to interpret and reason about.

For attention, the layer's units aren't rank-1: a head's pre-softmax score is a rank-

We'd be excited to see a high-rank attentional feature worked out in full: find a head that is plausibly monosemantic and not in superposition, and give a description of what it computes that holds up across the data distribution. The answer selection head might be a candidate, but we're nowhere near a complete description of it and don't know whether superposition is involved.

Several methods already attempt the decomposition we're describing. Attention-output SAEs 

One consequence of the rank-1 OV design is that features with higher-rank OV, like succession or copying induction, get split across many learned features, when they might be better understood as single behaviors. These approaches also inherit the high-rank interpretability challenge, since their QK circuits are still high rank. We'd be excited to see these decompositions applied to a small number of early-layer heads that are understandable with HeadVis; that is likely the easiest way to get a qualitative sense of their performance.

We're optimistic about interpreting attention. HeadVis surfaces concrete examples of most obstacles, and the examples aren't hard to find once the tool exists. Now that there's a set of crisp examples to work from, we think the field is positioned to tackle the obstacles directly. That will require new methods, but the examples tell us what those methods need to handle.
