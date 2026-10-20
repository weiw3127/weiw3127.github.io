
---
title: "Keeping Your Dataset Consistent"
description: "What I've learned from years of resolving annotation disagreements and keeping large-scale datasets on track."
pubDatetime: 2026-10-20T00:00:00+11:00
featured: true
draft: false
tags:
  - machine-learning
  - data-annotation
  - data-quality
canonicalURL: "https://substack.com/@weiwstack/p-221229221"
---
Managing annotation projects is exhausting. 

I've spent years working on annotation projects for training commercial applications, including ranking, categorisation, and recommendation. I often found the hardest thing to get right is **consistency**. How can we make sure that our data is providing a clear, consistent signal for our model to learn from, instead of a confused jumble of noise? 

The tricky part of building human data is that humans aren’t always consistent. Same film, same bite, we might rate it differently based on our mood, energy, or even the day of the week. Or sometimes, we are just simply making mistakes. 

**The key is to treat inconsistency as the default.** Instead of expecting everyone to make the same judgments, we should accept that disagreements will happen and build processes to catch and resolve them. And disagreements aren't always a bad thing! Sometimes they're exactly what we need to discover unclear guidelines, undecided decisions, or ambiguity in the annotation process before the data ever reaches the model.

In this post, I'll share what I've learned about identifying disagreements, understanding their root causes, and deciding what to do about them. 

# What Should the Data Represent?

The first thing to decide is always: **What do we want our data to represent?** Are we trying to capture how people see things differently, with all the complexity of human opinions? Or do we want the labels to follow a set of rules we defined, in order to reach a particular product or business objective? 

Röttger et al. (2022), in their research, described two paradigms for NLP data annotation tasks:

- **Prescriptive:** Annotators follow a set of rules to produce consistent labelling results. The goal is to adopt a specific interpretation, rather than show individual views.
- **Descriptive:** Annotators provide individual judgments based on their own interpretations. The goal is to preserve the differences so they reflect the diversity of human perspectives.

### Prescriptive annotation: consistency with a shared policy

I think most of my projects fell into this category. Before building the algorithm, we already had a clear picture in mind of what our product was going to be like. 

Like, one of my cases was to build a Search Results Page for an e-commerce site. The function of SRP should be pretty straightforward: finding people the relevant products. But think about this: when someone searches for "iPhone 17," after showing all the iPhone 17 models in our catalogue, what should come next? Should we show iPhone 17 accessories? Or other models, like the iPhone 17e or iPhone 18? Which one is more relevant?

There's no one correct answer. It really depends on what kind of experience we want to create and what makes sense for the business. Prescriptive annotation is useful when:

- The organisation has a specific product goal or desired behaviour in mind.
- The downstream services require predictable and reproducible results.

### Descriptive Annotation: Preserving Meaningful Differences

Sometimes, we want our dataset to capture how people think, feel, and interpret things when facing the same context. In one study, Wang et al. (2023) compared the safety ratings from over 100 crowd-sourced annotators with Trust and Safety experts. They found disagreement demonstrated meaningful signals related to dialogue content, structure, and reasoning. Rather than assuming experts are always right, the descriptive result preserved what each group could reveal about safety. 

And this isn't an academic-versus-commercial distinction. A company might use descriptive annotation to understand how different users see its products.

# **Building Consistency into the Annotation Process**

The fundamental step is **calibration**. Calibration in annotation means making sure annotators have a clear understanding of the task and its guidelines. Best practice is to start this process as early as possible, before a small misunderstanding spreads across hundreds of labels. I usually ask annotators to complete a small batch at the beginning of the process, so I can see how they're interpreting the guidelines and catch any confusion early on.

During the process, we have different expectations of what consistency should look like in prescriptive and descriptive annotation: 

## Prescriptive annotation ↔ Inter-annotator Agreement

For prescriptive annotation, we care about inter-annotator agreement. That’s everyone should follow the same rules and reach the same decisions, regardless of who's doing the labelling.

We usually have multiple annotators label the same data. Majority vote is a quick way to reach consensus when disagreements happen. If a disagreement leaves us with a tie, then we need a third reviewer or someone to make the final decision.

But voting has its limits. A room full of people can be confidently wrong together! And when a case reveals something the guidelines never covered, adding more votes won't solve the problem. 

## Descriptive Annotation ↔ Intra-annotator Consistency

For descriptive, we’re more interested in whether an annotator’s judgments stay consistent over time, which is known as intra-annotator consistency. If we ask the same person to judge a similar case again, would they give a similar answer? **Repeated judgments** help us understand whether their work is stable.

