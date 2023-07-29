---
title: A Tiny Static Site Generator with Pandoc and Bash
subtitle: For people who are allergic to frameworks.
date: 2023-07-29
word_count: X words ~Y minute read
---

I recently decided to overhaul this website. The inciting incident came when, after over a year of not touching it, I tried to write a new post. The blog was being built with Jekyll and a template that I found online. When coming back to the repo, I realised that I had no idea how anything worked.

In recent months, I have become more and more convinced by the Unix philosophy of small and modular command line programs. When looking at alternatives to Jekyll, such as Hugo, I despaired at the thought of having to learn a whole framework without fundamentally understanding what's happening.

Inspired by [this post](https://ekiim.xyz/blog/entries/blog-with-pandoc-and-git/), I developed a custom static site generator which builds this website. After reading this post, you will be able to too! 

## What is a static site generator, actually?

A static site is just a collection of html files. However, writing html directly is cumbersome, so most people prefer to write their content in a format like markdown. A static site generator simply compiles the markdown content into a collection of html pages to display. Notice that there's nothing special going on here -- all we need to do to make a basic static site generator is to wrap a document converter and add some convenience features.

`pandoc` (the 'universal document converter') is great for this job. It can convert markdown to html, respecting code blocks, LaTeX-style maths, and embedded assets. It also lets you inject extra html and css for styling the output. For a basic example, try this:

```bash
echo "# Here is some markdown" > example_post.md
echo "## Here is some more markdown" >> example_post.md

pandoc example_post.md -o example_post.html
```

Believe it or not, we are 90% of the way there! Now, we just have to sort out how to apply this to a convert a markdown blog to a website.

## How to structure your blog

We're going to make a script which maps all markdown files in your `src/` directory to html files in an output `build/` directory, respecting the tree structure. This works best if you structure your blog like so:

```
.
├── README.md
├── src
│   ├── blog
│   │   ├── entry-01
│   │   │   └── index.md
│   │   ├── example-entry
│   │   │   ├── assets
│   │   │   │   └── pattern-1.jpeg
│   │   │   └── index.md
│   │   └── index.md
│   └── index.md
├── blg
├── metadata.yaml
└── default.html
```

Notice that each page is called `index.md`, and lives in a directory with the actual name of the page. This is because it will be converted to `index.html`, which browsers will display when you navigate to the parent directory.

## Finding and mapping

