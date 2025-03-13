---
title: On Writing Beautiful Papers
subtitle: LaTeX tips for paper gestalt.
date: 2025-02-16
word_count: 2256 words ~9 minute read
generate_toc: true
---

I think an underappreciated element of writing good papers is making them look beautiful. It's tempting to believe that beauty shouldn't matter -- surely the science is the only thing we care about, right? But in my experience, this is not true.

Writing papers is about communication. The clearer we can communicate, the easier we make it for people to understand and build on our work. There are many resources online for shaping the _content_ of our work to improve its clarity. However, I think the power of the work's _form_ is often overlooked as a way of improving clarity. In this post, I will emphasise how our formatting choices make loud statements in their own right. I will argue that we can only write beautiful papers if these choices work with our content, not against it.

Besides, there is a lot of research in the world, and no one can read it all in detail. A high level of polish is a strong signal to readers and reviewers that you care about your work. Crafting your work to be clear and easy to follow shows respect for the people who have to read it.

## The big lie in typesetting

This post will contain miscellaneous tips to help make your papers more beautiful and easier to read. However, before starting, it's worth confronting what I think is the most pernicious misconception about writing: that content can be separated from style.

The promise of formats such as LaTeX and HTML is that you can write your content first and then transplant it into whatever template you like for publication. The premise of this post is that this is false.

I say this to emphasise that it's not enough to just apply the template for whatever conference or journal you are submitting to. If you want to write beautiful papers, you will have to write them specifically for their eventual home. Crucially, you will have to tweak your content too. If you are resubmitting a paper and need to change the template, you will likely have to apply all these tricks again to make it look right. There are no shortcuts.

## Write 'blocky' paragraphs

One of the first things that stands out to me when reading a paper is the 'blockiness' of the paragraphs. When the final sentence of a paragraph wraps onto the following line, but only for a few words, the paragraph has an ugly tail and wastes space. For example:

```
--------
--------
--------
--
  ^    ^
  |----|
Wasted space!
```

You can use that wasted space by finishing the sentence at the end of the line. An even better solution is to cut some words to make it finish at the end of the line above.

Often, the best-written papers make good use of space, so they will naturally take on a 'blocky' look, where each paragraph finishes near the end of the line. If you squint at the paper, it should look like a series of rectangles.

