---
layout: post
title: "Reviewing Renormalization Resources (in QFT)"
categories: [Physics, QFT, Renormalization]
---
There are many resources on renormalization in Quantum Field Theory (QFT). I have been reading a few of them and I will review them here. This is roughly in the order I recommend reading them (as a complete beginner), with the first few being the best.

Find my recommended reading order below.

Note this blog doesn't cover renormalization group and solid state physics. I will cover that in a separate post. The focus here is QFT.

# Zee QFT 
Anthony Zee's textbook [Quantum Field Theory in a Nutshell](http://home.ustc.edu.cn/~gengb/200923/A.%20Zee,%20Quantum%20Field%20Theory%20in%20a%20Nutshell.pdf) Part III Renormalization has great exposition. I especially like the dialogue between the experimentalist and the theorist. 

It is worth mentioning [Zee's Lectures](https://www.youtube.com/watch?v=_AZdvtf6hPU) although it is abit blurry. 

## Chap III.1: Conceptual

In $\phi^4$ theory, 2 particle scattering up to 1 loop with hard cutoff $\Lambda$:

$$\mathcal{M}(s,t,u,\Lambda,\lambda)=-i \lambda+i C \lambda^2\left[\log \left(\frac{\Lambda^2}{s}\right)+\log \left(\frac{\Lambda^2}{t}\right)+\log \left(\frac{\Lambda^2}{u}\right)\right]+O\left(\lambda^3\right)$$ 

(equation 2)

Zee then explains that we can measure $\mathcal{M}$ at momenta $s_0, t_0, u_0$, and we call that $\lambda_P := \mathcal{M}(s_0,t_0,u_0, \Lambda, \lambda)$. To make sure that $\lambda_P$ is independent of $\Lambda$, Zee says that $\lambda(\Lambda)$ must satisfy 

$$\Lambda \frac{d \lambda}{d \Lambda}=6 C \lambda^2+O\left(\lambda^3\right)$$

(equation 16)

What I especially like about Zee's exposition is that he writes out the full derivation for how to invert $\lambda_P(\lambda)$ to get $\lambda(\lambda_P)$ (equation 7). This is something that is often glossed over in other lectures. Although it is just 2 lines, it is not a common procedure at all, and I think it is important to see it done explicitly the first time. 

$$
\begin{aligned}
\mathcal{M}&=-i \lambda+i C \lambda^2 L+O\left(\lambda^3\right) \\
-i \lambda_P&=-i \lambda+i C \lambda^2 L_0+O\left(\lambda^3\right) \\
-i \lambda&=-i \lambda_P-i C \lambda^2 L_0+O\left(\lambda^3\right)\\
&=-i \lambda_P-i C \lambda_P^2 L_0+O\left(\lambda_P^3\right)
\end{aligned}
$$

(equations 5-7)

We then substitute $\lambda(\lambda_P)$ back into $\mathcal{M}$ to get

$$
\begin{aligned}
\mathcal{M}&=-i \lambda+i C \lambda^2 L+O\left(\lambda^3\right)\\
&=-i \lambda_P-i C \lambda_P^2 L_0+i C \lambda_P^2 L+O\left(\lambda_P^3\right) \\
&= -i \lambda_P+i C \lambda_P^2\left[\log \left(\frac{s_0}{s}\right)+\log \left(\frac{t_0}{t}\right)+\log \left(\frac{u_0}{u}\right)\right]+O\left(\lambda_P^3\right)
\end{aligned}
$$

(equation 9)

As Nabil Iqbal puts it, "you want to relate measureable quantities to the measureable quantities", like pressure to temperature. 

Zee then covers Pauli-Villars regularization, and dimensional regularization in the Appendix of Chapter III.1.

## Chap III.2: Renormalizable vs Non-renormalizable (using Dimensional Analysis)
Zee covers it quite well. I especially like his example of a vector boson theory being the UV completion of Fermi theory (equation 1, page 157).

## Chap III.3: Counterterms and Physical Perturbation Theory
I'll be honest I don't really like his explanation of mass renormalization that much. 

## Chap III.7: Polarizing the Vacuum and Renormalizing the Charge
No comments

# Nabil Iqbal Lectures
[Nabil Iqbal Lectures](https://youtu.be/UNVV2suXsEA) from [QFT2 at Durham](https://www.youtube.com/playlist?list=PL_kQfq_5BKqq1xPQe3LHeAuAoLyCMZx4Y).

## Lecture 3d: Intro to Renormalization
He mentions the problem of infinity if we take the limit $\Lambda \rightarrow \infty$ in the $\phi^4$ theory. 

[2:59](https://youtu.be/UNVV2suXsEA?t=2m59s) I especially like his analogy of ideal gas law and how we relate 2 observable things to each other.

[7:10](https://youtu.be/UNVV2suXsEA?t=7m10s) I also like that he explicitly says that $\lambda_P \equiv i\mathcal{M}(s_0,t_0,u_0)$ is a definition. In Zee's book, he says $\lambda_P$ is measured, but doesn't explicitly say that it is definitionally equal to $\mathcal{M}(s_0,t_0,u_0)$, and it must instead be inferred by looking at the equations 3-4.

[10:20](https://youtu.be/UNVV2suXsEA?t=10m20s) He inverts $\lambda_P(\lambda)$ to get $\lambda(\lambda_P)$, but leaves it to the viewer to verify. I prefer Zee for the explicit working in this regard.

## Lecture 4a: Renormalized Perturbation Theory
To be honest, I don't like this lecture that much. He explains the renormalization conditions (pole of 2-point function at $p^2 = m^2$ must have residue 1). 

He derives the same result

$$i \mathcal{M}\left(s, t, u\right)=-i \lambda+\frac{i \lambda^2}{32 \pi^2} \left[ \log \left(\frac{s_0}{s}\right)+\log \left(\frac{t_0}{t}\right)+\log \left(\frac{u_0}{u}\right) \right]$$

(equation at [18:12](https://www.youtube.com/watch?v=Mqrv22DExX8&t=18m12s)) 

using counterterms and renormalization conditions. 

## Lecture 4b: Renormalization of the two-point function
No comments
## Lecture 4c: Which Theories Are Renormalizable
No comments
## Lecture 4d: non-renormalizable theories
I like the examples he introduced.

# Tobias Osborne Lectures
[Tobias Osborne Lectures](https://youtu.be/J563CtQWpaY) from his [Advanced QFT Series](https://www.youtube.com/playlist?list=PLDfPUNusx1ErSu1JDVV1KKGQkJQCkzL9u). Tobias approaches renormalization in a very philosophical manner. He covers $\phi^4$ theory. 

## Lecture 8: Philosophy of Renormalization
He spends a good 40 minutes talking about philosophy of physics. I especially like [56:00](https://www.youtube.com/watch?v=J563CtQWpaY&t=56m0s) where he says we can declare victory even if the (bare) parameters of our theory are not necessarily observables, and they can totally depend on a cut-off $\Lambda$, and he labels them $z_i(\Lambda)$ to emphasise it. My takeaway is that the unrenormalized/bare parameters are merely a means to an end, and the end is to predict observables. 

## Lecture 9: Renormalization of $\phi^4$ theory
No comments

# Lancaster QFT
[Quantum Field Theory for the Gifted Amateur](https://www.amazon.com/Quantum-Field-Theory-Gifted-Amateur/dp/019969933X) Part VIII Renormalization. He introduces renormalization using counterterms first, then explaining the philosophy of "what is physically observable", mirroring the historical development of renormalization. 

I think the historical way is not a good way to learn it though. I much rather learn it from the modern perspective of effective field theory first, and then learn the historical development. This is because physicists were confused about renormalization for a long time, before a drastic change in perspective made it clearer. It would thus make more sense to use the benefit of hindsight when teaching renormalization.

## Chap 31: Quasiparticles
I don't really like the explanations here.

## Chap 32: Renormalization
He works with $\phi^4$ theory. While I like that he covers a lot of detailed mathematical steps, I don't really like the explanations here. This is a useful resource if you are already familiar with the conceptual understanding and wish to revise it.

## Chap 33: Propagators and Feynman Diagrams
I somewhat like his derivation of mass renormalization, as well as renormalization conditions for counterterms.

## Chap 34 & 35: Renormalization Group
I like his use of examples, especially examples in solid state physics.

# == Detailed Calculations == 
From here onwards, the resources are more detailed and less conceptual. They are good if you already know the conceptual idea and want to apply it to real calculations.

# Lewis Ryder QFT
Lewis Ryder's textbook [Quantum Field Theory](https://www.amazon.com/Quantum-Field-Theory-Lewis-Ryder/dp/0521478146) writes mostly about renormalization of gauge theories in Chapter 9. This is good if you already have a decent conceptual understanding of renormalization and want to see how it works in phenomenology.

# Ricardo D. Matheus Lectures
[QFT2](https://www.youtube.com/playlist?list=PL5-Gs_CjccK51MURXy4_Fgy9ihSul3-h5) from IFT UNESP (Sao Paulo) is a great series if you want to dive into QED and QCD calculations in gory detail. I especially like his [Kallen-Lehmann Lecture](https://www.youtube.com/watch?v=hiROq3bAr2g&list=PL5-Gs_CjccK51MURXy4_Fgy9ihSul3-h5&index=2) and his [LSZ Lecture](https://www.youtube.com/watch?v=tWezNBXXZfk&list=PL5-Gs_CjccK51MURXy4_Fgy9ihSul3-h5&index=4). 

# wky54321 Lecture (Summary)
[Renormalization of QED](https://youtu.be/NZkX0uT8gtM) is a standalone lecture summarising QED calculations. I think the first half is good for beginners, but the back half is hard to follow if you didn't already know the material.

# Francois David Lectures
[Francois David Lectures](https://youtu.be/9ggGqyvwzSY) from [QFT2 at Saclay](https://pirsa.org/C22037). 

# Ashoke Sen Lectures
[Ashoke Sen Lectures](https://youtu.be/1Qsf1ARZuo0) from [QFT2](https://www.youtube.com/playlist?list=PLfHqWfSignon_EFyk5Wuw6glhJi5FtPas)

# == Recommended Reading Order ==
If you are completely new to renormalization, I recommend the following resources in this order:

#### Conceptual Understanding / Vertex Renormalization
- Zee QFT Chapter III.1 (Conceptual)
- Zee QFT Chapter III.2 (Dimensional Analysis)
- Nabil Iqbal Lecture 3d (Intro to Renormalization)
- Tobias Osborne Lecture 8 (Philosophy of Renormalization)
- Tobias Osborne Lecture 9 (Dimensional Analysis)

#### Counterterms and Renormalization Conditions
- Lancaster QFT Chap 32 (Renormalization)

#### Mass Renormalization
- idk, I don't like any of the explanations so far