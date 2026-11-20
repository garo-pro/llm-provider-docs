# Prompting GPT-Live for languages and regional accents

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

Even if a voice agent speaks Spanish, it can still sound off for the audience you're building for. It might use an accent from a different region, choose wording that feels too formal, or read a price in a way a local speaker wouldn't expect.

While the default options for GPT-Live can go a long way, there are additional considerations when building voice applications for audiences across languages and regions. Let’s look at voice selection and session instructions to get better results, using regional varieties such as Indian English, Spanish from Spain, and Emirati Arabic.

## Start with the right voice

First, check GPT-Live’s [built-in voices](/api/docs/guides/live-conversations#voice-options). Some may already have a language or regional influence close to what you need: for example, Bossa and Tempo have a Brazilian Portuguese influence, while Nira is a Hindi voice. Test them with your own conversations before designing something new, as the built-in voices have high naturalness out of the box.



> Illustration: Table of additional GPT-Live voices grouped by language, regional influence, and presentation. The complete catalog is in the linked voice options guide.



Use lowercase API names when selecting a built-in voice (for example, `nira`). The [voice options guide](/api/docs/guides/live-conversations#voice-options) has the complete current catalog.

If you need a more specific regional sound, you can use [prompt-to-voice](/api/docs/guides/custom-voices#create-a-voice-from-a-description) to generate a synthetic voice from a text description. Describe the region as precisely as you can, along with the pitch, pace, and vocal character you want. When generating a voice, you can use the `script_hint` field to add representative speech in the target language, and for a bilingual voice, include both languages in the sample text and describe the desired sound in each. Use original adult voices, and don’t impersonate real people. See the [responsible-use guidance](/api/docs/guides/custom-voices#responsible-use).

Eligible customers can also create a [custom voice from a recording](/api/docs/guides/custom-voices#create-a-voice-from-a-recording) of a consenting speaker. Check the [supported models and accents](/api/docs/guides/custom-voices#select-the-voice-at-session-creation) before choosing a recording-based voice. Before uploading, obtain the speaker’s express consent and secure the rights needed to use the recordings and the resulting voice in your application. The consent recording and reference sample must come from the same speaker. For any generated voice, make clear to listeners that they are hearing AI-generated speech.

Voice creation is ineligible for Zero Data Retention. See the [data controls guide](/api/docs/guides/your-data) for the settings that apply to GPT-Live sessions.

### Example: Hindi-influenced English with Nira

For an English-speaking agent serving users in India, set `session.audio.output.voice` to `"nira"` when creating the session, then add this prompt to the session instructions:

```text
You are in a natural, clean English voice conversation. Speak fluent English in your natural Hindi-influenced Indian English accent. Keep that familiar rhythm throughout.
```

### Example: Spanish from central and northern Spain

Suppose you're building a shopping assistant for customers in Spain. Rather than asking for a generic Spanish accent, open the [voice Playground](https://platform.openai.com/audio/voices/edit) and try this description in its prompt-to-voice workflow:

```text
An original adult feminine voice, clear and lightly bright, with a medium pitch and gentle warmth. Speak Spanish with a natural standard Peninsular Castilian accent of central/northern Spain, with native Spanish vowels, consonants, stress, and sentence melody. Naturally distinguish s from z and c before e or i, without exaggeration. Use a moderately lively conversational pace, natural pauses, and varied intonation.
```

Pair it with a Spanish `script_hint` that gives the voice some ordinary connected speech to work with:

```text
Hola, ¿qué tal? Estoy aquí para ayudarte. Podemos revisar las opciones con calma y elegir la que mejor encaje contigo. ¿Prefieres empezar por los colores o por las tallas? Esta chaqueta azul combina bien con unos zapatos cómodos. Acércate a la ventana para ver el color con luz natural.
```



![Replica of the voice Playground editor with Write a prompt selected, the Spanish description in Voice prompt, and Spanish speech in Script hint.](</images/blog/prompting-multilingual-voice-agents/playground-replica.webp>)



This targets a particular variety of Spanish; the distinction described in the prompt isn't universal across Spain or the Spanish-speaking world. With this prompt, you can generate a few candidates, keep the saved voice you prefer, and test it on a fresh sentence that wasn't in the creation script until you’re satisfied.

### Example: Emirati Arabic from Dubai

Arabic is another case where naming the language alone leaves a lot unspecified. For a conversational Dubai voice, an Arabic voice-design description might be:



```text
صوت رجالي خيالي أصلي لرجل بالغ، بلهجة أهل دبي الإماراتية الحضرية المعاصرة. نطق الحروف وحركة الكلام وإيقاعه محليّان ومتناسقان، وتبقى الكلمات واضحة داخل الكلام اليومي المتصل. طبقة متوسطة تميل للانخفاض، ورنين دافي وصافي. يتكلم على مهله، والجملة تمشي كفكرة وحدة بنبر متنوع حسب المعنى.
```



The description asks for an original adult masculine voice with a contemporary urban Dubai accent, warm resonance, a slightly lower pitch, and connected, relaxed speech. Use a conversational Arabic `script_hint` as well:



```text
هيه، شفت رسالتك. أنا هني صوب الحديقة، لا، أقصد صوب المواقف عند الباب الثاني. زين؟ المكان أهدى من اللي كنا فيه أمس. خذ راحتك، أنا توني واصل وما عندي شي مستعيل عليه. تبانا نتمشى شوي ولا نقعد؟ عادي، اللي يريحك.
```



In general, keep the voice design focused on sound. Managing details like whether the agent should answer an Egyptian caller in Emirati Arabic, switch into English, or confirm an order belongs in the session instructions, which we’ll cover later on.

## Test the voice on what your application will actually say

But having a solid voice is only the starting point. Once you have a voice, you can build a small listening set with fresh sentences, realistic conversations, and the words or values your application handles most often to stress-test your voice.

A few edge cases are particularly useful to test:

- **Pronunciation in context:** Include names, place names, product names, and specialized vocabulary. For Japanese, for example, give the agent a reading such as `承る（うけたまわる）` and test it in a full support sentence. Or if your agent is pronouncing Lakshmi with a “my” ending instead of the intended “mee,” you can include guidance in the session instructions: `लक्ष्मी — अंत में ‘क्ष्मी’, ‘क्ष्माई’ नहीं`. For tonal languages, listen to important words in several sentence contexts.

- **Regional number conventions:** The Indian system of numeration uses lakh and crore. One lakh is `1,00,000` (100,000), and one crore is `1,00,00,000` (10,000,000). For an English-speaking agent serving India, test whether `₹1,25,000` is spoken as “one lakh twenty-five thousand rupees” when that's the intended convention.

- **Identifiers and corrections:** Have a caller spell a fictional booking code, pause, and correct one character. Check that the agent waits for the correction and reads back the final code accurately. Test email addresses and other alphanumeric inputs your product accepts.

- **Mixed-language speech:** Try an Arabic sentence containing an English product name and a price, then switch languages for the next turn. Check the spoken amount, currency, and product name separately from the accent.

For dates, amounts, and identifiers, validate what your application actually sends to the backend as well as what the caller hears. In many regions, `04/05` could mean two different dates, so it may be worth asking for clarification and reading the month back by name.

## Prompt the agent for the conversation

Once you've chosen a voice, select it when creating the GPT-Live session. For a saved generated voice, set `session.audio.output.voice.id` to the returned voice ID. Put the agent's role, language policy, delivery, and conversation behavior in `instructions`, and reinforce the original voice description there.

For a single-language application, write the instructions and examples in the language you want the agent to speak. Here's a starting point for the Spanish shopping assistant:

```text
Eres un asistente de compras. Habla en español de España, con acento castellano del centro y norte de la península, un ritmo conversacional y una entonación natural. Usa un tono cercano y respetuoso, sin sonar como un anuncio.

Responde en español salvo que el cliente pida cambiar de idioma. Haz una pregunta cada vez. Si un nombre o una talla no están claros, pide que los repita. No confirmes precios, disponibilidad ni cambios en un pedido hasta que el sistema los haya verificado.
```

For a multilingual application, decide what should trigger a language switch and whether the underlying regional speaking style should remain fixed. Those are separate choices. For an Emirati Arabic and English assistant, first extend the earlier voice description with English delivery guidance, such as “In English, speak naturally with a light Emirati influence while keeping the same warmth and lower pitch.” Include representative English speech alongside the Arabic in `script_hint`, such as “I can check the available sizes for you,” before generating the bilingual voice. Then use a session policy like this:

```text
Answer in the language of the caller's latest clear utterance. Switch between Arabic and English when the caller clearly switches. A short “OK,” number, name, or English product name inside an Arabic sentence does not by itself change the conversation language.

When speaking Arabic, use conversational urban Emirati Arabic from Dubai, even when the caller speaks another Arabic variety. Prefer natural local wording where appropriate: شو for “what,” الحين for “now,” وين for “where,” and مب for negation. Keep the dialect coherent without exaggeration. In English, keep the same speaker identity.

Use these as phrasing examples, not fixed answers:
- Clarification: «تقصد العلبة الصغيرة ولا الكبيرة؟ وكم وحدة تبا؟»
- Unclear name: «الاسم مب واضح عندي، ممكن تعيده أو تتهجّاه لي؟»

Say amounts in the current conversation language and preserve their exact value and currency. Only confirm a lookup, price, or completed action when the backend supports that statement.
```

This example deliberately keeps the assistant's Arabic dialect fixed while allowing English language switches. If your product should accommodate a caller's requested dialect instead, write and test that policy explicitly. While voice descriptions and session prompts can steer an accent, they don't guarantee that every reply will retain it, so test switches in both directions, including after a session reconnects.

### Give examples of the register you want

Language and accent also leave the level of formality open. In Hindi, a courteous support agent might use `आप` and respectful verb forms. You can make that preference concrete in the session prompt:

```text
ग्राहक से "आप" कहकर बात करें। सम्मानजनक भाषा और क्रिया रूपों का उपयोग करें। "हाँ, बताओ" की जगह "जी, बताइए" कहें। "तुम कैसे हो?" की जगह "आप कैसे हैं?" कहें।
```

The examples distinguish a casual “yes, tell me” from a more courteous equivalent. That gives the model a clearer target than simply asking it to sound professional. Use phrases a local support representative would actually say, and test them in context rather than translating an English script word for word.

For more on pronunciation cues, personality, greetings, and interruption behavior, see the [GPT-Live prompting guide](/api/docs/guides/live-prompting). The [voice agent evaluation guide](/api/docs/guides/voice-agents#evaluate-your-voice-agent) covers how to turn your listening cases into a repeatable evaluation as you change voices, prompts, or models.