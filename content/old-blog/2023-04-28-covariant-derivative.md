---
layout: post
title: Koszul Connection and Covariant Derivative
categories: [Math, Physics, QFT, Gauge Theory]
---

## Good Resources
- Wiki: [associated bundle](https://en.wikipedia.org/wiki/Associated_bundle), [Koszul connection](https://en.wikipedia.org/wiki/Connection_(vector_bundle))
- Frederic Schuller [Lectures 19-25](https://www.youtube.com/playlist?list=PLPH7f_7ZlzxTi6kS4vCmv4ZKm9u8g5yic), specifically 23 for this blog
- [Nakahara]((https://www.amazon.com/Geometry-Topology-Physics-Graduate-Student/dp/0750306068)) Chapter 10.4

We mostly follow the notation in Schuller [Lecture 25](https://youtu.be/ClIVG7ilm_Q).

As part 2 of my series building towards "Twisting Supersymmetry" (see [part 1 here](https://tch1001.github.io/math/physics/qft/gauge%20theory/2023/04/27/ehresmann.html)), I will talk about connections on associated bundles induced by connections on principal bundle. We actually see this very often: $U(1)$ gauge theory (electromagnetism) will often have a scalar field (scalar QED) or a Dirac spinor field (QED) charged under $U(1)$. If one inspects the covariant derivative in scalar QED, it actually arises from the connection on the complex line bundle ($\mathbb C$-bundle) induced by the connection on the $U(1)$-principal bundle. And for QED, it is the connection on the $K \oplus \bar K$ vector bundle (Dirac spinor bundle) induced by the connection on the $U(1)$-principal bundle.

## Associated Bundle
Loosely speaking, an associated bundle (associated to some $G$-principal bundle) is a way to attach to each point in $x\in M$ a fiber $\pi^{-1} x \cong F $ homeomorphic to a topological space $F$ on which $G$ acts on. This fiber could be a Lie group $G$ itself (so a $G$-principal bundle is technically associated to itself), but this definition also allows for vector spaces, which means we can have representations $(\rho, V)$ of $G$ as fibers too. 

The formal definition ([wiki](https://en.wikipedia.org/wiki/Associated_bundle)) is considering the direct product modulo an equivalence relation 

$$
\begin{aligned}
& P_V := P\times_\rho V := (P \times V) / \sim  \\
& (p, v) \sim (p \bullet_R g , g^{-1} \bullet_L v) 
\end{aligned}
$$

Intuition: it may initially seem weird that we are considering all pairs, but it turns out this equivalence relation basically collapses the entire $G$ fiber of $P$ to a single point in the base manifold $M$. So the only "degree of freedom" left is the $V$. So effectively we are attaching a copy of $V$ to every point in the base manifold $M$.

## Covariant Derivative of Associated Bundle Section (Teaser)
This is the end goal: given a vector $X\in T_p M$ and a section in a vector bundle $\sigma: M \rightarrow P_V$ (we need vector bundle structure because otherwise subtraction won't make sense in the fiber), we want to rigorously calculate the **covariant (directional) derivative of $\sigma$ wrt $X$ direction**

$$\nabla_X \sigma \vert_p \in \pi^{-1}(p) \cong V$$

We **want** this covariant derivative to satisfy various properties, which we can check later once we have defined it. For $f \in C^{\infty} (M),\ T,S \in T M$

1. $$\nabla_{fT+S} \sigma = f \nabla_T \sigma+ \nabla_S \sigma$$
2. $$\nabla_T \sigma_1 + \sigma_2 = \nabla_T \sigma_1 + \nabla_T \sigma_2$$
3. $$\nabla_T (f \sigma) = T(f) \sigma + f \nabla_T \sigma $$

The definition that we work towards is the following: for a *local* section $$s:\mathcal U \subset M \rightarrow P_V$$, a *local* trivialisation $$\varphi : \mathcal U \subset M \rightarrow P$$, the exterior covariant derivative on $P$, and a $V$-valued function $\phi:P\rightarrow V$, a $\mathfrak g$-valued connection one-form $\omega:P \rightarrow \mathfrak g$

$$\nabla_T s := (\varphi^* D \phi) (T) = ds(T) + (\varphi^* \omega)(T) \bullet_{d\rho} s$$

It is a definition dump for now but let's explain the details.

## Exterior Covariant Derivative on $P$
Definition: For an any-valued $n$-form $f$ on $\Omega^n(P)$,

$$Df := df \circ \text{hor}$$

Special case 1: if $\omega$ is $\mathfrak g$-valued connection one-form ($n=1$), for any $X,Y\in TP$,

$$D\omega(X,Y) := d\omega (\text{hor}(X), \text{hor}(Y))$$

Special case 2: if $\phi$ is a $F$-valued function on $P$ ($n=0$), for any $X\in TP$,

$$D\phi(X) := d\phi(\text{hor}(X))$$

## Correspondence between $F$-valued functions on $P$ & global sections of associated bundle $P_F$
One can define the covariant derivative using parallel transport on the associated vector bundle $P_V$ but that definition is difficult to work with. Instead we seek to define things by using the principal $G$-bundle $P$ and defining objects on it, then pulling them over to the associated bundle $P_V$.

We are going to work with a general fiber $F$ (not necessarily a vector bundle).

The first thing to establish is the one-to-one correspondence between (global) sections $\sigma$ on $P_V$ and $G$-equivariance $F$-valued functions $\phi$ on $P$. $G$-equivariance means $$\phi(p \bullet_R g) = g^{-1} \bullet_L \phi(p)$$

The correspondence from $\sigma$ to $\phi_\sigma$ is 

$$
\begin{aligned}
\phi_\sigma : P & \rightarrow F \\
p & \mapsto i^{-1}_p (\sigma (\pi(p)))\\
\text{where } i_p : F & \rightarrow P_F \\
f & \mapsto [(p, f)]
\end{aligned}
$$

One can check (watch Schuller Lec 25) that this $\phi$ is $G$-equivariant

The other direction from $\phi$ to $\sigma_\phi$ is 

$$
\begin{aligned}
\sigma_\phi: M & \rightarrow P_F\\
x & \mapsto [(p, \phi(p))] \text{ for any }p 
\end{aligned}
$$

One can check (watch Lec 25) that $$\phi_{\sigma_\phi} = \phi$$ and $$\sigma_{\phi_\sigma} = \sigma$$ 

## Exterior Covariant Derivative of $\phi:P\rightarrow V$
**Only if** we set the fiber $F$ to be some finite dimensional vector space $V$ (isomorphic to $\mathbb R^{\text{dim }V}$), a (left) representation of $G$, then the $G$-equivariance condition 

$$\phi(p\bullet_R \exp(At)) = \exp(-At) \bullet_\rho \phi(p) \quad \forall t\in \mathbb{R},A\in \mathfrak g$$

can be differentiated (because the vector space has an addition). So fixing $A\in \mathfrak g$ and differentiating wrt $t$,

$$ 
\begin{aligned}
\left. \frac{d}{dt}\phi(p\bullet_R \exp(At)) \right\vert_{t=0}  & = \left. \frac{d}{dt} \exp(-At) \bullet_{\rho} \phi(p) \right\vert_{t=0} \\
d\phi(X_p^{A}) & = -A \bullet_{d\rho} \phi(p) \in T_{\phi(p)} V
\end{aligned}
$$

where $d\phi : T_p P \to T_{\phi(p)} V$ is the pushforward of $\phi: P \to V$. The $\bullet_{d\rho}:\mathfrak g \times T_v V \rightarrow T_v V$ action is the [pushforward of the Lie group representation](https://en.wikipedia.org/wiki/Lie_algebra_representation#Infinitesimal_Lie_group_representations). Actually to be more precise it is the restriction of the pushforward to the tangent space at the identity of the group. To spell it out, we take the pushforward of $\rho: G \times V \to V$, namely $d\rho: T_g G \times T_v V \to T_{\rho(g, v)} V$ and restrict it to the subspace $T_g G \supset T_e G \cong \mathfrak g$. So the codomain becomes $T_{\rho(e,v)}V = T_{v} V$. 

If we had a connection on $P$, then $\omega(X^{A}) = A$ so we can rewrite the above as

$$d\phi(X^{A}_p ) + \omega(X^{A}_p)\bullet_{d\rho} \phi(p) = 0$$

Now using the above, we can show that $D\phi := d\phi \circ \text{hor}$ is equals to 

$$D\phi(X_p) = d\phi(X_p) + \omega(X_p) \bullet_{d\rho} \phi(p)$$

by decomposing $X=H+V$ and showing that the above holds for $H,V$ separately. 

## Pulling back $D\phi(X)$ to $$\nabla_{\pi_* X}\sigma$$

We have a good expression for $D\phi : TP \rightarrow \Gamma(P_V)$ (sections of the $P_V$ associated bundle), or pointwise a $V$-valued one-form $D\phi_p: T_p P \rightarrow V$. Suppose we have a local section $\varphi : \mathcal U \rightarrow P$, then we can pullback the $D\phi$ one-form to a one-form on the base manifold $\mathcal U \subset M$. For a vector $T\in  T_x \mathcal U$

$$
\begin{aligned}
\underbrace{(\varphi^{*} D\phi) (T)}_{:=} &= (\varphi^{*} d\phi) T + (\varphi^{*}\omega)(T) \bullet_{d\rho} (\varphi^* \phi) (x)\\
\nabla_T (\phi \circ \varphi) &= d(\phi \circ \varphi) T + \Gamma(T) \bullet_{d\rho} \phi(\varphi (x))\\
\nabla_T s&= ds (T) + \Gamma(T) \bullet_{d\rho} s(x)
\end{aligned}
$$

where we $\phi \circ \varphi \equiv s : \mathcal U \rightarrow V$. The last line is just the usual definition we see for covariant derivative! 

One can check that all the properties below hold true given the above definition.

For $f \in C^{\infty} (M),\ T,S \in T M$

1. $$\nabla_{fT+S} \sigma = f \nabla_T \sigma+ \nabla_S \sigma$$
2. $$\nabla_T \sigma_1 + \sigma_2 = \nabla_T \sigma_1 + \nabla_T \sigma_2$$
3. $$\nabla_T (f \sigma) = T(f) \sigma + f \nabla_T \sigma $$

## Examples
### General Relativity
In General Relativity, the principal bundle is the frame bundle $LM$, with gauge group $GL(d)$. The associated bundle is the tangent bundle $TM$. Since $GL(d)$ is a matrix group, the action $\bullet_{d\rho}$ is just matrix multiplication. The section $s:\mathcal U \rightarrow TM$ is a vector field. If there is a coordinate chart $x:\mathcal U \rightarrow \mathbb R ^ d$, then the section can be decomposed as $s^j (x) \partial_j$ where $s^j :\mathcal U \rightarrow \mathbb R$ are functions on $\mathcal U$.

$$\nabla_T s= ds (T) + \Gamma(T) \bullet_{d\rho} s(x)$$ 

becomes

$$
\begin{aligned}
(\nabla_{\partial_\mu} s)^i & = [ds(\partial_\mu)]^i + {\Gamma^i}_{j\mu} s^j \\ 
& = \partial_\mu (s^i) + {\Gamma^i}_{j\mu} s^j
\end{aligned}
$$

To unpack what we just wrote above, note that $\mathbf \Gamma_\mu$ is a matrix. If we represent $\mathbf s = s^j \partial_j$ as a column vector with entries $s^j$, then the action of $\mathbf \Gamma_\mu$ on $\mathbf s$ is just matrix multiplication $(\mathbf \Gamma_\mu \mathbf s)^i = {\Gamma^i}_{j\mu} s^j$ 

We have used property (1) to decompose $\nabla_T$ into 

$$\nabla_{T^\mu \partial_\mu} = T^\mu \nabla_{\partial_\mu} $$ 


On the LHS, $$ \nabla_{\partial_\mu} \mathbf s$$ is a vector in $T_xM$ so we can extract components $$(\nabla_{\partial_\mu} \mathbf s)^i$$. Do note that this is NOT the same as $$\nabla_{\partial_\mu} (\mathbf s^i)$$. The latter is interpreting $s^i$ as functions, and the associated bundle would be the line bundle $\mathbb R$. In our case we are dealing with an associated vector bundle, and elements living in the fiber are $\mathbf s(x) \in T_x M$.

The $\partial_\mu(s^i)$ term is admittedly very ambiguous (my fault). We start from the definition of the pushforward

$$ds(\partial_\mu) := \partial_\mu (\_ \circ s)$$

The function on $\mathcal U$, $$\_ \circ s: \mathcal U \to \mathbb R$$, is eaten by $$\partial_\mu \in T_x \mathcal U$$ to yield a real number. But this $ds(\partial_\mu)$ has a blank slot $$\_$$, and is waiting for a function on $TM$.

The $ds(\partial_\mu)$ eats a function on $TM$ and outputs a real number. In other words, it is a vector **on** $TM$ (meaning it lives **in** $TTM$, yea!). $$\nabla_{\partial_\mu} s$$ on the LHS is a vector on $TM$ too. All that remains to settle are the components: to show that 

$$[ds(\partial_\mu)]^i = [\partial_\mu(\_ \circ s)]^i \stackrel{!}{=} \partial_\mu (s^i)$$

Actually this doesn't make sense, because $i$ goes from $1$ to $d$ on the RHS, whereas the dimension of the $ds(\partial_\mu)$ vector is $2d$. The resolution to this is to note that $$ds(\partial_\mu)$$ is a horizontal vector since $$d(\pi \circ s) = \text{id}_{TM}$$, so it has effective dimension $d$. Since $d$ is a linear functional, $(ds)^i = d(s^i)$. Then $d(s^i) \partial_\mu$ is just $\partial_\mu(s^i)$ by definition. QED

I'm sorry to leave you on the edge but I do not know how to make the above argument more rigorous. It matches with the GR definition of covariant derivative so I'm sure there is some valid explanation. I think it has to do with the isomorphism $T_v V \cong V$ but I can't make this precise. Might update the blog next time (TODO).

### [Scalar Electrodynamics](https://en.wikipedia.org/wiki/Scalar_electrodynamics)
We switch to the convention that the $\exp$ map from Lie algebra to Lie group has a factor of $i$, so $\exp(i \mathfrak g) = G$. This is so that the Lie algebra is real to match physics convention.

The section $s:M \to \mathbb C$ is the complex line bundle. The action $\rho:G \to GL(\mathbb C)$ is given by complex rotation $\exp{(i \lambda)} \bullet_\rho \phi = \exp{(i\lambda)} \phi$. The pushforward of that action is $\bullet_{d\rho}: \mathfrak u(1) \rightarrow GL(\mathbb C)$, given by complex multiplication ${\lambda} \bullet_{d\rho} \phi = i \lambda \phi$. $\Gamma$ is the electromagnetic 4-potential $A = A_\mu dx^\mu$, with $A_\mu \in \mathbb R \cong \mathfrak g$.

$$\nabla_T s= ds (T) + \Gamma(T) \bullet_{d\rho} s(x)$$ 

becomes

$$
\begin{aligned}
D_{\partial_\mu}\phi & = d\phi(\partial_\mu) + A_\mu \bullet_{d\rho} \phi\\
& = \partial_\mu \phi + i A_\mu \phi\\
\end{aligned}
$$

If we had chosen the action to be $\exp(i \lambda) \bullet_\rho \phi = \exp(-i\lambda) \phi$ then we would have arrived at $- i A_\mu \phi$ instead of $+$ (a more common convention). It is just a matter of convention / choice of action.

The Dirac Electrodynamics version is not too different since $U(1)$ just acts on individual components (scalar multiplication as opposed to matrix multiplication) of the Dirac spinor.

## Gauge Transformations
You hear this in GR: "the covariant derivative transforms like a tensor". You also see things like $D_\mu' U(x) = U(x) D_\mu$ in electrodynamics. It turns out this is unified under the following

$$
\begin{aligned}
\nabla' \Omega^{-1} s &= d(\Omega^{-1} s) + (\Omega^{-1} \Gamma \Omega + \Omega^{-1} d \Omega) \Omega^{-1} s \\
& = d\Omega^{-1} s + \Omega^{-1} ds + \Omega^{-1}\Gamma s + \Omega^{-1} d \Omega \Omega^{-1} s \\
& = d\Omega^{-1} s + \Omega^{-1} ds + \Omega^{-1}\Gamma s - d\Omega^{-1} s \\
& = \Omega^{-1} (ds + \Gamma s) \\
& = \Omega^{-1} \nabla s
\end{aligned}
$$

where we used $d\Omega^{-1} \Omega + \Omega^{-1} d \Omega  = d(\Omega^{-1} \Omega) = 0$. 

I find it slightly odd that $s$ transforms under $\Omega^{-1}$ (from the left). I could have made a mistake, or maybe it's just interpreted as a right action (left vs right differs under composition of actions).