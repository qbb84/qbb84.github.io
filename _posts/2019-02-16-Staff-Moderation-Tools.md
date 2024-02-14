---
layout: post
author: qbb84
tags: [Minecraft, Staff, Moderation Tools, Utilities]
---

I was contacted to help develop for an acquaintances faction server. I was originally contacted to create a more advanced set of specialized tools for the server, and more specifically to help aid the staff with Moderation.

Below will be my favorite projects I was requested to make for the staff team.

## Freezing Potential Cheaters

I developed a plugin which allowed staff members to freeze potential cheaters. When you froze a player, you'd be prompted with previous history, notes left behind from previous interactions, recent anti-cheat flags, and more.

This was mainly used to assess closest cheaters as a last resort.

## Curated Responses

I developed a curated response system for staff members, streamlining the process of answering queries efficiently. This innovative solution leverages Language Model APIs (LLMs) to generate concise and contextually relevant responses. An LLM, or Language Model, is a type of artificial intelligence that excels in understanding and generating human-like text. Specifically, GPT-2, one of the models in the GPT series, was utilized for this purpose.

With Spigot, I integrated the GPT-2 model to enhance communication capabilities. This allowed for dynamic and accurate responses in the Minecraft chat, enriching the overall user experience, and efficiency of staff members.

Implementation was particularly nuanced, and I had to learn a lot. Here's how simple generating text became based on the GPT-2 Python library:

```java
GPT2.generateText(100, getQuestion());
```