Aside from the aesthetic value, writing blocky paragraphs conveys to your readers a feeling of polish and space efficiency. This is not a hard rule (and it is definitely okay to violate it when you're writing math-y portions of the paper); however, it's good practice to ensure that the introduction section is blocky at a minimum.

I think blockiness is also a good example of how the constraints of your format dictate your writing and formatting. Paragraphs should be blocky because they save space if you have a page limit. In contrast, this post is written to be displayed in HTML on my blog (with no space constraints), so I am not worried about blockiness here.

## Never use negative `vspace`

When trying to cram a lot of work into an 8-10 page template, it's often tempting to use negative `vspace` commands to create a little extra space. As subtle as you think you're being, this frequently backfires. It violates the formatting guidelines and usually looks noticeably out-of-place and ugly. However, this isn't even my biggest issue with this practice...

Think briefly about what using negative `vspace` says to your readers. You ran out of space (understandable), but instead of making your work more concise (or applying other space-saving tricks like the one I suggested above), you decided to cram more into the template. It's insulting to your readers -- who now have to get through even more of your waffle than they bargained for -- and it's insulting to other authors who put in the time to condense their work appropriately before submitting it.

Using negative `vspace` is a loud proclamation that you think your work is already as concise as possible. I can assure you it is not. Whenever I read a paper with an egregious `vspace` violation, I take it as a personal challenge to count how many sentences I can reword to be clearer and more concise. I have never failed to find opportunities to do this. On average, I would say that it comes out to at least one sentence per paragraph.

This post is not about writing well, so I won't labour this point any longer. The main takeaway is that bad typesetting choices distract from the content of our work and can unintentionally send the wrong message.

## Cross-reference your contributions

A common practice in machine learning papers is to finish the introduction section with a bullet-point list of the paper's contributions. This is good. However, we can do even better.

One of the issues I have with this practice is that these bullet points are usually created post hoc by the authors because they know that they _should_ include them. This makes it tempting for the authors to pad the list and be vague. As a reader, sometimes these lists feel a bit meaningless and wishy-washy -- exactly the oppisite of their original purpose!

I think the best remedy is to cross-reference each bullet point to the section that makes the relevant contribution. This forces you to be concrete about your contributions and can help encourage you to structure your paper into appropriately scoped sections.

It also allows readers interested in one contribution to jump directly to it. This makes the paper very easy to review. All a reviewer has to do is (1) scan the list and decide if the stated contributions are sufficient for publication and (2) verify that each section adequately does the thing you said it does.

The way I like to do this is to use LaTeX to make the bullet points themselves become the references, for example:
```latex
\begin{itemize}
  \item[\cref{sec:2}] We do X.
  \item[\cref{sec:3}] We do Y.
  ...
\end{itemize}
```

This will get you something like this screenshot, taken from one of my recent papers.

<img align="left" width="100%" src="src/blog/latex/assets/contributions.jpg">

## Include a table of notation

In machine learning papers, we often introduce acronyms and mathematical notation to express our ideas. However, conventions can vary wildly across different papers. Obviously, you should define everything before you use it; however, if you use notation heavily, your readers may be forced to constantly go back and forth to where it was defined. One feature that can help the reading experience is collating it into a table of acronyms and notation. For example, see this screenshot taken from one of my recent papers.

<img align="left" width="100%" src="src/blog/latex/assets/notation.jpg">

I really appreciate papers which include a table like this. Even if it is just in the appendix like this example. I think it shows that you care at least somewhat about the experience readers will have with your work.

## Think about figure placement

A pet peeve of mine when reading papers is having paragraphs interrupted by figures. I find this breaks my reading flow as it forces me to jump down to find the end of the paragraph. Bonus annoyance points if the figure is unrelated to the paragraph I'm reading. This often happens when crossing a page boundary because LaTeX likes to put figures at the top of pages.

This is particularly annoying in two-column layouts. For example:

```
--------  --------
--------  --------

<page break>

------------------
|                |
|    fig.        |
|                |
-----------------

--------
--
^
(flows over from
last paragaph
on page above)
```

The key tip here is: don't trust where LaTeX will automatically place figures. You know best where your figures should go! I like to use `[ht]` as a default setting to try to keep my figures close to the paragraphs they are mentioned in. But you may have to manually move things around to find a placement that works.

Two-column layouts can be particularly tricky because we may have a double-column figure that needs to go at the top of the page. So, in some cases, it may be unavoidable that it breaks up a paragraph. Personally, I'll still try to do everything I can to avoid this, including cutting text to make the paragraph end at the bottom of the preceding page.

## Use author-year citations

We often get the choice of whether to use a numeric or author year style for the in-text part of our citations.

Numeric:
```
X is true [1].
```

Author-year:
```
X is true (Jones et al., 2024).
```

With author-year citations, we get the additional benefit that we can work the citation into the sentence (i.e. `citet` instead of `citep` in natbib):
```
The study by Jones et al. (2024) says X is true.
```

While I appreciate the compactness of the numerical style -- and have indeed used this style in the past -- I much prefer author-year. I've found that, as I get more experienced in my field, I can recognise which papers are being cited from just the in-text part of an author-year citation. This is a really significant benefit. It helps me understand the context and background of the work I'm reading without forcing me to break my flow by going back and forth to the reference list.

One additional benefit of author-year comes from how we typically use it. It's easy to get lazy with the numerical style and just slap a bunch of citations onto the end of a sentence. When we do this, it can be unclear which part of the sentence the citations refer to; it's also easy to cite things that are only tenuously related (or are related in some non-trivial way). When we are forced to work our citations into our sentences, as in author-year, it encourages us to be much more precise about what we are citing and why.

For example:
```
X is true [1-3].

Jones et al. (2020) initially derived result X for linear functions, which was generalised to the injective case by Smith et al. (2021, 2022).
```

While more verbose, I think the latter format is much more explicit. For readers unfamiliar with the literature, it explains the background clearly. Experts in the field can understand the context of the work just by scanning the in-text citations.

## Make your reference list beautiful

One aspect of formatting that we generally don't get much choice over is how to reference other material. For consistency with publication guidelines and reader expectations, we usually pick from one of a handful of predetermined styles (IEEE, Harvard, Vancouver, APA, etc.).

These styles are good (they're standards for a reason!) but may not be perfectly adapted to the environment of today's research. So, for preprints and conferences where you can choose your own citation style, I like to make a few changes to improve the reading experience.

<img align="left" width="100%" src="src/blog/latex/assets/references.jpg">

Again, this screenshot was taken from one of my most recent papers. It has three key features:

1. Designed for conciseness (enough information to unambiguously identify the paper, but no superfluous information such as city of publication, conference proceedings editors, etc.).

1. Backlinks from the reference list to the place in the paper where the work was cited.

1. Hyperlinks from the title of the referenced work to its DOI/URL.

I especially like point 3 (hyperlink to the URL/DOI). In most referencing styles, DOIs and URLs are ugly, take up a lot of space, and are awkward in both print and digital formats.

In print, they are almost useless. Seriously, when's the last time you manually typed a URL you read in a physical book? In digital formats, they are also surprisingly clunky -- they are often not hyperlinked, so we must copy-paste them while fixing any errors introduced by line breaks, etc.

I think that this format helps to make the experience as nice as possible for people reading the work digitally (without removing any important information for people reading print versions). To use this style yourself, copy the LaTeX code from [here](https://gist.github.com/Charl-AI/6d52320a85fe4c53aff2167f0b947b64).

## Conclusions

The aim of this post was to convince you to spend some time making your papers look better and give you some actionable tips to make immediate improvements. I hope that at least some of these tips have been helpful.

Beauty is, of course, subjective, so you're welcome to disagree with my opinions. However, more important than any of the tips I gave here is the general concept that you should empathise with your readers. You already know that you should write your content as clearly as possible; I'm advocating for you to also present it in a way that helps your readers on their journey to understand the work.
