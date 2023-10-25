---
title: An Informal, Formal Introduction to Vector Spaces
subtitle: Building mathematical intuition from the ground-up. Part 1.
date: 2023-10-25
word_count: X words ~Y minute read
generate_toc: true
---

I've been exposed to linear algebra many times. First in school, then as a Mechanical Engineering undergraduate, and now as a Machine Learning PhD student. I have a pretty good grasp on how to use maths as a _mechanism_ to solve problems. However, I've been wondering recently if I'm missing out on something. Maybe if I re-learn maths again, as a _mathematician_, I'll gain a deeper appreciation and understanding for the concepts. Perhaps this will even unlock new skills and avenues for me as a researcher.

This series contains first-principles introductions to the _concepts_ behind mathematical topics that I've been self-studying. It will be best enjoyed by readers like me, who have already been exposed to these topics informally in science or engineering or computing. I'll focus on intuitively explaining the ideas behind the topics, which will hopefully help you to get over the conceptual hurdles that I struggled with during my own learning. It is not a substitute for textbook study and is best read as an accompaniment to more rigorous resources.

This post is about understanding vector spaces. I'll try to clear up some points that I got stuck on when learning linear algebra more formally. For example: what exactly _is_ a vector? What about fields? algebras? Why do we need abstract algebraic structures anyway?

## So what is a vector, actually?

A vector is an element of a vector space. While this is a pithy answer, internalising it is important. _Vector spaces_ are the fundamental mathematical objects we're studying. 'Vector' is just a shorthand word mathematicians use so they don't have to keep repeating 'element of a vector space' in conversation.

The 'vectors' you've been exposed to so far -- row and column vectors like $[1,2,3]$ and $[-4,0]^T$ -- are more accurately described as elements of finite-dimensional vector spaces over the reals. Let's explain what this means.

_NB: In my experience, I find it helps to start off by throwing away any notions and intuitions you already have about vectors, matrices, tensors, etc. It's especially important to stop thinking of vectors as little arrows. We can then build back up from a more solid foundation. It's quite satisfying to eventually learn the deeper reason why you were initially taught those notions._

## Okay, so what is a vector space?

A vector space is simply a set $V$, with well-defined operations of addition ($+$) and scalar multiplication ($\cdot$). For notation, we often represent vector spaces as 3-tuples: $(V, +, \cdot)$. The addition and scalar multiplication properties must satisfy a few rules (e.g. commutativity, associativity, etc.), and if you are familiar with Abelian groups, you may recognise that a vector space is just an Abelian group, with an extra scalar multiplication operation defined. Any vector space $(V, +, \cdot)$, can thus be converted into an Abelian group $(V, +)$ by deleting the scalar multiplication operation.

_NB: I won't list out all the properties needed for an object to qualify as a vector space. I want to keep this as a fairly light discussion and I don't want to get lost in mathematical definitions. Besides, this is not meant to be a standalone introduction to the topic -- textbooks are much better suited to that role._

For example, our old concept of two-dimensional 'vectors' can be defined as a vector space over $\mathbb{R}$ like so:

$$ \mathbf{x} = \begin{bmatrix} x_1 \\ x_2 \end{bmatrix}, \quad \mathbf{y} = \begin{bmatrix} y_1 \\ y_2 \end{bmatrix}, \quad \mathrm{where} \quad \mathbf{x},\mathbf{y} \in \mathbb{R}^2, $$

$$\mathbf{x} + \mathbf{y} = \begin{bmatrix} x_1 + y_1 \\ x_2 + y_2 \end{bmatrix}, \quad a \cdot \mathbf{x} = \begin{bmatrix} ax_1 \\ ax_2 \end{bmatrix}, \quad \mathrm{where} \quad a \in \mathbb{R}.$$

Okay, so at this point, you're probably pretty annoyed with me. I've just used a lot of fancy words to arrive back at something that you learned in your first linear algebra lesson. The key insight comes once we understand that the definition of vector space is not restricted to the types of vector that we've seen so far.

## Vector spaces are more general than you think

