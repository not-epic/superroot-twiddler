# Superroot

# Twiddler Quick Start Guide

## Introduction

The twiddler framework is the part of Superroot ([http://go/sr](http://go/sr))
responsible for re-ranking of results from a single corpus. (The other
major ranking component in Superroot is the universal packer, which
combines results from multiple corpora, i.e., for universal search.)

This article is a summary of what the twiddler framework can do and of
guidelines to using its various features. It is by no means an exhaustive
discussion, but rather a starting point for further study. If you haven't
used twiddlers before, we hope that this will give you enough to know
whether you should look deeper: there are also links to larger
documents and wikis scattered around this article and we are happy to
chat. If you've done ranking since before there were twiddlers, we hope
you'll still find some interesting perspective. In any case, please check
the [Twiddler YAQS queue](go/twiddler-yaqs) if you have any questions.

A [twiddler](https://cs.corp.google.com/#piper///depot/google3/quality/twiddler/twiddler.h) is a C++ object that makes ranking recommendations
(twiddles) given a provisional search response from a single corpus.
Twiddling differs from Ascorer ranking in that twiddlers act on a ranked
sequence of results, rather than results in isolation.

There are two supported types of twiddler: predoc and lazy. Predoc
twiddlers run on thin responses, which typically have several hundred
results that don't contain any docinfo (snippets and other data). These
twiddlers run over the full set of results returned from the backend.

After all predoc twiddlers have run, the framework reorders the thin
results. It then makes an RPC that fetches docinfo for a prefix of the
results, runs lazy twiddlers on that prefix, and attempts to pack a
response. This attempt can fail if, for example, lazy twiddlers filter
results from the top or push them down the ranking. In that case the
framework fetches more docinfo, lazy twiddles the new results, and
tries packing again.

More information about this packing flow can be found in the
[SuperrootBasicIntro](http://wiki/Main/SuperrootBasicIntro).

# Goals and design principles
* Isolation: In contrast to Ascorer, which has relatively few but
complex algorithms developed over longer periods, the twiddler
framework supports hundreds of twiddlers (>65 are currently
active in production in WebMixer alone), each trying to optimize
for certain signals. Under these conditions, letting each of these
components depend on the behavior of the others would result in
unmanageable complexity. Therefore, the twiddler framework
conceptual model is of twiddlers in isolation (without knowledge of
the others' decisions).
* Interaction resolution: Because twiddlers run in isolation, they can
only provide constraints and recommendations of how to change
the ranking. The framework then reconciles these constraints.
* Provide context: The framework provides safe read-only access
to the context in which results are being twiddled.
* Hide the complexities of docinfo fetching and pagination: By
constraining the operations that lazy twiddlers may perform, the
twiddler framework prevents a broad range of pagination bugs—
skipped or duplicated results across search result page
boundaries. This is covered more in the [SuperrootBasicIntro](http://wiki/Main/SuperrootBasicIntro).
* Ease of experimentation: Because they run within Superroot it is
often easier to run ranking experiments by writing a twiddler (you
only need to bring up a few superroot jobs rather than building an
Ascorer section or attachment, or bringing up 1,400 jobs). On the
other side, if you need huge amounts of data, Ascorer is a better
choice

## Writing a twiddler

To create a new twiddler, simply derive from the base `Twiddler` class
in [quality/twiddler/twiddler.h](https://cs.corp.google.com/#piper///depot/google3/quality/twiddler/twiddler.h) and override the `Apply` method:
```cpp
Twiddler::Apply(TwiddlerAPI* api, Rank start, Rank end, int debug,
 			Closure* done)
```
* `api`: Twiddlers call methods on this object to perform their reranking operations. These methods are the focus of the majority
of the rest of this article.
* `start`, `end`: The range of results that the twiddler is allowed act
upon, though it can examine the state of all results.
* debug: The level of debugging requested. For a full discussion of
debugging see [Debugging Options for Superroot](https://g3doc.corp.google.com/superroot/g3doc/user/howto/cookbooks/debugging.md).
* `done`: Twiddlers must always call this closure when they are
done.

There are a few other minor implementation details, such as registering
the twiddler for construction, but they don't require much explanation.
(insert triangle here)

We will discuss the various `TwiddlerAPI` methods in more detail later,
but first some advice: You don't need to understand all of twiddler.h to
use the twiddler framework; in fact, a large fraction of the methods
there have very specialized uses that only one or two projects need.
Here is a way to group the `TwiddlerAPI` operations that we find to be a
useful mind map:

* `Boost` and `BoostAboveResult`: The bread-and-butter APIs and
what most ranking work uses. Almost all BU and most PQ wins
from twiddlers come exclusively from these two methods.
* `Filter`, `max_total`, and `stride` categories: Used to increase
diversity, remove duplicates, reduce unwanted results such as
spam and foreign pages.
* `AnnotateResult`, `AnnotateResponse`: Don't change ranking
directly; used to communicate with GWS and other parts of
Superroot.
* `SetRelativeOrder` and `max_position` categories: Useful in very
special cases; tricky as they don't interact well with the rest of the
framework and need special considerations. Consult with
[superroot-team@](mailto:superroot-team@google.com) if you think you need them.
* `merge_cluster` and `stride_demotion_factor` categories:
Don't use them. Seriously.
* There are also accessors and methods for debug, but they don't
need any special consideration at this point.

The way we generally recommend looking at the twiddler operations is
to use *twiddler methods and category types to express semantic intent*,
rather than focusing on the operational details of what they do and how
they interact. For example, don't use *Boost* when what you want is to
demote a result to the second page, and don't use *max_position* if
you've only determined that a result is better than than the current top
result (if you don't yet know what *Boost* or *max_position* mean, it's
ok, we'll explain further down). This is the best way to ensure that your
wins will stay around as new twiddlers and framework changes are
introduced.


