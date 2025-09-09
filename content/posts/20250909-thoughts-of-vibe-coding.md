---
title: "Thoughts of Vibe Coding"
date: 2025-09-09T11:07:59+08:00
draft: false
description: ""
tags: ["Vibe Coding", "GenAI"]
categories: ["Thoughts", "GenDev"]
author: "Shawn Zhang"
showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
searchHidden: false
cover:
    image: "/images/20250909-vibe-coding.jpeg"
    alt: "Vibe Coding methodology illustration"
    caption: "The evolution of coding with AI assistance"
    relative: true
    hidden: false
---

## Has the Human Engineer Become Obsolete? On the Contrary, Their Role is More Critical Than Ever

Can AI coding assistants replace human engineers? I believe the answer is no, at least not in the current technological landscape.

Through my recent work on several customer proof-of-concepts, I've observed that AI coding tools following the [Vibe Coding](https://en.wikipedia.org/wiki/Vibe_coding) methodology excel at rapid prototyping when provided with clear specifications. They perform exceptionally well for straightforward tasks—implementing specific fixes, adjusting styles, or building simple applications like Todo lists or Snake games. For these well-defined, isolated tasks, AI assistants demonstrate remarkable efficiency.

However, when dealing with more complex systems, AI tools can become problematic for two main reasons. First, when the optimal solution or architectural direction isn't immediately clear, the AI tends to pursue a single approach relentlessly (as we say in Chinese, "一条道走到黑"—going down one path to the bitter end), unable to pivot or reconsider alternative strategies. This behavior can consume valuable time through endless response cycles, retries, and iterations, ultimately leading to a frustrating development experience.

The second challenge emerges when systems grow in complexity: AI assistants often lack a global view and fail to tackle issues systematically. Rather than starting small, ensuring each component works independently, and then integrating them methodically, current AI tools tend to address everything simultaneously. This approach leads to a cascade of interconnected problems across multiple components, making debugging exponentially more difficult and time-consuming. The result is often a chaotic development cycle where developers spend more time waiting for AI responses and untangling complex issues than actually making progress.

In these situations, human intervention becomes essential. Engineers must step in to provide clear direction and architectural guidance, even if that insight comes from consulting other AI tools with better contextual understanding. Once a clear path is established, coding assistants can effectively implement solutions based on specific requirements. Often, the most successful approach involves breaking down complex problems into smaller, manageable modules, ensuring each component functions correctly before integration.

## References

- [Vibe Coding - Wikipedia](https://en.wikipedia.org/wiki/Vibe_coding)
- [What is Vibe Coding? - Google Cloud](https://cloud.google.com/discover/what-is-vibe-coding)
