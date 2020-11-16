---
layout: post
title: Duplication is evil
subtitle: How any scale of duplication can hurt you
tags: [language-agnostic, good-practices]
comments: true
---

It is generally recognized that duplication in software is a bad thing. It increases maintenance cost and creates room for inconsistencies and bugs. I'll personally go further:
duplication is plain evil. Yet, the temptation can be great to simply copy a bit of code from somewhere else in the system instead of refactoring to make it accessible from 
where you want it.
 
In this short article, I'll try to prove that no matter the type and scale of the duplicated piece of code or software, it is generally more expensive and risky
in the long term to duplicate code than to refactor it to avoid any duplication. With that said, no guideline should be absolute and there are exceptions to everything. I just hope this will 
at least convince you to have an extremely high bar on absence of duplication, and only limit it to specific situations. 

# Categorizing duplication instances

One possible way of classifying duplication types is the following (at least that's how I personally think about it):

- code duplication (e.g. duplicated snippet instead of having a helper method)
- logic duplication (e.g. duplicated type hierarchy, error handling logic, geometry logic etc)
- business logic duplication (similar to logic duplication, but specifically logic that is directly linked to business requirements)

# How duplication can and will harm you

- *Code duplication* harms conciseness, readability and maintainability, but it generally doesn't hurt functionality as the duplicated bits are very simple. This is therefore the most
 tolerable form of duplication, which doesn't mean it should be accepted when it's easy to avoid.
- *Logic duplication* harms in all the ways code duplication does, and in addition harms correctness. Indeed, duplicating logic almost inevitably leads to a divergence in the logic of 
at least one of the duplicated bits. Logic duplication is unacceptable in most instances, and even when it looks acceptable ("temporary" migration), it is strongly discouraged because 
in real life, "temporary" often turns into "forever". Trust me, I've seen this happen countless times with people full of good intentions and motivation, including myself. Things don't
always go as planned so we should not make over-optimistic assumptions, and ensure our system is always in a pretty good shape.
- *Business duplication* harms in all the ways code duplication does, except that the type of correctness breaches it causes are generally worse, because it directly leads to violating
 a business requirement. Business duplication is as unacceptable as logic duplication, and when it does happen it requires even stronger justifications (e.g. very high cost to not 
 duplicating).
 
If it wasn't bad enough in a single system, duplication tends to also occur cross-service, if your team owns several services. The typical example would be a team which eventually splits
into two closely related sub-teams as it grows, and which doesn't take enough care to keep a coherent system. Neatly encapsulating your cross-functional features (logging, monitoring,
generic error translation etc) into shareable components will make it easy to bootstrap new services, maintain them, use consistent operational tools, which facilitates debugging, 
development, and collaboration between the sub-teams.
 
# Take-away 

Here's what I personally push for in the teams I work with:

* very low tolerance for any kind of duplication within a module or a service (group of logically related modules)
* low tolerance for duplication of cross-functional logic across several services owned by the same team, or close sub-teams
* mild tolerance for duplication of non cross-functional logic across several services, except if it's business logic.
 In this case, the tolerance should be next to zero and the fact the problem even exists might point at architectural issues (unclear responsibility split).
 
 Often, you'll find that the cost of avoiding duplication isn't that great, and it will make your code so much clearer in the case of *code duplication*. 
 Your efforts will definitely pay off!