---
layout: post
title: Ehresmann Connection and Gauge Theory
categories: [Math, Physics, QFT, Gauge Theory]
---
## Ehressmann Connection
**tldr;** Connections on Fiber bundles are are specified by vertical bundle-valued one-forms

One might be familiar with the connection as the Christoffel symbols ${\Gamma^\mu}_{\nu\rho}(x)$ or the gauge field $A^\mu(x)$. We will review the idea of [Ehresmann connection](https://en.wikipedia.org/wiki/Ehresmann_connection) that unifies these 2 seemingly disparate structures in a more abstract framework. It is essential if we are going to generalize the Levi Civita connection (Christoffel symbols) on the tangent bundle to a spin connection on the spin bundle.

The Ehresmann connection can be defined for any smooth Fiber bundle $\pi: E\rightarrow M$ with smooth fiber $F=\pi^{-1}(m),\ m\in M$. If we consider tangent bundle $d\pi: TE \rightarrow TM$, one can see that $\dim T_{(m,f)} TE = \dim E = \dim M + \dim F = \dim T_m M + \dim F$, which implies that $\dim \ker d\pi = \dim F$. Intuitively, the tangent bundle $TP$ has some linear subspace that are "along $F$", and this leads to the definition of $$VE := \ker(d\pi: TE \rightarrow TM)$$
One can look for another subspace that is complementary to $VE$, i.e.
$$TE = VE \oplus HE$$
The intuition is that vectors in $HE$ point in a direction that leads to **another fiber**, while vectors in $VE$ point in a direction **tangent** to the **current fiber**. I would love to add a diagram showing an example with $\dim M = \dim F = 1$ (the only intuitive diagram one can draw in my opinion).

 In the absence of any additional structure, there are many valid choices of $HE$, **and our choice of $HE$ affects $\text{ver}(X)$**. This is analogous to saying that the vector space of degree 2 polynomials $\text{span} (1,x,x^2)$ can be decomposed in many ways

 $$
 \begin{aligned}
 \text{analogous to }TE &= VE \oplus HE \\
 \text{span} (1,x,x^2) &= \text{span} (1) \oplus \text{span} (x,x^2) \quad \quad \Rightarrow \text{ver}(x+1) = 1\\
 \text{span} (1,x,x^2) &= \text{span} (1) \oplus \text{span} (x+1, x^2) \ \Rightarrow \text{ver}(x+1) =0 \\
 \end{aligned}
 $$

### Connection Form
Instead of saying we provide a horizontal subspace at every tangent space of the the tangent bundle $TE$, it is an equivalent definition to just provide a projection map
$$\Phi : TE \rightarrow VE$$
from which $HE = \ker \Phi $ can be inferred.

 **Punchline**: One can say that $\Phi$ is a $VE$-valued one-form on $E$ since it takes a vector in $TE$ and gives an element of $VE$.

### Ehressmann Connection on $G$-bundle

**tldr;** Connection on $G$-bundle $P$ is a Lie algebra-valued one-form on $TP$

If we consider $E$ to be a principal $G$-bundle $P$, then the $\Phi:TE\rightarrow VE$ is a Lie group-valued one-form. Since $VE$ consists of vectors that live in the tangent space $T_{(m,g)} E$ that have the same dimension as $\dim G$, the vertical subspace $V_{(m,g)}E \cong T_g G \cong T_e G \cong \mathfrak{g}$ is isomorphic to the Lie algebra. So the connection $\Phi$ on a $G$-bundle is specified by a Lie algebra-valued one-form **on P** (the full bundle $P\rightarrow M$).

However, it turns out that given a local trivialization $\mathcal{U}\times G$ of $P \rightarrow M$, specifying a Lie algebra-valued one-form $\omega^\mathcal{U} : T\mathcal{U}\rightarrow \mathfrak{g}$ on $M$ actually specifies enough information to specify $\Phi_\mathcal{U}$ on $\pi^{-1}(\mathcal{U})$, which is then glued together with other charts to obtain $\Phi$ on the full bundle $P$. This might seem wild at first since surely projecting $\Phi$ onto $M$ to yield $\omega$ will lose information with regard to how this one form behaves on the fibers. The answer is that the condition that there is an action of $G$ on the fibers $G$ is constraining enough to reconstruct $\Phi$ given $\omega$. These $\omega^\mathcal{U}$ are known as the Yang-Mills fields. There are alot of details that are covered in the [excellent lecture 22 by Frederic Schuller](https://youtu.be/KhagmmNvOvQ).

### Details of Theorem
The main result is a theorem
- presented at [21:28](https://youtu.be/KhagmmNvOvQ?t=21m28s) of Schuller Lecture 22 (I follow his notation)
- Theorem 10.1 in Nakahara
- [Wiki](https://en.wikipedia.org/wiki/Connection_(principal_bundle)#Pull_back_via_trivializing_section)

Given a section $\sigma:\mathcal U \rightarrow P$ which induces canonical local trivialization $h:\mathcal{U}\times G \rightarrow P$, a $\mathfrak g$-valued one-form $\omega$ on $P$ can be pulled back via $\sigma$ or $h$.

The $\sigma^* \omega$ pullback induces the $h^*\omega$ pullback by the following definition
$$ h^* \omega_{(m,g)}(v,\gamma) := \text{Ad}_{g^{-1}} ((\sigma^* \omega) (v)) + (L_{g^{-1}})_* (\gamma)$$
where $v\in T_m \mathcal U,\ \gamma \in T_g G$.

Theorem: This $h^* \omega$ satisfies the 2 properties of a connection, namely
- $$(h^* \omega)(h^{-1}_* A^\#) = A$$
- $$(R_g^* (h^* \omega))(h^{-1}_* X) = \text{Ad}_{g^{-1}} (h^*\omega (h^{-1}_* X))$$

where $A \in \mathfrak g$ and $$A^\#:P\rightarrow TP$$ is the fundamental vector field on $P$ generated by $A$, defined pointwise as
$$\ A^\#_{p\in P} := \left.\frac{d}{dt} R_{\exp(tA)} (p) \right\vert_{t=0} \in V_pP \subset T_pP$$

Proof of $$(h^* \omega)(h^{-1}_* A^\#) = A$$:

Since $$A^\# \in V_p P$$, the pushforward via $h$ yields $$h_* A^\# = (0, \gamma) \in T_m \mathcal U \times T_g G$$, where $\gamma$ is
$$\gamma =\left. \frac{d}{dt}  g \exp (tA) \right\vert_{t=0} $$
where $(m,g) = h^{-1} (p)$. It is clear from this that $(L_{g^{-1}})_*\gamma = A$.

This condition is basically saying that the inverse pullback $$h^{-1*} h^* \omega = \omega$$ satisfies the connection one-form condition $$\omega (A^\#) = A$$.

Proof of $$(R_g^* (h^* \omega))(h^{-1}_* X) = \text{Ad}_{g^{-1}} (h^*\omega (h^{-1}_* X))$$:

To prove this,

$$
\begin{aligned}
\text{LHS} & = (R_k^* (h^* \omega))(h^{-1}_* X) \\
 & = h^* \omega( R_{k*} h^{-1}_* X )\\
& = (h^* \omega)_{(m, gk)}(v, R_{k*} \gamma) \\
& = \text{Ad}_{(gk)^{-1}} (\sigma^* \omega(v)) + L_{(gk)^{-1} * R_{k*} \gamma } \\
& = \text{Ad}_{k^{-1}}\text{Ad}_{g^{-1}}(\sigma^* \omega(v)) + L_{k^{-1}*} R_{k*} L_{g^{-1}*}\gamma \\
& = \text{Ad}_{k^{-1}} \left( \text{Ad}_{g^{-1}}(\sigma^* \omega(v)) + L_{g^{-1}*}\gamma \right) \\
& = \text{Ad}_{k^{-1}} \left( h^* \omega(h^{-1}_* X)  \right) \\
& = \text{RHS}
\end{aligned}
$$

So the above calculations has shown that given a $\mathfrak{g}$-valued one-form $\omega^\mathcal{U}$ on $\mathcal{U}$ and a section $\sigma:\mathcal U \rightarrow \pi^{-1}(\mathcal U)$, there exists a unique connection one-form $\omega$ on $\pi^{-1}(\mathcal U)$ such that $\omega^\mathcal{U} = \sigma^* \omega$. The constructed connection depends on $\sigma$.
<!-- In other words, specifying the $\mathfrak g$-valued one-form on the base space $M$ is enough information to uniquely construct the full connection on $P \rightarrow M$.  -->

$\Leftarrow$ construction: Split $P$ into charts $\mathcal U ^ i$. Then each local trivialization induces a section $\sigma_i : M \rightarrow P$. The pullback of $\omega$ on $P$ via $\sigma_i$ will give an $\omega^\mathcal{U_i}$ on $\mathcal{U}_i$, which is a Yang-mills field $\omega^M$ on the base manifold $M$ after gluing. The gluing is what we call "gauge transformations of $A$" or "how $\Gamma$ transforms under change in coordinates".

$\Rightarrow$ construction: Split $P$ into charts $\mathcal U^i$. On each chart, the one-form $\omega^{\mathcal U^i}$ on $\mathcal U^i$ induces a one-form (which is actually $h^* \omega$) on the local trivialisation $h$ of $\pi^{-1}(\mathcal U^i)$ via the theorem above. These $h^*\omega$ can be glued together to construct a one-form $\omega$ on $P$.

### Clearing the Ambiguity in Nakahara's Notation
I learned alot from Nakahara's book, but I find their section on connection one-form slightly ambiguous. For the connection one-form on $P$, they write
$$\omega_i \equiv g_i^{-1}\pi^* \mathcal A_i g_i + g_i^{-1} dg_i$$
where $\mathcal A_i$ is the Yang Mills one-form on the base manifold $M$.
The first term $$g^{-1}_i \pi^* \mathcal A_i g_i$$ actually means
$$\text{Ad}_{g_i^{-1}} (\pi^* \mathcal A_i(X))$$
where $$\text{Ad}_{g_i^{-1}}: T_e G \rightarrow T_e G$$ is the adjoint action of $G$ on $\mathfrak g$, and $X\in T_p \mathcal P$. The $$\text{Ad}_{g^{-1}}(Y)$$ and $$g^{-1} Y g,\ Y \in \mathfrak g$$ notations coincide for matrix Lie groups, where the multiplication between $g$ and $Y$ is matrix multiplication. I still prefer to talk purely in non-matrix notation because I find it's clearer.

Because $X = H + V$ and $V \in \ker \pi^*$, the above actually reduces to

$$\text{Ad}_{g_i^{-1}} (\mathcal A_i(v))$$

where $v\in T_m \mathcal U^i \cong H_p P$ is a horizontal vector. The first term takes care of the vertical component of $X$.

The second term is extremely ambiguous. $g_i^{-1}$ is simply just $$L_{g^{-1}*}$$. Trouble comes from $dg_i$. Usually one uses $dg$ to denote the pushforward of $g$. But in this case, $dg$ actually means $\gamma \in V_p P \cong T_g G$. If I include the implicit annotations,
$${\omega_i}_{(m,g)} (X) \equiv \text{Ad}_{g_i^{-1}} (\mathcal A_i(\text{hor}(X))) + L_{g^{-1}*} (\text{ver}(X))$$
So how does one justify $dg = \gamma$? $g$ is interpreted as a map $\mathbb R \rightarrow G$ in the local trivialization $h^{-1}(p) = (m,g)$. A vector is the derivative of a curve at one point of the curve, so given a curve $C$ in $P$ which leads to vector $X=\frac{dC}{dt}$, we actually mean to define
$$dg \equiv \frac{d}{dt} g(C(t)) \in V_p P$$
i.e. derivative of the $G$ fiber "coordinate" of our curve $C(t)=(m(t),g(t))$. $dg:TM\rightarrow VP $ is the pushforward of the map $g:M\rightarrow P$. So $dg(\pi^*X ) = \text{ver}(X)$.

**tldr;** $dg$ is $\text{ver} (X)$.

## Gluing of Connection One-Form
Given 2 overlapping local trivializations $\mathcal U^i \cap \mathcal U^j$ (which induces 2 local sections $\sigma_i, \sigma_j : \mathcal U^i \cap \mathcal U^j \rightarrow P$), we would like to find how $\sigma_{i}^* \omega$ is related to $\sigma_{j}^* \omega$. The answer is obtained by investigating how they act on a vector $X \in T_{m} (\mathcal U^i \cap \mathcal U^j)$. We need to calculate $$\sigma_{i*} X \in T_{\sigma_i(m)} P $$ in terms of $$\sigma_{j*} X \in T_{\sigma_j(m)} P$$. Or in terms of the local trivialisation $${(h_i \circ \sigma_i)}_{*} X \in T_{(m, e_i)} \mathcal U^i \times G$$ and $${(h_j \circ\sigma_j)}_{*} X \in T_{(m, e_j)} \mathcal U^j \times G$$.

We know that $t_{ij}: \mathcal U^i \cap \mathcal U^j \rightarrow G$ is the transition map defined as the unique (guaranteed since $G$ acts freely and transitively on the fibers of $P$) element $t_{ij}(m) \in G$ such that
$$R_{t_{ij}(m)}(p) = h_j \circ h_i^{-1}(p)$$
This definition can be rephrased in a more friendly manner as

$$
\begin{aligned}
h_i^{-1}(p) = &\ h_j^{-1}(p) \bullet_R t_{ij}(m) \\
\text{Denoting } h^{-1}(p) := &\ (m, g(p)) \text{ ,}\\
g_i(p) = &\ g_j(p) \bullet_G t_{ij}(m)
\end{aligned}
$$

Repeating the derivation with $i \leftrightarrow j$ swapped, we get that $t_{ij} = t_{ji}^{-1}$

If we let $p=\sigma_j(m)$, then the last equation becomes
$$g_i(\sigma_j(m)) = e \bullet_G t_{ij}(m) = t_{ij}(m)$$
which translates the definition into a statement about transition maps between sections

$$\sigma_j(m) = \sigma_i(m) \bullet_R t_{ij}(m)$$

Suppose we have a curve $\gamma(t)$ in $M$ that yields the vector $T_m M \ni X = \frac{d\gamma}{dt} \vert_{t=0}$ at $m:=\gamma(0)$. This vector acts on a function $f:M\rightarrow \mathbb R$. The pushforward is given by

$$
\begin{aligned}
\sigma_{j*}(X)(f) & = \left. \frac{d}{dt} f[\sigma_j (\gamma (t))] \right\vert_{t=0} \\
& = \left. \frac{d}{dt} f[\sigma_i(\gamma(t)) \bullet_R t_{ij}(\gamma(t))] \right\vert_{t=0}\\
& = \left. \frac{d}{dt} f[\sigma_i(\gamma(t))\bullet_R t_{ij}(m)] \right\vert_{t=0} + \left. \frac{d}{dt} f[\sigma_i(m) \bullet_R t_{ij}(\gamma(t))] \right\vert_{t=0} \\
& = \left. \frac{d}{dt} (f \circ R_{t_{ij}(m)})[\sigma_i(\gamma(t))] \right\vert_{t=0}
+ \left. \frac{d}{dt} f[\sigma_i(m) \bullet_R \exp(\log t_{ij}(\gamma(t)))] \right\vert_{t=0} \\
& = \left( \left. \frac{d}{dt} \sigma_i(\gamma(t)) \right\vert_{t=0} \right) (f \circ  R_{t_{ij}(m)})
+ (\log t_{ij}(m))^\#_{\sigma_i(m)} f\\
& = R_{t_{ij}(m)*}\left( \left. \frac{d}{dt} \sigma_i(\gamma(t)) \right\vert_{t=0} \right) f
+ \left(L_{t_{ij}^{-1}*} \left. \frac{d}{dt} t_{ij}(\gamma(t)) \right\vert_{t=0} \right)^\#_{\sigma_i(m) \bullet_R t_{ij}(m)} f\\
& = R_{t_{ij}(m)*} \sigma_{i*} \left( \left. \frac{d}{dt} \gamma(t) \right\vert_{t=0} \right) f
+ \left(L_{t_{ij}^{-1}*} t_{ij*}\left. \frac{d}{dt} \gamma(t) \right\vert_{t=0} \right)^\#_{\sigma_j(m)} f\\
& = (R_{t_{ij}(m)*} \sigma_{i*} X) f + \left(L_{t_{ij}^{-1}*} t_{ij*} X \right)^\#_{\sigma_j(m)} f \\
& = \left(R_{t_{ij}(m)*} \sigma_{i*} X + \left(L_{t_{ij}^{-1}*} t_{ij*} X \right)^\#_{\sigma_j(m)} \right) f \\
\end{aligned}
$$

so the vector fields are related as follows
$$\sigma_{j*}X = R_{t_{ij}*} (\sigma_{i*} X) + \left(L_{t_{ij}^{-1}*} t_{ij*} X\right)^\# $$

Note that each term is responsible for the horizontal $H$ and vertical $V$ component of the vector $X=H+V$. Right $G$-invariance of the horizontal bundle $R_{t_{ij}*} : H_{\sigma_i(m)} P \rightarrow H_{\sigma_j(m)} P$ means horizontal gets mapped onto horizontal. The $$Y^\#$$ vector field generated from an element $Y$ of the Lie algebra $\mathfrak g$ lives in the vertical bundle too, so vertical maps onto vertical.

This allows us to relate $\sigma_i^* \omega$ to $\sigma_j^* \omega$, which is what we call "gauge transformations" of $\mathcal A$.

$$
\begin{aligned}
\sigma_{j}^* \omega (X) & = \omega(\sigma_{j*} X)\\
& = \omega(R_{t_{ij}*}(\sigma_{i*}X) + \omega\left((L_{t_{ij}^{-1}*} t_{ij*} X)^\# \right) \\
& = (R^*_{t_{ij}} \omega)(\sigma_{i*} X) + L_{t_{ij}^{-1}*} t_{ij*} X \\
& = \text{Ad}_{t^{-1}_{ij}} (\omega(\sigma_{i*}X)) + L_{t_{ij}^{-1}*} t_{ij*} X \\
& = \text{Ad}_{t^{-1}_{ij}} (\sigma_{i}^* \omega(X)) + L_{t_{ij}^{-1}*} t_{ij*} X \\
\end{aligned}
$$

So this is the mathematically precise expression for the $A \mapsto g^{-1} A g + g^{-1} d g$

### Comparison with Frederic Schuller Lecture 22
In case you were wondering how to match the result we derived with Schuller's Lecture 22 timestamp [59:08](https://youtu.be/KhagmmNvOvQ?t=3548), where he says that
$$\omega^{\mathcal U^{(2)}}_m = \text{Ad}_{\Omega^{-1}(m)} \circ \omega^{\mathcal U^{(1)}} + \Omega^* \Xi $$
- In our case $\omega$ is a $\mathfrak g$-valued one-form on $P$ whereas his $\omega$ is a one-form on $M$. So our $\sigma_j^* \omega$ is his $\omega^{\mathcal U^{(j)}}$.
- His $\Omega$ is our $t_{ij}$
- His Maurer Cartan form $$\Xi: T_g G \rightarrow T_e G \cong \mathfrak g$$ is just $$L_{g^{-1}*}$$, in this case $g$ is $\Omega$. For our expression to match his, we just rewrite $$L_{t_{ij}^{-1}*} (t_{ij*} X)$$ as $$(t_{ij}^* L_{t_{ij}^{-1}*}) X$$, i.e. we pullback the $\mathfrak g$-valued one-form on $G$ $$L_{t_{ij}^{-1}*}:T_{t_{ij}}G \rightarrow T_e G \cong \mathfrak g$$ via $t_{ij}:M\rightarrow G$ to yield a $\mathfrak g$-valued one-form on $M$ $$t_{ij}^* L_{t_{ij}^{-1}*}:T_{m}M \rightarrow T_e G \cong \mathfrak g$$ So our $$t_{ij}^* L_{t_{ij}^{-1}*}$$ is identified with his $$\Omega^* \Xi$$

<!-- ## Calculating $U(1)$ Connection of Magnetic Monopole
I intend to write a more extensive article discussing [monopoles and gauge theory](https://TODO)!

Given a $U(1)$-principal bundle $P$ over $M=\mathbb R^3 \backslash \{0\}$, TODO fill in details -->

## Calculating Christoffel Symbols
Christoffel symbols $\Gamma$ is just the connection one-form $\omega:T_p M\rightarrow \mathfrak g$ on the base manifold of a tangent [frame bundle](https://en.wikipedia.org/wiki/Frame_bundle) (which can be shown to be a $GL(d)$-principal bundle). The goal will be to calculate how $\Gamma$ changes when we change coordinates.

The nice thing is that a coordinate chart $x:\mathcal U^{(1)} \rightarrow {\mathbb R}^d$ on $\mathcal U^{(1)} \subset M$ already defines a local trivialization of the $GL(d)$-bundle. Namely, the $d$ coordinate functions $x^i: \mathcal U^{(1)} \rightarrow \mathbb R$ can be used to define a section $\sigma^{(1)}:\mathcal U^{(1)}\rightarrow F(T_{\sigma^{(1)}(x)}M)$

$$\sigma^{(1)}(x) = \left(\frac{\partial}{\partial x^1}, \frac{\partial}{\partial x^2},...\ , \frac{\partial}{\partial x^d}\right) $$

A vector $X\in T_p M$ can be written as $X = X^\mu \partial_\mu$, with $\mu=1,...\ ,\dim M$. Hence the "Yang-Mills" one-form on the base manifold $M$ takes the form $$\sigma^{(1)} \omega := \mathbf{\Gamma}_\mu dx^\mu $$, where $$\mathbf \Gamma_\mu \in \mathfrak g$$.

Since $G=GL(d)$, the Lie algebra is just $\mathfrak g = \mathfrak{gl}(d)$, consisting of all $d\times d $ real matrices. So $\mathbf \Gamma_\mu = {\Gamma^i}_{j\mu}$ where $i,j=1,...\ ,d$ are indices for the matrix Lie algebra $\mathfrak{gl}(d)$.

We are interested in the transformations laws for this $\Gamma$. Suppose we have another chart $y:\mathcal U^{(2)} M \rightarrow \mathbb R^d$. Then on the intersection $\mathcal U^{(1)} \cap \mathcal U^{(2)}$, if $\gamma(t)\in M$ is a curve with tangent vector $X=\dot\gamma(0)$,

$$
\begin{aligned}
\sigma_{j}^* \omega (X) &= \text{Ad}_{t^{-1}_{ij}} (\sigma_{i}^* \omega(X)) + L_{t_{ij}^{-1}*} t_{ij*} X \\
\mathbf \Gamma^{(2)}_\mu dy^\mu (X) &= \mathbf \Omega^{-1} \left( \mathbf \Gamma^{(1)}_\mu dx^\mu (X) \right) \mathbf \Omega + \mathbf \Omega^{-1} \left. \frac{d}{dt} \mathbf \Omega(\gamma(t)) \right\vert_{t=0}
\end{aligned}
$$

The last term can be rewritten as

$$ \mathbf\Omega^{-1} \frac{\partial \mathbf\Omega(x)}{\partial x^\mu}  \frac{dx^\mu(\gamma(t))}{dt} = \mathbf\Omega^{-1} \frac{\partial \mathbf\Omega(x)}{\partial x^\mu} X^\mu $$

So with all the indices flashed out,

$$
\begin{aligned}
{\Gamma^{(2) i}}_{j\mu} dy^\mu(X) &= {(\Omega^{-1})^i}_{k} \left( {\Gamma^{(1)k}}_{l\mu} dx^\mu (X)\right) {\Omega^l}_j + {(\Omega^{-1})^i}_k\partial_\mu {\Omega^k}_j X^\mu
\end{aligned}
$$

The vector $$X = X^\mu \frac{\partial}{\partial x^\mu}$$ expressed in the 2nd chart is $$X = \tilde X^\mu \frac{\partial}{\partial y^\mu}$$, with $$X^\mu = \frac{\partial x^\mu}{\partial y^\nu} \tilde X^\nu$$
So

$$dx^\mu(X) = X^\mu = \frac{\partial x^\mu}{\partial y^\nu}\tilde X^\nu$$

Putting this back into the above expression,

$$\begin{aligned}
{\Gamma^{(2) i}}_{j\mu} \tilde X^\mu &= {(\Omega^{-1})^i}_{k} \left( {\Gamma^{(1)k}}_{l\mu} \frac{\partial x^\mu}{\partial y^\nu}\tilde X^\nu \right) {\Omega^l}_j + {(\Omega^{-1})^i}_k\partial_\mu {\Omega^k}_j X^\mu
\end{aligned}$$

Now the transition functions $\Omega:M\rightarrow GL(d)$, or in coordinates, ${\Omega^i}_j: M \rightarrow \mathbb R$ are actually

$${\Omega^i}_j = \frac{\partial x^i}{\partial y^j}$$

$${(\Omega^{-1})^i}_j = \frac{\partial y^i}{\partial x^j}$$

so

$$\begin{aligned}
{\Gamma^{(2) i}}_{j\mu} \tilde X^\mu &= \frac{\partial y^i}{\partial x^k} \left( {\Gamma^{(1)k}}_{l\mu} \frac{\partial x^\mu}{\partial y^\nu}\tilde X^\nu \right) \frac{\partial x^l}{\partial y^j} + \frac{\partial y^i}{\partial x^k} \frac{\partial^2 x^k}{\partial y^\mu \partial y^j} \tilde X^\mu
\end{aligned}$$

Removing the $X$ (more rigorously, choosing $X=\delta^{\mu\rho} \frac{\partial}{\partial y^\rho}$),

$$\begin{aligned}
{\Gamma^{(2) i}}_{j\rho} &= {\Gamma^{(1)k}}_{l\mu} \frac{\partial y^i}{\partial x^k}  \frac{\partial x^\mu}{\partial y^\rho}  \frac{\partial x^l}{\partial y^j} + \frac{\partial y^i}{\partial x^k} \frac{\partial^2 x^k}{\partial y^\rho \partial y^j}
\end{aligned}$$

Not all indices of $\Gamma$ are the same type! 1 of them is spacetime, the other 2 actually label the Lie algebra.

### Memorisation Trick
When we learn this in GR we usually get giddy just trying to write it with the correct indices. However, if we write it as follows and compare with the abstract form, it becomes much clearer what is really going on

$$\begin{aligned}
{\Gamma^{(2) i}}_{j\rho} dy^\rho &= \frac{\partial y^i}{\partial x^k} \left({\Gamma^{(1)k}}_{l\mu}  \frac{\partial x^\mu}{\partial y^\rho} dy^\rho \right) \frac{\partial x^l}{\partial y^j} + \frac{\partial y^i}{\partial x^k} \frac{\partial}{\partial y^\rho } \left(\frac{\partial x^k}{\partial y^j} \right) dy^\rho\\
\mathbf \Gamma^{(2)}_\rho dy^\rho &= \mathbf\Omega^{-1} \left(\mathbf \Gamma^{(1)}_\mu dx^\mu\right) \mathbf \Omega + \mathbf \Omega^{-1} d \mathbf \Omega
\end{aligned}$$

