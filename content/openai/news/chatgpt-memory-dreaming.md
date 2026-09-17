Title: Dreaming: Better memory for a more helpful ChatGPT

URL Source: https://openai.com/index/chatgpt-memory-dreaming

Markdown Content:
Today we’re beginning to roll out a more capable and scalable system for synthesizing memory, developed to tackle the staleness, correctness, and scalability challenges that we observe when memory is applied to the hundreds of millions of users and multi-year time horizons in ChatGPT.

Memory is what helps ChatGPT learn your preferences, projects, and constraints, allowing future conversations to start from shared context rather than from scratch.

Over the last two years, memory has grown into a critical part of the ChatGPT experience, helping ChatGPT better understand your context so it can help you accomplish meaningful goals over time. This is central to making ChatGPT more useful: knowing you, helping you, and doing more for you.

This update is available to Plus and Pro users in the US today, and will roll out to additional countries and Free and Go users over the coming weeks.

Memory first launched in April 2024 (also known as saved memories). The feature let you ask ChatGPT to remember information and carry it forward into future chats.

![Saved memories modal showing a searchable list of personal details, including work, interests, travel, and response preferences.](https://images.ctfassets.net/kftzwdyauwt9/6kVwo3uYRhe81W7gx1uGoR/c7c242fd100e910d28b451eada638b89/Saved_memories.png?w=3840&q=90&fm=webp)

Saved memories were only written during the conversation and relied on strong cues to decide when to trigger memory, such as an instruction to "remember I’m traveling to Singapore in July." In practice, interacting with this system could feel like talking to someone who took a few notes, but still forgot everything that wasn’t written down. Saved memories also tend to go stale over time and eventually become incorrect or irrelevant.

In April 2025, we updated ChatGPT’s memory by giving the model the ability to reference chat context outside of the saved memories list; this was done by introducing the first version of **dreaming**—a method for ChatGPT to *automatically* curate memories in the background by referencing chat history.

![Memory settings page in ChatGPT showing options to reference chat history, use saved memories, manage saved memories, and enable Pulse memory suggestions.](https://images.ctfassets.net/kftzwdyauwt9/nSkMroPG2SPQCrcIUh2l4/bf05aa9cfb0287cfa7f187f9d84674dd/Memory_settings__1_.png?w=3840&q=90&fm=webp)

In contrast to saved memories, dreaming leverages a background process that allows ChatGPT to learn from many conversations and synthesize ChatGPT’s memory state in order to always provide the freshest, most relevant context to your conversations. Dreaming also makes it easier for memory to include context that occurs naturally in conversation, without relying on explicit requests to remember something.

Over the last year, dreaming *supplemented* saved memories to create a step-function improvement in ChatGPT's ability to personalize responses and offset the staleness of saved memories. However, it historically was never sufficient as a standalone memory system.

Today, we are launching a significantly more capable and compute-efficient memory architecture built on top of dreaming.

The memories synthesized by dreaming are reviewable through a summary of them made visible in the memory summary page. From the memory summary, you can quickly glean the highlights of what ChatGPT knows about you, add or update information about yourself, and provide instructions on what topics ChatGPT should bring up and when. If you want to drill down into a particular area to learn more, just chat with the model.

![Memory summary modal showing a personalized overview of a user’s work, hobbies, travel interests, and community involvement, with options to correct or dismiss specific details.](https://images.ctfassets.net/kftzwdyauwt9/7C8sUaE2XdajvQ8ckphLG5/326a719fbdcfbc3a85e8990155d5aaed/Memory_summary.png?w=3840&q=90&fm=webp)

When we think about what "good memory" looks like in ChatGPT, a few things come to mind:

1. **Carry forward useful context:** You tell ChatGPT something once, and it remembers that information in your subsequent chats.
2. **Follow preferences and constraints:** If you describe a preference (e.g., you’re vegetarian), then ChatGPT should take actions that are consistent with that preference going forward.
3. **Stay current over time:** Memory should account for the passage of time. Imagine "The user is planning their birthday party for next Saturday"; eventually, Sunday arrives.

We can evaluate how ChatGPT Plus and Pro memory has improved over time with respect to each of the three memory objectives above. We do this for each of:

1. **2024** : Saved memories
2. **2025** : Saved memories + Dreaming V0
3. **2026** : Dreaming V3

When you start a new chat with ChatGPT, you don’t have to introduce yourself from scratch. ChatGPT can save you time and *build on prior context*, especially for complex, long-running projects.

For example, imagine you’re using ChatGPT to shop for new camera gear that’s compatible with your current camera. If you’ve discussed your camera setup with ChatGPT in the past, you can ask for products that are compatible with "my photography setup" and get tailored recommendations that meet your needs.

## Without memory

*The model produces a generic response that leaves the user to do complicated compatibility checks on their own.*

## With memory

*The model remembers the user’s camera setup and recommends a compatible product.*

We can construct an eval from examples that resemble this where the model is asked to respond to a prompt that requires it to recall factual information about the user. The model is then rewarded if it responds in a way that correctly uses the relevant context. In this evaluation, the new dreaming-based system improves the model's ability to recall relevant facts.

Memory also helps ChatGPT respond in ways that better match your preferences and constraints.

Imagine that you’re planning a trip to Singapore. Two months before your trip, you ask ChatGPT to help with an itinerary. ChatGPT already knows from past travel planning that you enjoy wildlife photography, prefer hotels with strong AC, and enjoy a quiet dinner over a crowded bar.

## Without memory

*The model produces a generic response that is more touristy, doesn’t help with hotel booking, and largely ignores the user's interests.*

## With memory

*The model produces a response that is personalized to the user’s interests in wildlife photography, quiet dinners, and their priorities when booking a hotel.*

Preferences can take several forms:

1. Instructions for how ChatGPT should respond ("don’t bring up Stan again").
2. Your personal preferences or constraints ("I’m vegetarian")
3. Implicit preferences that shape what’s relevant to you ("I live near San Francisco" → local options should be tailored to this area)

In developing the new memory system, we improved ChatGPT’s ability to apply relevant preferences from past conversations. Following the "I’m vegetarian" example above, we can evaluate whether the model correctly leverages memory to produce vegetarian-friendly dining options when a vegetarian user asks for meal prep suggestions.

*Time doesn’t stop when your chat ends.*

Traditional memory systems can become stale. For example, you tell ChatGPT "I’m in Singapore and need a dinner recommendation for tonight." Then, time passes, your trip ends, and you wonder why ChatGPT still thinks you’re in Singapore.

With dreaming, memories are automatically updated as time passes, allowing ChatGPT to revise its memory from "You’re going to Singapore in July" to "You went to Singapore in July 2026" when the trip ends. Then, when you’re back home, ChatGPT can again provide recommendations that are tailored to your home location and time zone.

## Stale memory

*The model thinks the user is still in Singapore.*

## With memory

*The model provides responses that are relevant to the user’s home location.*

In our memory evaluations, we measure whether ChatGPT can correctly respond to prompts where the passage of time materially affects the correct answer or recommendation. Dreaming provides a substantial lift in this area:

*At OpenAI, our mission is to ensure that artificial general intelligence benefits all of humanity.*

While dreaming-based memory has been available to Plus and Pro users for some time, we are only now able to offer Free users a version that meets our quality bar and is practical to serve at scale. Recent improvements reduced the compute required to serve dreaming to Free users by approximately 5x, making it possible to begin rolling out dreaming to Free users over the coming weeks and to increase memory capacity for Plus and Pro users.

Looking ahead, dreaming now provides us with a shared memory foundation for all users. This update represents our most capable memory system yet, and we’ll continue improving it.

To learn more about this release and memory user controls, visit our [__Memory FAQ__(opens in a new window)](https://help.openai.com/en/articles/8590148-memory-faq).

## Keep reading

[View all](https://openai.com/news/)
