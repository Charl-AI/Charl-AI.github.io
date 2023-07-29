# My Blog

Heavily inspired by [this](https://ekiim.xyz/blog/entries/blog-with-pandoc-and-git/) example, my blog uses a shell script + pandoc as a homemade static site generator.

This repo is centred around the `blg` script, which performs a tree map, converting all *.md files in src/ to *.html files in build/. Example usage:

```bash
blg --build    # Build website to build/
blg --serve    # Serve build/ on localhost
blg --clean    # rm -rf build/
blg --help     # show help message
```

We also include a helper script `lsblg` which finds all blog posts, extracts metadata, and formats in markdown style. This should be used to generate the blog index:

```bash
lsblg > src/blog/index.md
```

To build the blog and ensure that the index is up to date:

```bash
./lsblg > src/blog/index.md && ./blg -b
```

There is one final helper script `wcpp` which pretty prints a word count and reading time summary for each post.

## To create a new blog post

1. Create the markdown file in `src/blog/new_post_short_name/index.md`
2. Add the following metadata to the markdown file:

```markdown
---
title: post title
subtitle: descriptive or witty subtitle
date: yyyy/mm/dd
word_count: X words ~Y minute read
---
```

3. Write the post
4. Find the word count with `wc -w path/to/host` -- divide by 230 to get a reading time estimate.
5. Run `./lsblg > src/blog/index.md` to update list of posts
6. Build blog with `./blg -b`
7. View changes locally with `./blg -s`
