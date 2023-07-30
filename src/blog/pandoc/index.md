---
title: A Tiny Static Site Generator with Pandoc and Bash
subtitle: For people who are allergic to frameworks.
date: 2023-07-29
word_count: 1462 words ~5 minute read
---

I recently decided to overhaul this website. The inciting incident came when, after over a year of not touching it, I tried to write a new post. The blog was being built with Jekyll and a template that I found online. When coming back to the repo, I realised that I had no idea how anything worked.

In recent months, I have become more and more convinced by the Unix philosophy and have enjoyed solving problems with small and modular shell scripts. When looking at alternatives to Jekyll, such as Hugo and Gatsby, I despaired at the thought of having to learn these massive frameworks without fundamentally understanding what's they're doing.

Inspired by [ekiim's blog](https://ekiim.xyz/blog/entries/blog-with-pandoc-and-git/), I developed a custom static site generator which builds this website. After reading this post, you will be able to too! 

_Note: web development is well outside my area of expertise, so this post is admittedly pretty basic. It's the guide I wish I had before I started this project. The target audience is people like me who want to build simple, brutalist websites from scratch._

---

## Table of contents
[What is a static site generator, actually](#1)\
[How to structure your blog](#2)\
[`blg`: the beating heart of this project](#3)\
[HTML styling and metadata](#4)\
[Bonus: auto generate blog index with `lsblg`](#5)\
[Bonus: serving on localhost](#6)\
[Bonus: deploying to GitHub pages](#7)\
[Conclusions](#8)

---

## What is a static site generator, actually? <a name="1"></a>

A static site is just a collection of html files. However, writing html directly is cumbersome, so most people prefer to write their content in a format like markdown. A static site generator simply compiles the markdown content into a collection of html pages to display. Notice that there's nothing special going on here -- all we need to do to make a basic static site generator is to wrap a document converter and add some convenience features.

`pandoc` (the 'universal document converter') is great for this job. It can convert markdown to html, respecting code blocks, LaTeX-style maths, and embedded assets. It also lets you inject extra html and css for styling the output. For a basic example, try this (you can open the html file in your web browser):

```bash
echo "# Here is some markdown" > example_post.md
echo "## Here is some more markdown" >> example_post.md

pandoc example_post.md -o example_post.html
```

Believe it or not, we are 90% of the way there! Now, we just have to sort out how to apply this to a convert a markdown blog to a website.

## How to structure your blog <a name="2"></a>

We're going to make a script which maps all markdown files in your `src/` directory to html files in an output `build/` directory, respecting the tree structure. This works best if you structure your blog like so:

```
.
├── README.md
├── src
│   ├── blog
│   │   ├── example-post
│   │   │   ├── assets
│   │   │   │   └── fig-1.jpeg
│   │   │   └── index.md
│   │   └── index.md
│   └── index.md
├── blg
├── metadata.yaml
└── default.html
```

We name each page of the website `index.md` -- when the site is built, they will be converted to `index.html`, which browsers will display by default when navigating to their parent directory. For example, the top level `index.md` file is your homepage and lives at the absolute root of the site once it's built. Inside your markdown files, you can link to pages like so: 

```
[This is a link to the homepage](/)
[This is a link to the blog page](/blog)
[This is a link to the example-post page](/blog/example-post)
```

We will be using the `--embed-resources` pandoc option to compile any images into the final html. You can include any assets in your markdown files using their actual location in the repo:

```
![alt-text](src/blog/example-post/assets/fig-1.jpg)
```

We will come to the `metadata.yml` and `default.html` files in a second, but first, let's focus on `blg`.

## `blg`: the beating heart of this project <a name="3"></a>

The `blg` bash script does all the heavy lifting. Conceptually, it performs a tree map, converting all markdown files in `src/` to html files in `build/`. Since pandoc embeds the assets into the built html, we can simply skip over all non markdown files. The full script can be found [here](https://github.com/Charl-AI/Charl-AI.github.io/blob/main/scripts/blg), but for brevity, I'll just show the important bit:

```bash
pages=()

# simply find all markdown files in src
readarray -d '' pages_found < <(find "src" -maxdepth 3 -type f -name "*.md" -print0;)
pages+=("${pages_found[@]}")

num_entries=${#pages_found[@]}
echo "Found $num_entries pages in src/"
echo "${pages[@]}"

for page in "${pages[@]}"; do
  ( 
  # output file has html extension and no src/ prefix
  new_page="${page%.md}.html"
  new_page="${new_page#src/}"

  # make the tree structure in build/ reflect the one in src/
  directory=$(dirname -- "$new_page")
  mkdir -p "./build/$directory"

  # build the page 
  # --katex makes math look best, but you can use --mathjax for a faster build
  pandoc "$page" -o "./build/$new_page" \
    --katex \
    --template=default.html \
    --metadata-file metadata.yaml \
    --embed-resources

  echo "Built $page"
  ) & # process all pages in parallel
done
wait # wait for all pages to be built
```

There's not much else to say about this script. It does one thing, and does it well. You could always add the ability to skip bulding unchanged pages, however, I would consider this a premature optimisation (at the time of writing, this site has <10 pages, and builds instantly).

## HTML styling and metadata <a name="4"></a>

`pandoc` allows us to inject metadata and html into our documents. The `metadata.yml` file specifies default values, which will be overridden by anything found in the header of your markdown files. In my blog, I chose to include the following metadata in each post:

```markdown
---
title: post title
subtitle: this is a subtitle
date: YYYY/MM/DD
word_count: X words ~Y minute read
---
This is the content of the markdown post,
the bit above is the metadata header.
```

Once we've got this metadata about the posts, we can now write our `default.html` file, which styles the output. A bare minimum example is shown below, showing how to use the markdown metadata, as well as including a basic header bar.

```html
<!DOCTYPE html>
<head>
  <meta charset="utf-8" />
  <meta name="generator" content="pandoc" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes" />
  $if(title)$
  <title>$title$</title>
  $endif$
  $if(subtitle)$
  <meta name="description" content="$subtitle$">
  $endif$
  <style>
  <!--Custom CSS goes here-->
  </style>
  $if(math)$
    $math$
  $endif$
</head>
<body>
  <div class="navbar">
    | <a href="/">Home</a> | <a href="/blog">Blog</a> |
  </div>
  $if(title)$
  <header id="title-block-header">
    <h1 class="title">$title$</h1>
    $if(subtitle)$
    <p class="subtitle">$subtitle$</p>
    $endif$
  </header>
  $endif$
  <main>
    $if(date)$
    <p class="date"><i>$date$</i></p>
    $endif$
    $if(word_count)$
    <p class="word_count"><i>$word_count$</i></p>
    $endif$
    $body$
  </main>
</body>
</html>
```

This is all we need for a working website! You can now run the `blg` script with this html template and you're done. It will look pretty spartan at this point, so be sure to add some CSS in the `<style>` section.

## Bonus: auto-generate blog index with `lsblg` <a name="5"></a>

For me, the `src/blog/index.md` file is pretty simple (just a reverse-chronological list of my blog posts, with their metadata displayed in a pretty way), however, it's a hassle to keep this up to date when you are changing posts.

I solved this with the `lsblg` script, which auto-generates the blog index from your posts and metadata. You can find the script [here](https://github.com/Charl-AI/Charl-AI.github.io/blob/main/scripts/lsblg), and run it with `./lsblg > src/blog/index.md`. 


## Bonus: serving on localhost <a name="6"></a>

Simple. Just run this, no installation required:

```bash
python3 -m http.server --directory build/
```

## Bonus: deploying to GitHub pages <a name="7"></a>

If you want to host your site on GitHub pages, like this one, you've basically got three options:

1. Build site locally, commit the final html to the repo. Let GitHub pages publish the `build/` dir.
2. Use a GitHub action to build and publish the site remotely.
3. Build locally, upload the zipped html site as a GitHub release, publish using GitHub action.

I prefer option 3. It allows me to build the site locally without checking it into version control. I wrote [this](https://github.com/Charl-AI/Charl-AI.github.io/blob/main/.github/workflows/static.yml) GitHub action to publish the site each time I upload a new version of the site to GitHub releases.

## Conclusions <a name="8"></a>

Building a website from scratch is way easier than I thought. This may not be the most fully-featured site, but I like it more than anything I'd have gotten from a hugo template. It feels like, by making it myself, I imbued a little of my own personality into it. I am genuinely proud of it.

The experience has also affirmed my faith in the Unix philosophy -- why buy into a complicated framework when you can achieve your goals with simple scripts?
