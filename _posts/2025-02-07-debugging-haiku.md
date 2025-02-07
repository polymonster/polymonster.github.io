---
title: 'A Haiku About Debugging and the Perception of Productivity'
date: 2025-02-07 00:00:00
---

Often a small fix<br>
Requires thorough debugging<br>
Its trace left unseen<br>

I wrote an almost-haiku in a Slack message to a coworker as we approached the end of the sprint. It came after spending the better part of two days debugging a difficult problem, which ultimately led to the addition of just two characters to a C++ source file to fix the issue. Along the way, I actually wrote a significant amount of ephemeral code across a data pipeline executable, a graphics runtime executable, and shader code.
Earlier in my career, I struggled with deleting this kind of transient code. I often felt it might be worth keeping, just in case a similar issue arose in the future. I didn’t want my efforts to go to waste. But over time, I’ve learned to let it go. Debugging isn’t just about the code that remains; it’s also about the effort invested in understanding the problem. It includes all the temporary print statements, UI widgets, debug primitives, and countless other tools hastily written and discarded.

Then there’s the time spent in the debugger; stepping through execution, analysing hex values, copying and pasting into notepads, diffing outputs, crafting heroic watch expressions, or tracing obscure memory aliasing issues. All of this work requires experience and patience.
I was fortunate to learn from engineers who were magicians at hardcore debugging, and they passed their wisdom on to me. In turn, I passed it down to the next generation. But still, a part of me feels the need to justify the work I’ve done, because some things are difficult to measure.
When you finally fix a one-liner, it seems obvious in hindsight. It’s frustrating to think you didn’t find the solution sooner. Sometimes you were close, only for another clue to send you down the wrong tangent. Other times, you feel like a badass, pulling off some low-level trickery, only to realise it had nothing to do with the actual problem, and now it just feels like time wasted, showing off to yourself.
Among respected colleagues and peers, we can talk about these endeavours. We learn, laugh, and congratulate one another. That’s never been an issue because we understand what it’s like. But often, there are people outside our world who don’t.

They want time logged. They want burn-down charts. They want to know why something took so long. They ask, “Why did you say this was a small t-shirt size?” (or whatever nonsense they think helps quantify complexity).

This isn’t meant as a dig at those people — they’re often just asking a question, not accusing anyone of wasting time. But for me, it triggers something. It makes me overthink, constantly trying to justify the effort. I suspect this comes from a deeply ingrained attitude toward work, conditioned by society:

That work means sitting at a desk from 9 to 5. That you must be physically present to be productive. That you must return to the office, not because it’s better, but because we don’t like the idea of you doing your washing at home during the workday. That, deep down, we are still mill workers.

Ironically, I don’t even have a return-to-office mandate. I’m not required to go in at all. But I still feel it. It’s been subliminally drilled into me since childhood: showing up is the job. It doesn’t account for the times I’ve solved a bug in my sleep. Or that time on the tube, when I mentally fixed an atomic race condition in a shader, then got to my desk and wrote a few lines of code to solve it, before slowly shifting into the headspace of the next challenge.

This haiku is my reminder - maybe even the start of reconditioning myself. To anyone else reading this: the unseen work, while ephemeral, is often the most important.