The definition of a vector space is surprisingly general. _Any_ 3-tuple $(V, +, \cdot)$ which satisfies the necessary properties _is_ a vector space. So far, we've looked at a vector space where $V = \mathbb{R}^2$, the set of all two-dimensional lists (i.e. 2-tuples) containing real numbers. However, in general, $V$ does not have to be a set of lists. $V$ can be a set of sets, functions, lists, or other mathematical objects. Similarly, the addition and scalar multiplication operations need not be identical to the ones we saw in our standard $\mathbb{R}^2$ vector space.

For example, the set of degree $n$ polynomials, with real coefficients ($P_n[x]$) forms a vector space over $\mathbb{R}$:

$$ P_n[x] = \{\sum_{i=0}^n c_ix^i : c_i \in \mathbb{R} \}, \quad \mathrm{consider} \quad \mathbf{u},\mathbf{v} \in P_n[x]: $$

$$ \mathbf{u}(x) = u_0 + u_1x + ... + u_nx^n , \quad \mathbf{v}(x) = v_0 + v_1x + ... + v_nx^n, $$

$$ (\mathbf{u} + \mathbf{v})(x) = \sum_{i=0}^n (u_i + v_i)x^i, \quad a \cdot \mathbf{u}(x) = \sum_{i=0}^n au_ix^i, \quad a \in \mathbb{R} .$$

We're not used to thinking of polynomials as vectors, but they are indeed elements of a vector space. Polynomials aren't even a particularly wild example. I'll leave it up to you to try your hand at creating more interesting vector spaces.

One thing you might have noticed by now is that I keep saying that things form vector spaces _over_ $\mathbb{R}$. This is related to how the scalar multiplication operation is defined. The easiest way to understand this is to contrast it to the addition operation.

With addition, we take two vectors and map them onto another element in the same vector space ($+: V \times V \rightarrow V$). In contrast, scalar multiplication is slightly different. Scalar multiplication maps a vector and a 'scalar' onto an element of the vector space ($\cdot: F \times V \rightarrow V$, where $F$ is a _field_). Once we have defined the scalar multiplication operation with $F$, we may say that the vector space is _over_ $F$.


## Fields and algebras

Just like vector spaces, fields are painfully abstract definitions for things that seem intuitively obvious. But just like vector spaces, their real power becomes apparent once you understand their generality.

A field is a 3-tuple $(F, +, *)$, consisting of a set $F$, with well-defined addition ($+: F \times F \rightarrow F$) and multiplication ($*: F \times F \rightarrow F$) operations. Again, there are some properties that the operations must satisfy that we will not list out here. Like vector spaces, fields are closely related to Abelian groups. If $(F, +, *)$ is a field, $(F, +)$ is an Abelian group.

You'd be forgiven at this point for not spotting how fields differ from vector spaces. One difference is that fields get a multiplicative inverse operation (i.e. division), whereas vector spaces do not. The other major difference is in the function 'signature' of the multiplication operations. In fields, $*$ takes two elements of a field and maps onto another element of the same field. Contrast this to the 'signature' of the $\cdot$ operation for vector spaces in the section above.

In practice, this distinction means that $F$ tends to be a set of _numbers_, whereas $V$ is more often a set of _collections of numbers_. In the case where $V = F$, $*$ and $\cdot$ coincide, and we may say that the field is a vector space over itself. Every field forms a vector space over itself.

Some quickfire examples of sets that form fields are $\mathbb{C}$, $\mathbb{R}$, and $\mathbb{Q}$. Note that the integers $\mathbb{Z}$ do not form a field because they are not closed under division (e.g. $2^{-1} \not \in \mathbb{Z}$).

## Complex vector spaces

Note on complex numbers and fields

Note on how complex numbers are already sort of vectors

At this point, the water we're treading in goes very deep. When you start asking these kinds of questions, you very quickly end up talking about quaternions. While writing this post, I ended up in a rabbit hole on the [Frobenius theorem](<https://en.wikipedia.org/wiki/Frobenius_theorem_(real_division_algebras)>), which explains why associative division algebras can only have dimension one ($\mathbb{R}$), two ($\mathbb{C}$), or four ($\mathbb{H}$). Let's wade back to the shallows before we get lost at sea.

