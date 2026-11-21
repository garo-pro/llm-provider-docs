# How I stay in the flow with Ultrafast

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

I kind of miss tweaking CSS in DevTools. Trying different colors, font sizes, or animation curves until things felt right. I didn't necessarily know what I wanted when I started. I was figuring it out as I went. I was in the zone.

These days I fire off a lot of tasks to Codex and walk away from my computer. But especially when building a new UI, I often want to iterate towards a solution instead. Ultrafast has given me a lot of that DevTools feeling back.

Earlier this week, we launched GPT-6.2 Astra and Sol with [Ultrafast](https://learn.chatgpt.com/docs/agent-configuration/speed), offering up to 4.5× faster token generation in Codex. Sol Ultrafast is now available on the $200 Pro plan, and we're running an [API promotion](https://developers.openai.com/api/docs/pricing) at 3× Standard pricing instead of 6×.

At this point, I can't imagine working without it. So here are a couple of ways I've found Ultrafast to be magical beyond moments where every minute counts, like debugging an incident, and when the extra speed might be less useful.

## Getting in the zone

For UI work, I’ll usually start with a rough direction and get something on screen as quickly as possible.

If I don’t know what I want the whole experience to look like, waiting for a long task and burning a bunch of extra tokens on higher reasoning can actually be counterproductive. I’m basically hoping to hit the jackpot and have the model build exactly what I had in mind, even though I haven’t fully figured that out myself. I’d rather sit there, try a few things, see the changes, and give feedback as I go. I can react to what Codex is building instead of trying to proactively dictate every detail up front. And because the individual changes often only take a couple of seconds, I can stay in the zone.

![Codex updates a shopping bag in the in-app browser as I annotate the free-shipping indicator and ask to move it next to the title.](/images/blog/how-i-stay-in-the-flow-with-ultrafast/annotation-and-live-iteration.webp)

There’s a couple of things I’ve learned that make Ultrafast feel even more magical:

- **Use a dev server with hot module reloading.** I want to see changes as soon as Codex makes them instead of waiting for a full build.

- **Keep the site open in the in-app browser.** Having the preview next to the chat lets me see changes while I’m prompting.

- **Give short, individual tasks.** I’ll often send one small piece of feedback at a time and jump between different parts of the site as I notice things.

- **Use annotations.** It’s much easier to point at the exact thing I want to change than to describe where it is.

- **Try dictation or voice mode.** I can quickly queue up several small tasks and let Codex work through them while I keep looking at the result.

- **Start with Light/Low reasoning.** Most of these instructions are fairly easy. For something more nuanced, like designing an animation, I might switch to High and then go back to Low.

- **Keep verification out of the quick iteration loop when it’s unnecessary.** For small visual edits, I tell Codex to focus on the code changes while I check the result in the browser. Once I’m happy, I ask it to write the relevant tests and run the additional checks.

**Bonus tip:** If you have a Codex Micro, using it to trigger dictation with Ultrafast is a great way to quickly queue up a bunch of feedback as you scroll through a page to review it. Alternatively, set up a keyboard shortcut for dictation so you can do the same thing.

![Dictated requests queue in Codex beside a shoe product page: show the color name, cross out unavailable sizes, and move the size guide up.](/images/blog/how-i-stay-in-the-flow-with-ultrafast/dictation-and-queue.webp)

## Using Ultrafast for documents and computer use

I use the same approach for slides and documents. Rather than asking Codex to design a full deck, I’ll often go one slide at a time, give feedback, see the changes, and iterate from there. With documents, I’ll give feedback on individual passages as I’m reading and work through the edits that way.

More generally, I’ve found it really useful for tasks that rely mostly on the intelligence of the model and don’t need a lot of additional validation between changes. That could be tweaking slides, editing documents, filling out spreadsheets, or even some computer use tasks where the individual actions are quick.

## When Ultrafast is less useful

Faster token generation doesn’t mean the entire task finishes 4.5× faster. If most of a turn is spent waiting for a build, test suite, compiler, or other tool call, the overall speedup will be smaller. The compiler still takes as long as it takes.

I asked GPT-6.2 Sol at Medium reasoning to make an Android workout timer keep the screen awake while it was running. Each bar shows the average of three runs, including the one-line edit, an incremental build, and all 18 tests.

![Average Android timer task duration: 16.7 seconds on Standard, 16.0 on Fast, and 15.7 on Ultrafast. Tests take about 13 seconds in all three modes.](/images/blog/how-i-stay-in-the-flow-with-ultrafast/android-timer-speed-modes.webp)

Image generation is another good example. If I’m asking Codex to generate an image as part of a task, I might spend most of the time waiting for the image to be generated. Ultrafast can speed up the steps around it, but it won’t make the image itself generate faster.

For a well-defined task that needs a lot of experimentation and validation from the model, the extra speed might also be less valuable. Especially if I’m comfortable walking away and letting Codex run it overnight.

## Where I’d try it in the API

The same principle applies to the API: look at how much time someone spends waiting for the model versus waiting for tools.

A couple of use cases where Ultrafast can be helpful:

- **Document summaries and Q&A.** If a document is already in context, the model can generate a summary or answer a follow-up without a lot of additional tool calls.

- **Fast tool calls and computer use.** Form filling is an interesting example. If your app supports a code mode, the model could write a script to fill several fields instead of going back and forth for every action.

- **Voice experiences.** I’d try Ultrafast as a backend model alongside GPT-Live-1 when a quick turnaround matters.

If you already have an API workflow, try [setting `service_tier` to `ultrafast`](https://developers.openai.com/api/docs/guides/ultrafast-mode) and measure the time to the result.

![A Responses API request sets service_tier to ultrafast for GPT-6.2 Sol to answer a question about returning shoes worn outdoors.](/images/blog/how-i-stay-in-the-flow-with-ultrafast/responses-api-ultrafast.webp)

For GPT-6.2 Astra and Sol, the [promotional rate](https://developers.openai.com/api/docs/pricing) is 3× Standard through February 28, 2027, instead of the usual 6× starting March 1.

## Staying in the flow

For me, the biggest change is that I can get back into the zone when I’m working towards something I haven’t fully figured out yet. Whether I’m designing a UI, tweaking a slide, or editing a document, I can try things, see what feels right, and keep going without losing my train of thought. That’s the feeling I used to get experimenting in DevTools, and it’s been really fun getting it back with Codex. If you’re trying Ultrafast, start with Light or Low reasoning, keep your feedback loops short, and see where you end up.