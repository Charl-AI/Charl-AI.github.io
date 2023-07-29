---
title: Devcontainers Diminish Dependency Difficulties in Deep Learning
date: 2022-01-20
subtitle: Friends don't let friends develop in unreproducible environments.
word_count: 3762 words ~15 minute read
---

This post walks you through the basics of using Docker (optionally with VScode Devcontainers) to create reproducible Deep Learning project environments. I first provide some motivation as to why this is a good idea, then explain the design principles that I believe projects should follow in order to improve environment reproducibility. I then give a brief description of relevant docker concepts for deep learning, explain how dockerfiles work, and introduce VScode devcontainers which include some useful features. The post finishes by listing some miscellaneous gotchas, limitations, and tips.

This post is accompanied by a [repo with the examples shown in this post](https://github.com/Charl-AI/Deep-Learning-Devcontainers).

## Table of contents
[Why Containerise Your Development Environment?](#intro)\
[Devcontainers Suggested Structure](#manifesto)\
[Anatomy of Docker and Dockerfiles](#dockerfile)\
[VScode Devcontainers](#devcontainers)\
[Miscellaneous: Tips & Tricks, Gotchas, Limitations, and Workarounds](#gotcha)

# Why Containerise Development Environments? <a name="intro"></a>

The environments we use for GPU-accelerated deep learning research are fragile. They rely on a web of dependencies between the CUDA toolkit, GPU driver, OS, Python version, and Python dependencies. The current standard practice is to use python virtual environments and a `requirements.txt` file to manage dependencies, but this only pins the versions of the python packages used and does nothing to manage the rest of the technology stack we are building on. This makes our environments extremely fragile. How many hours do you think you spend each year fixing broken environments? How many times have you struggled to reproduce papers even if the authors were kind enough to provide a requirements file? Not convinced? Try this: delete the current project you're working on and try to set it up again on a coworker's machine. How long do you think it will take before you become productive on the new machine?

![](src/blog/devcontainers/assets/container_diagram.jpg)

The above figure shows the technology stack that deep learning projects typically sit on, demonstrating how virtual environments alone are hardly enough to ensure full reproducibility. Conda seems to be a solution to this problem - and it definitely helps - but it comes with a few key drawbacks: Conda environments are not portable across operating systems, and they also do not manage OS packages. This is problematic because it is common to include shell scripts with your project to automatically download datasets and do any other setup. Even if this is not a big deal to you, **there is one major drawback to using Conda: lack of incremental buy-in**. If you use Conda, you lock yourself and anyone who wishes to reproduce your project into using it. Although it is a fairly popular tool and many researchers know how to use it, I don't feel comfortable limiting the set of researchers who can reproduce my work to those who are proficient in a non-standard technology.

Containers are a great solution to this problem. They can be thought of as a lightweight alternative to virtual machines and allow the entire technology stack for a project to be reproduced - they also do not preclude you from using any of the other technologies we have mentioned so far. The best solution I have found is to use Docker containers + VScode devcontainer features + Python virtualenv (inside the container). This setup even allows people to reproduce the entire development environment in one click! This article is an introduction to the ideas behind this.

# Devcontainers Suggested Structure <a name="manifesto"></a>

Before diving into the details of how containers work, I think it's first best to demonstrate how this new workflow will fit into your repository. **The only change you will need to make to your current project is to add a `.devcontainer` directory** with two specific files in it. These files are a `Dockerfile` which tells docker how to build the image (specifying details about the OS, CUDA, Python version etc) and a `devcontainer.json` file which tells VScode how to create/access the container with any tools you want to use in the project. These are explained further in the next sections.

An example file tree for your project is shown below:

```
your_brilliant_project/
    ├── .devcontainer/
    │    ├── Dockerfile
    │    └── Devcontainer.json
    ├── src/
    │    ├── main.py
    │    └── other_source_code.py
    ├── LICENSE
    ├── README.md
    └── requirements.txt
```

The key principle behind this file structure is incremental buy-in. For example, someone who does not use Docker can ignore the .devcontainer directory and install dependencies in a virtual environment in the usual way; someone who uses Docker but not VScode can build an image/container from the dockerfile manually; and someone who uses Docker and VScode can install and run the entire project in one click. When you set up the project like this, installation becomes very smooth; below is an example set of instructions for installation, taken from the README of one of my projects.

> ## Installation
>
> This project includes a `requirements.txt` file, as well as a `Dockerfile` and `devcontainer.json`. This enables two methods for installation.
>
> ### Method 1: devcontainers (recommended for full reproduction of development environment)
>
> If you have Docker and VScode (with the remote development extension pack) installed, you can reproduce the entire development environment including OS, Python version, CUDA version, and dependencies by simply running `Remote containers: Clone Repository in Container Volume` from the command palette (alternatively, you could clone the repository normally and run `Remote Containers: Open folder in Container`). This is the easiest way to install the project. If you use Docker but don't like VScode, feel free to try building from the Dockerfile, although some minor tweaks may be necessary.
>
> *This method requires GPU drivers capable of CUDA 11.3 - you can check this by running `nvidia-smi` and ensuring CUDA Version = 11.3 or greater*
>
> ### Method 2: python virtual environments (recommended if you do not have a CUDA 11.3 capable GPU or if you do not use Docker)
>
> Clone the repository, then create, activate, and install dependencies in a [Python virtual environment](https://docs.python.org/3/tutorial/venv.html) in the usual way. *Ensure you are using Python 3.8 - this is what the project is built on*.
>
> Depending on your CUDA driver capabilities / CUDA toolkit version, you may have to reinstall the deep learning libraries with versions suited to your setup. Instructions can be found here for [PyTorch](https://pytorch.org/get-started/locally/), [JAX](https://github.com/google/jax#installation), and [TensorFlow](https://www.tensorflow.org/install/gpu).

**Notice that there is still one thing that the user must check - their GPU capability**. Unfortunately, there is no simple way to make all code run on all GPUs because some hardware simply has different capabilities. Generally, I would recommend picking a minimum CUDA version that you want your project to support (it is common to pick either 10.2 or 11.3) and using your Dockerfile to enforce this (I'll explain how later). You can then guarantee that anyone whose hardware supports this version can perfectly reproduce your project - those who don't can use a virtual environment and they are no worse off than they would have been otherwise. This is the only compatibility that the user needs to think about and it is made explicit by using containers.

# Anatomy of Docker and Dockerfiles <a name="dockerfile"></a>

Docker is the most popular system for building containers and is the one we focus on today. We will use Nvidia's container runtime which allows GPUs to be visible inside Docker containers. To install Docker with the Nvidia container runtime, follow the official instructions [here](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker).

A `Dockerfile` tells Docker how to build an 'image' and an image can be used to create a container. Building images can take a few minutes (especially if you have lots of requirements) but once the image is built, it is cached and stored locally and containers can then be spun up very quickly. There are lots of resources to help you understand the details/syntax of Dockerfiles, so I won't go into them here. Instead, I provide an example Dockerfile for deep learning projects and annotate it with explanation of the key ideas below.

```dockerfile
# Pull from base image (1)
FROM nvidia/cuda:11.3.1-cudnn8-devel-ubuntu20.04

ARG PYTHON_VERSION=3.8

# Install system packages (2)
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    build-essential \
    cmake \
    git \
    wget \
    unzip \
    python${PYTHON_VERSION} \
    python${PYTHON_VERSION}-dev \
    python${PYTHON_VERSION}-venv \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=1000

# Create the user (3)
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && apt-get update \
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# Create virtual environment and add to path (4)
ENV VIRTUAL_ENV=/opt/venv
RUN python${PYTHON_VERSION} -m venv $VIRTUAL_ENV && chmod -R a+rwX $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Install requirements (5)
COPY requirements.txt /tmp/pip-tmp/
RUN pip3 --disable-pip-version-check --no-cache-dir install -r /tmp/pip-tmp/requirements.txt \
    && rm -rf /tmp/pip-tmp
```

**1.** Most dockerfiles start by pulling from a base image. This base image comes from Nvidia and includes CUDA 11.3.1 and CuDNN 8 and is built upon ubuntu 20.04. If your GPU + driver is not compatible with CUDA 11.3.1, the image will fail to build. [Nvidia provide lots of other base images](https://hub.docker.com/r/nvidia/cuda) so you can use a different one if you want to support an older/newer mininum CUDA capability.

**2.** This line installs system packages. You can see that we install Python 3.8 and some other relevant packages such as wget and unzip. Other packages can be added here if you would like to include them in the image.

**3.** Create a non-root user for the container. It is good practice to use the container as a user and not as root (if you use a container as root and edit mounted files, it might mess with the permissions in your local filesystem).

**4.** Create a python virtual environment and add it to the path - [this is equivalent to activating it](https://pythonspeed.com/articles/activate-virtualenv-dockerfile/). Again, it is good practice to use a virtualenv inside your container even if it is not strictly necessary. Some people even use Anaconda inside containers although I think it is a bit overkill (and also has the disadvantage of locking your users into Anaconda). The chmod part is simply to change the permissions to allow the user to add packages while the container is running.

**5.** Install requirements from `requirements.txt`.

*Note, most dockerfiles end with a `CMD` line which specifies what command to run when the container starts, this could be running the application or simply running bash. This is not included in this example because you do not need that line when using VScode devcontainers. If you want to use the Dockerfile without VScode devcontainers, you will need to add this line.*

# VScode Devcontainers <a name="devcontainers"></a>

Docker containers are great for ensuring a consistent environment but there's an issue with using them on their own - Docker is mostly designed for deployment, not development. You *could* spin up a container, SSH into it, mount your data and source code, forward any necessary ports, and do your development through the terminal but that would not be fun. Fortunately, VScode has a brilliant feature called Devcontainers which does those steps for you and also attaches a VScode window to the container. This allows you to develop seamlessly using the remote development features of VScode and gives you the same experience as any local project. To use this, simply ensure you have the [Remote Development Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack) installed. The [VScode docs](https://code.visualstudio.com/docs/remote/create-dev-container) are very good and are worth reading but the main thing to understand is that you can use a `devcontainer.json` file to tell VScode how to build the container - an annotated example file is shown below, and the reference docs can be found [here](https://code.visualstudio.com/docs/remote/devcontainerjson-reference):

```json
{
	"name": "Deep Learning GPU: CUDA 11.3",
	// Build args (1)
	"build": {
		"dockerfile": "Dockerfile",
		"context": "..",
		"args": {
			"PYTHON_VERSION": "3.8",
		}
	},
	// Run args (2)
	"runArgs": [
		"--gpus=all",
		"--privileged",
	],
	// Mounts (3)
	"mounts": [
		"source=/vol/biodata/data,target=${containerWorkspaceFolder}/mounted-data,type=bind"
	],
	// settings for the vscode workspace (can also be set when it's running)
	"settings": {
		// This is the venv path set in the Dockerfile
		"python.defaultInterpreterPath": "/opt/venv/bin/python",
	},
    // Extensions to preinstall in the container (you can install more when it's running)
	"extensions": [
		"ms-python.python",
		"ms-python.vscode-pylance",
		"github.copilot",
		"github.vscode-pull-request-github",
		"njpwerner.autodocstring"
	],
	"features": {
		"github-cli": "latest",
	},
	"containerUser": "vscode", // we created this user in the Dockerfile
	"shutdownAction": "none", // don't stop container on exit
}
```

**1.** Behind the scenes, VScode is simply building an image from the Dockerfile by running `docker build`. These are the arguments to provide for that command. Here, we provide the path to the Dockerfile to build, the context (we set it to `..` to make the rest of the project visible to Docker), and the `PYTHON_VERSION` argument that overrides the default value in the Dockerfile.

**2.** Again, VScode starts the container with `docker run`, we set `--gpus=all` to make all available GPUs visible to the container. We also run as `--privileged`. This is not always necessary but fixes a bug that [sometimes happens where your GPUs aren't visible](https://bbs.archlinux.org/viewtopic.php?id=266915).

**3.** Mount any necessary files/data to the container (you do not need to mount the source code of the project because VScode does that automatically). In this example, we mount the `biodata/data/` datasets to a directory in the workspace called `mounted-data/`.

Notice also that you can specify any settings and extensions you want to set up for the workspace in the container. These are not strictly necessary because you can always change extensions/settings while the container is running (i.e. in the normal way through the VScode UI). The nice thing about this is that someone can easily reproduce all your tooling to get a setup that suits the project needs. It appears that companies are starting to see the benefits of this and have started using devcontainers to [help speed up onboarding](https://dev.to/b3ncr/make-onboarding-simple-using-vs-code-remote-containers-2emg).

A nice feature of VScode devcontainers is one-click installation of projects, simply run `Remote containers: Clone Repository in Container Volume` from the command palette (alternatively, clone the repository normally and run `Remote Containers: Open folder in Container`) and you're done! There is a minor difference between these two methods - the first stores your source code in a volume (essentially a filesystem that only Docker can access), whereas the second method involves storing the code in your local filesystem and mounting it to the container. On Linux, there is generally not much difference between these options, but on Windows/MacOS, volumes are generally a bit more performant.


# Miscellaneous: Tips & Tricks, Gotchas, Limitations, and Workarounds <a name="gotcha"></a>

- **How do I make my containers work on SLURM?** In general, containers are great for scaling and distributing computation (e.g. Kubernetes), however, SLURM is not perfectly set up for this. I think there are ways of making containers and SLURM work together but I am not an expert on it at this time. A common workflow is to prototype on a GPU machine, then use SLURM for running large jobs/hyperparameter sweeps etc. Even if you cannot use containers on SLURM, it is still advantageous to use containers for prototyping because you can set up your containers to be identical to the SLURM cluster, hence reducing friction when you want to scale up.

- **How do I use my custom terminal/vim/tmux config in the container?** By default, devcontainers will give you a plain bash shell. You may be tempted to put all your configurations (e.g. zsh, tmux) in the Dockerfile. This is not a good idea, it is best to create a [dotfiles repository](https://dotfiles.github.io/) instead - [VScode can then install your customisations automatically](https://code.visualstudio.com/docs/remote/containers#_personalizing-with-dotfile-repositories) in every container (having a dotfiles repository is good practice anyway because it allows you to replicate your personal customisations and configurations on any machine). You can look at [my dotfiles repository](https://github.com/Charl-AI/dotfiles) for inspiration.

- **How do I move Docker?** All your containers and images are stored in /var/lib/docker by default. This can get pretty big, so you might want to move it. [This post](https://www.guguweb.com/2019/02/07/how-to-move-docker-data-directory-to-another-location-on-ubuntu/) has some useful instructions for doing so.

- **My Docker image is big, should I be worried?** Probably not. A CUDA image with lots of python dependencies can easily be 15GB or more in size. There are methods and best practices to reduce the size of your image but, in general, images for deep learning projects will be pretty big. This is something you will have to get used to - a few gigabytes of space is cheap and is definitely worth trading for better reproducibility and consistency. If space is really a concern, you can consider using a single image with all the libraries you need for all of your projects - this will avoid you storing lots of separate almost-identical images.

- **The build context is big when Docker is building my image.** This is not too worrying but it might slow the process of building images. It is probably because you have your data/venv in the workspace when the container is being built. You can delete the venv because Docker will make one specific to the container if you are using the example Dockerfile I provided. Likewise, if you have any large datasets, you can store them somewhere else and mount them to the workspace in the devcontainer.json (you should probably not be storing large datasets in your project workspace anyway, although it is usually fine for small ones like MNIST). You can also look into using a `.dockerignore` file to list files that you don't want to appear in the build context in a similar way to a `.gitignore` file, although I have found it to be overkill.

- **Which Nvidia base image should I use?** First, decide what CUDA version and OS you want to support, then pick the image that supports that. Nvidia provides three types of images 'base', 'runtime', and 'devel'. I would generally recommend using the devel image if you are using JAX and using the base image if you are only using PyTorch. This is because PyTorch comes bundled with its own version of CUDA, so you will not need all of the features of the devel image.

- **What do the Nvidia base images *actually* do?** Not that much, really. They just give you an OS with CUDA (and sometimes CuDNN) preinstalled and also set a bunch of useful environment variables such as `NVIDIA_DRIVER_CAPABILITIES` and `NVIDIA_REQUIRE_CUDA` (this is the one that ensures you have a GPU capable of running the CUDA toolkit requested). You can actually try using a different base image and you may find that you can still use the GPU (especially if you use PyTorch which comes bundled with its own CUDA toolkit).

- **Then why do your examples use the Nvidia base images and not X/Y/Z other image?** Even though you *could* use a different base image for your projects, doesn't mean you *should*. Yes, it is possible to take a plain python image and install CUDA + set all the relevant environment variables, but you'd just be doing the same thing as the Nvidia images (probably in a less efficient way, too). You might also be tempted to use the images provided by PyTorch/TensorFlow etc... but that is also not a great idea because you will probably have to install your own requirements on top of it anyway so you're back to the same place we were but now you also have to decide whether to include TF/PyTorch in your requirements file - if you do, then using the PyTorch/TF image was pointless - if you don't then your requirements file is incomplete and someone who does not use Docker will not be able to create a correct virtualenv from the file.

- **Can I use devcontainers over SSH?** Yes. You can connect using VScode remote-SSH to connect to the machine you want to run the container on, then [run the container as if you were running it locally](https://code.visualstudio.com/remote/advancedcontainers/develop-remote-host). If this doesn't work, it may be because you need to update VScode, it hasn't always had this feature. One quirk I have found is that the 'clone repository in container volume' option sometimes fails over SSH. I think this is because it is trying to clone it into your local filesystem, not the remote one. I suspect this will be fixed soon, but it's probably best to use the alternate method of cloning the repo into the remote filesystem normally and running 'open folder in container' for now.

- **I can't push my changes to GitHub from inside a container.** The solution to this depends on how you authenticate github. I would recommend using HTTPS and a credential helper such as the [GitHub CLI](https://cli.github.com/) to fix this issue (sometimes if you cloned a repo before you set up gh CLI, you might need to delete it and clone it again). If you use SSH, then you will need to ensure you have an SSH agent running - VScode tries to forward your git credentials and ssh agents into the container. More details can be found [here](https://code.visualstudio.com/docs/remote/containers#_sharing-git-credentials-with-your-container).

- **I am using an institutional account with UID and GID != 1000, do I need to change those values in the Dockerfile?** Surprisingly, no! It turns out that VScode devcontainers are clever enough to update the UID and GID of the container user to match the account you used to create the container. This means the container user has the same permissions as your usual user so you shouldn't have to worry about permissions issues with mounted filesystems.

- **Do I need to make any changes to my requirements.txt file when I use devcontainers?** Not usually, but there are a few situations where you should. For example, if you want to install PyTorch for CUDA 11.3, you will need to run a command like this one `pip3 install torch==1.10.1+cu113 -f https://download.pytorch.org/whl/cu113/torch_stable.html` (from the [PyTorch get started page](https://pytorch.org/get-started/locally/)), however, if you add this to your requirements file in the usual way with `pip freeze`, you will get something like this `torch==1.10.1+cu113`. Now, if you install from the requirements file, you will get an error because pip tried to find torch 1.10.1+cu113 in pypi, when it actually needed to look in the PyTorch website. To solve this, you may need to manually add the find-links to your requirements file - here is an example for installing the CUDA 11.3 versions of PyTorch and JAX:
```
-f https://storage.googleapis.com/jax-releases/jax_releases.html
-f https://download.pytorch.org/whl/cu113/torch_stable.html
jax==0.2.26
jaxlib==0.1.75+cuda11.cudnn82
torch==1.10.0+cu113
```
Note: this is not a container-specific problem and is something you should do in all your projects.