One interesting study I found is Gordon et al. (2021), who tracked intra-annotator consistency through a method called *Disagreement Deconvolution*. The team used **test-retest annotation**, which was kind of like repeated judgments where the same annotator labels the same case. However, there weren’t enough repeated judgments. They decided to group cases based on level of disagreement to estimate how often annotators give an answer different from their usual judgment. This helped them separate genuine differences in opinion from inconsistencies.

Having multiple annotators review the same cases is still useful. When their judgments differ, our goal isn’t to reach an agreement, but to discuss those differences, which may reveal errors or things others have overlooked.

## Research through Disagreement Patterns

I often utilise agreement metrics, such as Cohen’s Kappa, Krippendorff's alpha, raw agreement rates, and label-by-label confusion tables, to observe the overall disagreement distribution within our dataset. I found it a very helpful way to discover problems in our training, guidelines, tooling, or process.

Here are a few patterns I usually look for:

- **Disagreements spread across different categories and annotators:** This might suggest a system problem, such as unclear guidelines, missing context, or inconsistent training. Though in descriptive annotation, some of these differences may be exactly what we want to capture.
- **Disagreements concentrate in certain categories:** This helps us narrow down where the confusion comes from. We might need a clearer definition of these cases, or perhaps we've discovered a blind spot we hadn't considered.
- **One annotator always disagrees with others:** one-on-one conversation needed. It’s worth reviewing the reasoning behind their judgments. The disagreements may be caused by misunderstanding of the guidelines, or this person may have spotted something everyone missed.
- **Everyone agrees, but everyone is wrong:** This one is difficult for agreement metrics to catch. High agreement doesn't guarantee correctness. Though it seems like everyone has reached agreement, we still need sampling quality checks to make sure we don't have a collective blind spot.

# How to make sure consistency when disagreements are resolved.

One of the most frustrating parts of managing annotation projects is discovering a problem halfway through. Maybe the product team suddenly decides to change the ranking order. Or we discover that one annotator has been applying a rule incorrectly for weeks. These things happen more often than we'd like. 

The question is no longer just how to resolve the disagreement. **What about all the data we've already labelled?** How are we going to trace back to the previous work effectively? 

## Deterministic Check

If the problem follows a clear pattern, writing a simple script can save us a lot of work. I once worked on a summarisation evaluation where, halfway through, we decided to change the sentence-boundary calculation. We used regular expressions to scan the entire dataset instead of checking thousands of sentences by hand. 

## Manual categorisation

Changing the judgement is so common that I find it helpful to just embed a tracing into our system. A more traditional and manual approach is to ask the annotator to categorise data or issues while they’re reviewing the data. It’s easier for us to find the related issue altogether later 

Annotators can propose new categories based on their observations throughout the process, since we can’t predict every problem in advance. We keep the definitions and examples in a shared document so team can stay on the same page. However, a category created halfway through a project doesn't magically appear on older records. Earlier annotations might contain the same issue without the tag. Different annotators might also create overlapping categories or interpret them differently. 

### A language model can help maintain the taxonomy

LLMs could help us manage these categories as the taxonomy grows. When a new category gets proposed, an LM could compare its definition and examples with existing ones to spot potential overlaps.  It could also scan the data to map a new category into previous data. 

Of course, I wouldn't let the model make that decision on its own. Two categories might sound similar but represent different problems in practice. Let the model make suggestions, but to keep the process under control, people should be the ones making the call. 

# References

1. Röttger, P., Vidgen, B., Hovy, D., & Pierrehumbert, J. (2022). Two contrasting data annotation paradigms for subjective NLP tasks. In *Proceedings of the 2022 Conference of the North American Chapter of the Association for Computational Linguistics: Human Language Technologies* (pp. 175–190). Association for Computational Linguistics. https://doi.org/10.18653/v1/2022.naacl-main.13
2. Wang, D., Díaz, M., Parrish, A., Aroyo, L., Homan, C. M., Serapio-García, G., Prabhakaran, V., & Taylor, A. S. (2023). *All that agrees is not gold: Evaluating ground truth labels and dialogue content for safety* [Preprint]. Google Research. https://research.google/pubs/all-that-agrees-is-not-gold-evaluating-ground-truth-labels-and-dialogue-content-for-safety/
3. Gordon, M. L., Zhou, K., Patel, K., Hashimoto, T., & Bernstein, M. S. (2021). The disagreement deconvolution: Bringing machine learning performance metrics in line with reality. In *Proceedings of the 2021 CHI Conference on Human Factors in Computing Systems* (pp. 1–14). Association for Computing Machinery. https://doi.org/10.1145/3411764.3445423
