---
title: Lazy-loading conda for faster shell startup
subtitle: Plus a bonus trick for switching envs.
date: 2023-07-30
word_count: 421 words ~1 minute read
---

In machine learning, we often use conda to manage our python environments, and have several conda installations (on network drives, in $HOME, etc...). It can be a pain to manage them. Worse still, since conda loads on shell startup, it can noticeably slow you down when opening terminals -- especially when loading it off a slow network.

Here's a simple trick for your shell config to alleviate the pain:

```bash
# Uses the first conda installation found in the following list
set -x CONDA_PATH /data/miniconda3/bin/conda $HOME/miniconda3/bin/conda

function conda
    echo "Lazy loading conda upon first invocation..."
    functions --erase conda
    for conda_path in $CONDA_PATH
        if test -f $conda_path
            echo "Using Conda installation found in $conda_path"
            eval $conda_path "shell.fish" "hook" | source
            conda $argv
            return
        end
    end
    echo "No conda installation found in $CONDA_PATH"
end 
```

This snippet is for the fish shell and goes in your `config.fish`. It replaces the block that conda auto-generates when you run `conda init` (the one that begins with `# !! Contents within this block are managed by 'conda init' !!`). If you use bash or zsh, it should be pretty trivial to translate this snippet with the help of an AI chatbot.

This snippet solves both of my problems. It lets me define an ordered list of locations to search for conda environments (so I can have the same dotfiles across multiple machines). It also defers loading conda until the first time I use the `conda` command, preventing conda from slowing shell startup.

## Bonus tip: quick switch environments

I often forget what my conda environments are called. I also often forget what the command is to show them (is it `conda env list` or `conda list env`? I always get this wrong). Even when I do remember, it's annoying to type them all the time, especially when the names are long.

If you use `fzf`, which you should, here is another snippet for your `config.fish`:

```bash
# (c)hancge (e)nvironment with conda
function ce
    conda activate (conda info --envs | fzf | awk '{print $1}')
end
```

With this function, typing `ce` in your shell brings up a list of your conda envs with a fuzzy finder. The env you pick then gets activated. This function plays nice with the above script too, when conda is lazy loading, it just takes a second or two extra for the list to pop up.
