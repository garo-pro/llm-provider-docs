Title: Jukebox

URL Source: https://openai.com/index/jukebox

Markdown Content:
![Jukebox](https://images.ctfassets.net/kftzwdyauwt9/1f223ed0-8e6b-4add-283f1d99d604/97cb057c8dee1a92cb2c85623620fbf5/image-4.webp?w=3840&q=90&fm=webp)

Illustration: Ben Barry

We’re introducing Jukebox, a neural net that generates music, including rudimentary singing, as raw audio in a variety of genres and artist styles. We’re releasing the model weights and code, along with a tool to explore the generated samples.

Provided with genre, artist, and lyrics as input, Jukebox outputs a new music sample produced from scratch. Below, we show some of our favorite samples.

Automatic music generation dates back to more than half a century.<sup>, , ,</sup>  A prominent approach is to generate music symbolically in the form of a piano roll, which specifies the timing, pitch, velocity, and instrument of each note to be played. This has led to impressive results like producing Bach chorals,<sup>,</sup>  polyphonic music with multiple instruments,<sup>, ,</sup>  as well as minute long musical pieces.<sup>, ,</sup> 

But symbolic generators have limitations—they cannot capture human voices or many of the more subtle timbres, dynamics, and expressivity that are essential to music. A different approach <sup>is to model music directly as raw audio.</sup><sup>, , ,</sup>  Generating music at the audio level is challenging since the sequences are very long. <sup>A typical 4-minute song at CD quality (44 kHz, 16-bit) has over 10 million timesteps. For comparison, GPT‑2 had 1,000 timesteps and</sup> [OpenAI Five](https://openai.com/five/) took tens of thousands of timesteps per game. Thus, to learn the high level semantics of music, a model would have to deal with extremely long-range dependencies.

One way of addressing the long input problem is to use an autoencoder that compresses raw audio to a lower-dimensional space by discarding some of the perceptually irrelevant bits of information. We can then train a model to generate audio in this compressed space, and upsample back to the raw audio space.<sup>,</sup> 

We chose to work on music because we want to continue to push the boundaries of generative models. Our previous work on [MuseNet](https://openai.com/index/musenet/) explored synthesizing music based on large amounts of MIDI data. Now in raw audio, our models must learn to tackle high diversity as well as very long range structure, and the raw audio domain is particularly unforgiving of errors in short, medium, or long term timing.

Jukebox’s autoencoder model compresses audio to a discrete space, using a quantization-based approach called VQ-VAE. <sup>Hierarchical VQ-VAEs</sup> <sup>can generate short instrumental pieces from a few sets of instruments, however they suffer from hierarchy collapse due to use of successive encoders coupled with autoregressive decoders. A simplified variant called VQ-VAE-2</sup> <sup>avoids these issues by using feedforward encoders and decoders only, and they show impressive results at generating high-fidelity images.</sup>

We draw inspiration from VQ-VAE-2 and apply their approach to music. We modify their architecture as follows:

- To alleviate codebook collapse common to VQ-VAE models, we use random restarts where we randomly reset a codebook vector to one of the encoded hidden states whenever its usage falls below a threshold.
- To maximize the use of the upper levels, we use separate decoders and independently reconstruct the input from the codes of each level.
- To allow the model to reconstruct higher frequencies easily, we add a spectral loss <sup>,</sup> that penalizes the norm of the difference of input and reconstructed spectrograms.

We use three levels in our VQ-VAE, shown below, which compress the 44kHz raw audio by 8x, 32x, and 128x, respectively, with a codebook size of 2048 for each level. This downsampling loses much of the audio detail, and sounds noticeably noisy as we go further down the levels. However, it retains essential information about the pitch, timbre, and volume of the audio.

Next, we train the prior models whose goal is to learn the distribution of music codes encoded by VQ-VAE and to generate music in this compressed discrete space. Like the VQ-VAE, we have three levels of priors: a top-level prior that generates the most compressed codes, and two upsampling priors that generate less compressed codes conditioned on above.

The top-level prior models the long-range structure of music, and samples decoded from this level have lower audio quality but capture high-level semantics like singing and melodies. The middle and bottom upsampling priors add local musical structures like timbre, significantly improving the audio quality.

We train these as autoregressive models using a simplified variant of Sparse Transformers.<sup>,</sup>  Each of these models has 72 layers of factorized self-attention on a context of 8192 codes, which corresponds to approximately 24 seconds, 6 seconds, and 1.5 seconds of raw audio at the top, middle and bottom levels, respectively.

Once all of the priors are trained, we can generate codes from the top level, upsample them using the upsamplers, and decode them back to the raw audio space using the VQ-VAE decoder to sample novel songs.

To train this model, we crawled the web to curate a new dataset of 1.2 million songs (600,000 of which are in English), paired with the corresponding lyrics and metadata from [LyricWiki(opens in a new window)](https://lyrics.fandom.com/wiki/LyricWiki). The metadata includes artist, album genre, and year of the songs, along with common moods or playlist keywords associated with each song. We train on 32-bit, 44.1 kHz raw audio, and perform data augmentation by randomly downmixing the right and left channels to produce mono audio.

The top-level transformer is trained on the task of predicting compressed audio tokens. We can provide additional information, such as the artist and genre for each song. This has two advantages: first, it reduces the entropy of the audio prediction, so the model is able to achieve better quality in any particular style; second, at generation time, we are able to steer the model to generate in a style of our choosing.

This t-SNE <sup>below shows how the model learns, in an unsupervised way, to cluster similar artists and genres close together, and also makes some surprising associations like Jennifer Lopez being so close to Dolly Parton!</sup>

In addition to conditioning on artist and genre, we can provide more context at training time by conditioning the model on the lyrics for a song. A significant challenge is the lack of a well-aligned dataset: we only have lyrics at a song level without alignment to the music, and thus for a given chunk of audio we don’t know precisely which portion of the lyrics (if any) appear. We also may have song versions that don’t match the lyric versions, as might occur if a given song is performed by several different artists in slightly different ways. Additionally, singers frequently repeat phrases, or otherwise vary the lyrics, in ways that are not always captured in the written lyrics.

To match audio portions to their corresponding lyrics, we begin with a simple heuristic that aligns the characters of the lyrics to linearly span the duration of each song, and pass a fixed-size window of characters centered around the current segment during training. While this simple strategy of linear alignment worked surprisingly well, we found that it fails for certain genres with fast lyrics, such as hip hop. To address this, we use Spleeter <sup>to extract vocals from each song and run NUS AutoLyricsAlign</sup>[\[](https://openai.com/index/jukebox/#rf33)^reference-33] on the extracted vocals to obtain precise word-level alignments of the lyrics. We chose a large enough window so that the actual lyrics have a high probability of being inside the window.

To attend to the lyrics, we add an encoder to produce a representation for the lyrics, and add attention layers that use queries from the music decoder to attend to keys and values from the lyrics encoder. After training, the model learns a more precise alignment.

**Lyric–music alignment learned by encoder–decoder attention layer**Attention progresses from one lyric token to the next as the music progresses, with a few moments of uncertainty.

While Jukebox represents a step forward in musical quality, coherence, length of audio sample, and ability to condition on artist, genre, and lyrics, there is a significant gap between these generations and human-created music.

For example, while the generated songs show local musical coherence, follow traditional chord patterns, and can even feature impressive solos, we do not hear familiar larger musical structures such as choruses that repeat. Our downsampling and upsampling process introduces discernable noise. Improving the VQ-VAE so its codes capture more musical information would help reduce this. Our models are also slow to sample from, because of the autoregressive nature of sampling. It takes approximately 9 hours to fully render one minute of audio through our models, and thus they cannot yet be used in interactive applications. Using techniques<sup>,</sup>  that distill the model into a parallel sampler can significantly speed up the sampling speed. Finally, we currently train on English lyrics and mostly Western music, but in the future we hope to include songs from other languages and parts of the world.

Our audio team is continuing to work on generating audio samples conditioned on different kinds of priming information. In particular, we’ve seen early success conditioning on MIDI files and stem files. Here’s an example of a [raw audio sample(opens in a new window)](https://soundcloud.com/openai_audio/generated-raw) conditioned on [MIDI tokens(opens in a new window)](https://soundcloud.com/openai_audio/midi-input-given-to-model-as-midi-tokens-rendered-here-by-timidity). We hope this will improve the musicality of samples (in the way conditioning on lyrics improved the singing), and this would also be a way of giving musicians more control over the generations. We expect human and model collaborations to be an increasingly exciting creative space. If you’re excited to work on these problems with us, [we’re hiring](https://openai.com/careers/).

As generative modeling across various domains continues to advance, we are also conducting research into issues like [bias(opens in a new window)](https://arxiv.org/abs/1908.09203) and [intellectual property rights(opens in a new window)](https://cdn.openai.com/policy-submissions/OpenAI%20Comments%20on%20Intellectual%20Property%20Protection%20for%20Artificial%20Intelligence%20Innovation.pdf), and are engaging with people who work in the domains where we develop tools. To better understand future implications for the music community, we shared Jukebox with an initial set of 10 musicians from various genres to discuss their feedback on this work. While Jukebox is an interesting research result, these musicians did not find it immediately applicable to their creative process given some of its current [limitations](https://openai.com/index/jukebox/#limitations). We are connecting with the wider creative community as we think generative work across text, images, and audio will continue to improve. If you’re interested in being a creative collaborator to help us build [useful tools](https://openai.com/index/musenet/) or new works of art in these domains, please [let us know(opens in a new window)](https://forms.gle/8npHSMnE5hfSxkkU9)!

*To connect with the corresponding authors, please email* *jukebox@openai.com**.*

### Timeline

- **July 2019**Our first raw audio model, which learns to recreate instruments like Piano and Violin. We try a dataset of rock and pop songs, and surprisingly it works.
- **September 2019**We collect a larger and more diverse dataset of songs, with labels for genres and artists. Model picks up artist and genre styles more consistently with diversity, and at convergence can also produce full-length songs with long-range coherence.
- **January 2020**We scale our VQ-VAE from 22 to 44kHz to achieve higher quality audio. We also scale top-level prior from 1B to 5B to capture the increased information. We see better musical quality, clear singing, and long-range coherence. We also make novel completions of real songs.
- **January 2020**We start training models conditioned on lyrics to incorporate further conditioning information. We only have unaligned lyrics, so model has to learn alignment and pronunciation, as well as singing.

## Footnotes

## References

## Equal contributos

## Contributos

## Acknowledgments

Thank you to the following for their feedback on this work and contributions to this release: Jack Clark, Gretchen Krueger, Miles Brundage, Jeff Clune, Jakub Pachocki, Ryan Lowe, Shan Carter, David Luan, Vedant Misra, Daniela Amodei, Greg Brockman, Kelly Sims, Karson Elmgren, Bianca Martin, Rewon Child, Will Guss, Rob Laidlow, Rachel White, Delwin Campbell, Tasso Smith, Matthew Suttor, Konrad Kaczmarek, Scott Petersen, Dakota Stipp, Jena Ezzeddine

## Acknowledgments

Editor: Ashley Pilipiszyn

Design & Development: Justin Jay Wang & Brooke Chan

## Related articles

[View all](https://openai.com/news/)
