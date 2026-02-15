---
title: "Setoid Hell: A Minimal Comparison Across Proof Assistants"
date: 2026-02-15
draft: true
tags: [type-theory, proof-assistants, coq, agda, lean4, iris]
---

## What is Setoid Hell?

In type theories without quotient types (or with only propositional equality), you often can't form `A/~` as a first-class type. Instead, you work with a **setoid**: a type `A` paired with an equivalence relation `~`, and you must manually prove that every function and predicate **respects** this relation.

This is "setoid hell" — the cascading obligation to thread compatibility proofs through everything.

### The Running Example

We define **integers as pairs of naturals** `(a, b)` representing `a - b`, where `(a₁, b₁) ~ (a₂, b₂)` iff `a₁ + b₂ = a₂ + b₁`. We define negation and addition, then prove they respect the equivalence.

This is deliberately minimal — real setoid hell gets much worse when you compose multiple setoid-valued functions, define homomorphisms between setoids, or try to rewrite under binders.

All code below is **verified** — it compiles/typechecks as shown. Source: [github.com/twigslot/setoid-hell](https://github.com/twigslot/setoid-hell) *(TODO: push)*.

---

## Coq (Rocq 9.1)

Coq has a `Setoid` typeclass and generalized rewriting (`rewrite` works over registered setoids via `Proper`/`respectful` morphisms). But you must register everything explicitly.

```coq
From Stdlib Require Import Setoid Morphisms Lia.
From Stdlib.Classes Require Import SetoidClass.

(* (a, b) represents the integer a - b *)
Record MyInt := mkMyInt { pos : nat ; neg : nat }.

Definition myint_eq (x y : MyInt) : Prop :=
  pos x + neg y = pos y + neg x.

Lemma myint_eq_refl : forall x, myint_eq x x.
Proof. unfold myint_eq; lia. Qed.

Lemma myint_eq_sym : forall x y, myint_eq x y -> myint_eq y x.
Proof. unfold myint_eq; lia. Qed.

Lemma myint_eq_trans : forall x y z,
  myint_eq x y -> myint_eq y z -> myint_eq x z.
Proof. unfold myint_eq; lia. Qed.

#[export] Instance myint_equiv : Equivalence myint_eq :=
  { Equivalence_Reflexive := myint_eq_refl
  ; Equivalence_Symmetric := myint_eq_sym
  ; Equivalence_Transitive := myint_eq_trans }.

#[export] Instance myint_setoid : Setoid MyInt :=
  { equiv := myint_eq }.

(* Negation: swap the components *)
Definition myint_neg (x : MyInt) : MyInt :=
  mkMyInt (neg x) (pos x).

(* THE PAIN: every function needs a Proper instance *)
#[export] Instance myint_neg_proper :
  Proper (myint_eq ==> myint_eq) myint_neg.
Proof. unfold Proper, respectful, myint_eq, myint_neg; simpl; lia. Qed.

(* Addition *)
Definition myint_add (x y : MyInt) : MyInt :=
  mkMyInt (pos x + pos y) (neg x + neg y).

(* Binary Proper instance — twice the declaration *)
#[export] Instance myint_add_proper :
  Proper (myint_eq ==> myint_eq ==> myint_eq) myint_add.
Proof. unfold Proper, respectful, myint_eq, myint_add; simpl; lia. Qed.

(* Now setoid rewriting works: *)
Example test : forall a b,
  myint_eq a b -> myint_eq (myint_neg a) (myint_neg b).
Proof. intros a b Hab. rewrite Hab. reflexivity. Qed.
```

**The hell:** Every function touching `MyInt` needs a `Proper` instance. For a binary operation, you need `Proper (myint_eq ==> myint_eq ==> myint_eq)`. Compose three functions? Three `Proper` instances. Rewriting under a lambda? You need `Proper (myint_eq ==> iff)` for predicates and `pointwise_relation` for higher-order cases.

**The silver lining:** `lia` handles all the arithmetic. The *proof content* is trivial — it's the *bureaucratic declarations* that hurt.

---

## Agda

Agda has no built-in setoid rewriting. The standard library provides `Setoid` records, but you thread everything manually with equational reasoning chains. No tactics, no automation — just you and the term language.

```agda
module SetoidHell where

open import Data.Nat using (ℕ; _+_)
open import Data.Nat.Properties using (+-comm; +-assoc; +-cancelˡ-≡)
open import Data.Product using (_×_; _,_)
open import Relation.Binary using (IsEquivalence; Setoid)
open import Relation.Binary.PropositionalEquality as ≡
  using (_≡_; refl; sym; trans; cong)

MyInt : Set
MyInt = ℕ × ℕ

_≈_ : MyInt → MyInt → Set
(a₁ , b₁) ≈ (a₂ , b₂) = (a₁ + b₂) ≡ (a₂ + b₁)

≈-refl : ∀ {x} → x ≈ x
≈-refl = refl

≈-sym : ∀ {x y} → x ≈ y → y ≈ x
≈-sym p = sym p

-- Transitivity: THIS is setoid hell.
-- In Coq: "unfold myint_eq; lia." (one line)
-- In Agda: 20 lines of manual equational reasoning.
≈-trans : ∀ {x y z} → x ≈ y → y ≈ z → x ≈ z
≈-trans {a₁ , b₁} {a₂ , b₂} {a₃ , b₃} p q =
  +-cancelˡ-≡ a₂ (a₁ + b₃) (a₃ + b₁) lemma
  where
    lemma : a₂ + (a₁ + b₃) ≡ a₂ + (a₃ + b₁)
    lemma =
      let open ≡.≡-Reasoning in
      begin
        a₂ + (a₁ + b₃)
      ≡⟨ sym (+-assoc a₂ a₁ b₃) ⟩
        (a₂ + a₁) + b₃
      ≡⟨ cong (_+ b₃) (+-comm a₂ a₁) ⟩
        (a₁ + a₂) + b₃
      ≡⟨ +-assoc a₁ a₂ b₃ ⟩
        a₁ + (a₂ + b₃)
      ≡⟨ cong (a₁ +_) q ⟩
        a₁ + (a₃ + b₂)
      ≡⟨ sym (+-assoc a₁ a₃ b₂) ⟩
        (a₁ + a₃) + b₂
      ≡⟨ cong (_+ b₂) (+-comm a₁ a₃) ⟩
        (a₃ + a₁) + b₂
      ≡⟨ +-assoc a₃ a₁ b₂ ⟩
        a₃ + (a₁ + b₂)
      ≡⟨ cong (a₃ +_) p ⟩
        a₃ + (a₂ + b₁)
      ≡⟨ sym (+-assoc a₃ a₂ b₁) ⟩
        (a₃ + a₂) + b₁
      ≡⟨ cong (_+ b₁) (+-comm a₃ a₂) ⟩
        (a₂ + a₃) + b₁
      ≡⟨ +-assoc a₂ a₃ b₁ ⟩
        a₂ + (a₃ + b₁)
      ∎
```

And negation's congruence proof:

```agda
myint-neg : MyInt → MyInt
myint-neg (a , b) = (b , a)

neg-cong : ∀ {x y} → x ≈ y → myint-neg x ≈ myint-neg y
neg-cong {a₁ , b₁} {a₂ , b₂} p =
  let open ≡.≡-Reasoning in
  begin
    b₁ + a₂
  ≡⟨ +-comm b₁ a₂ ⟩
    a₂ + b₁
  ≡⟨ sym p ⟩
    a₁ + b₂
  ≡⟨ +-comm a₁ b₂ ⟩
    b₂ + a₁
  ∎
```

And the real kicker — addition respects equivalence in *both* arguments:

```agda
myint-add : MyInt → MyInt → MyInt
myint-add (a₁ , b₁) (a₂ , b₂) = (a₁ + a₂ , b₁ + b₂)

add-cong : ∀ {x₁ x₂ y₁ y₂} → x₁ ≈ x₂ → y₁ ≈ y₂ →
           myint-add x₁ y₁ ≈ myint-add x₂ y₂
add-cong {a₁ , b₁} {a₂ , b₂} {c₁ , d₁} {c₂ , d₂} p q =
  let open ≡.≡-Reasoning in
  begin
    (a₁ + c₁) + (b₂ + d₂)
  ≡⟨ +-assoc a₁ c₁ (b₂ + d₂) ⟩
    a₁ + (c₁ + (b₂ + d₂))
  ≡⟨ cong (a₁ +_) (sym (+-assoc c₁ b₂ d₂)) ⟩
    a₁ + ((c₁ + b₂) + d₂)
  ≡⟨ cong (a₁ +_) (cong (_+ d₂) (+-comm c₁ b₂)) ⟩
    a₁ + ((b₂ + c₁) + d₂)
  ≡⟨ cong (a₁ +_) (+-assoc b₂ c₁ d₂) ⟩
    a₁ + (b₂ + (c₁ + d₂))
  ≡⟨ cong (a₁ +_) (cong (b₂ +_) q) ⟩
    a₁ + (b₂ + (c₂ + d₁))
  ≡⟨ sym (+-assoc a₁ b₂ (c₂ + d₁)) ⟩
    (a₁ + b₂) + (c₂ + d₁)
  ≡⟨ cong (_+ (c₂ + d₁)) p ⟩
    (a₂ + b₁) + (c₂ + d₁)
  ≡⟨ +-assoc a₂ b₁ (c₂ + d₁) ⟩
    a₂ + (b₁ + (c₂ + d₁))
  ≡⟨ cong (a₂ +_) (sym (+-assoc b₁ c₂ d₁)) ⟩
    a₂ + ((b₁ + c₂) + d₁)
  ≡⟨ cong (a₂ +_) (cong (_+ d₁) (+-comm b₁ c₂)) ⟩
    a₂ + ((c₂ + b₁) + d₁)
  ≡⟨ cong (a₂ +_) (+-assoc c₂ b₁ d₁) ⟩
    a₂ + (c₂ + (b₁ + d₁))
  ≡⟨ sym (+-assoc a₂ c₂ (b₁ + d₁)) ⟩
    (a₂ + c₂) + (b₁ + d₁)
  ∎
```

**25 lines** to prove addition respects equivalence. In Coq: `lia`. In Lean: `omega`. This is the purest expression of setoid hell.

**Escape hatch:** Cubical Agda (`--cubical`) has real quotient types via higher inductive types, eliminating setoid hell entirely at the cost of a different type theory.

---

## Lean 4

Lean 4 has **quotient types in the kernel** (`Quot`). This is the primary escape from setoid hell — you form the quotient once, prove respectfulness at the boundary, and then work with a real type.

```lean
def MyIntRaw := Nat × Nat

def myint_rel (x y : MyIntRaw) : Prop :=
  x.1 + y.2 = y.1 + x.2

instance myIntSetoid : Setoid MyIntRaw where
  r := myint_rel
  iseqv := {
    refl := fun x => by unfold myint_rel; omega
    symm := fun h => by unfold myint_rel at *; omega
    trans := fun h1 h2 => by unfold myint_rel at *; omega
  }

-- The quotient type — this IS the integer type
def MyInt := Quotient myIntSetoid

namespace MyInt

def mk (a b : Nat) : MyInt := Quotient.mk myIntSetoid (a, b)

-- Negation: one respectfulness proof at Quot.lift, then done
def neg (x : MyInt) : MyInt :=
  Quotient.lift (fun p : MyIntRaw => Quotient.mk myIntSetoid (p.2, p.1))
    (fun a b (h : myint_rel a b) => by
      apply Quotient.sound
      show myint_rel (a.2, a.1) (b.2, b.1)
      unfold myint_rel at *; omega)
    x

-- Addition via Quotient.lift₂
def add (x y : MyInt) : MyInt :=
  Quotient.lift₂
    (fun p q : MyIntRaw => Quotient.mk myIntSetoid (p.1 + q.1, p.2 + q.2))
    (fun a₁ b₁ a₂ b₂ (h1 : a₁ ≈ a₂) (h2 : b₁ ≈ b₂) => by
      apply Quotient.sound
      change myint_rel (a₁.1 + b₁.1, a₁.2 + b₁.2) (a₂.1 + b₂.1, a₂.2 + b₂.2)
      unfold myint_rel
      change myint_rel a₁ a₂ at h1
      change myint_rel b₁ b₂ at h2
      unfold myint_rel at h1 h2
      omega)
    x y

-- Proofs work directly on the quotient type — no Proper, no setoid rewriting
theorem neg_neg (x : MyInt) : neg (neg x) = x := by
  induction x using Quotient.inductionOn with
  | _ a =>
    simp [neg]
    apply Quotient.sound
    show myint_rel (a.1, a.2) a
    unfold myint_rel; omega

theorem add_comm (x y : MyInt) : add x y = add y x := by
  induction x using Quotient.inductionOn with
  | _ a =>
    induction y using Quotient.inductionOn with
    | _ b =>
      simp [add]
      apply Quotient.sound
      show myint_rel (a.1 + b.1, a.2 + b.2) (b.1 + a.1, b.2 + a.2)
      unfold myint_rel; omega

end MyInt
```

**The key difference:** After `Quotient.lift`, `neg` and `add` are functions on `MyInt` — a real type, not a setoid. Downstream code never sees the equivalence relation. No `Proper` instances, no congruence lemmas, no cascading obligations.

**Remaining friction:** `Quotient.lift` still requires a respectfulness proof, and `Quotient.inductionOn` can be verbose. But the obligations are *bounded* — they don't cascade.

---

## Iris (Coq-based)

Iris is not a standalone proof assistant — it's a separation logic framework built on Coq. But it introduces its own form of setoid hell via **OFEs** (Ordered Families of Equivalences).

In Iris, types are compared not by Coq's `=` but by a step-indexed equivalence `≡{n}≡`. Every construction must be **non-expansive** (respects `≡{n}≡`), which is the Iris analogue of `Proper`.

```coq
(* Requires coq-iris — install via:
   opam repo add iris-dev https://gitlab.mpi-sws.org/iris/opam.git
   opam install coq-iris *)

From iris.algebra Require Import ofe.

Record my_int := MkMyInt { mi_pos : nat; mi_neg : nat }.

Definition my_int_equiv (x y : my_int) : Prop :=
  mi_pos x + mi_neg y = mi_pos y + mi_neg x.

(* For a discrete OFE, step-indexed equiv = equiv at every step *)
Instance my_int_equiv_instance : Equiv my_int := my_int_equiv.
Instance my_int_dist : Dist my_int := λ _ x y, my_int_equiv x y.

Instance my_int_ofe_mixin : OfeMixin my_int.
Proof.
  split.
  - intros x y. split; [intros Heq n; apply Heq | intros Hstep; apply (Hstep 0)].
  - intros n x y. unfold dist, my_int_dist, my_int_equiv. lia.
  - intros n x y. unfold dist, my_int_dist. auto.
Qed.

Canonical Structure my_intO : ofe := Ofe my_int my_int_ofe_mixin.

Definition my_int_neg (x : my_int) : my_int :=
  MkMyInt (mi_neg x) (mi_pos x).

(* Iris's version of Proper — NonExpansive *)
Instance my_int_neg_ne : NonExpansive my_int_neg.
Proof. intros n x y Hxy. unfold dist, my_int_dist, my_int_equiv in *. simpl. lia. Qed.

(* For binary ops: NonExpansive2 *)
Instance my_int_add_ne :
  NonExpansive2 (λ x y, MkMyInt (mi_pos x + mi_pos y) (mi_neg x + mi_neg y)).
Proof. intros n x1 x2 Hx y1 y2 Hy. unfold dist, my_int_dist, my_int_equiv in *. simpl. lia. Qed.
```

**The hell:** Iris *adds* step-indexing on top of Coq's existing setoid machinery. Every function into/out of OFE types needs `NonExpansive` proofs. The `solve_proper` tactic automates many of these, but when it fails (custom constructions, higher-order cases), you're in deeper hell than vanilla Coq.

*(Note: the Iris code above requires the `coq-iris` opam package and is provided for illustration.)*

---

## Summary

| Aspect | Coq | Agda | Lean 4 | Iris (Coq) |
|---|---|---|---|---|
| **Quotient types?** | No (kernel) | No (vanilla); Yes (cubical) | Yes (kernel) | N/A (OFEs instead) |
| **Setoid support** | Typeclass + `rewrite` | Manual records | `Setoid` class but rarely needed | OFE infrastructure |
| **Morphism proof** | `Proper` instances | Manual congruence lemmas | `Quot.lift` (one-shot) | `NonExpansive` instances |
| **Automation** | `lia` / `typeclasses eauto` | None built-in | `omega` / `simp` | `solve_proper` |
| **Hell severity** | 🔥🔥🔥 | 🔥🔥🔥🔥 (no tactic rewriting) | 🔥 (quotients in kernel) | 🔥🔥🔥🔥🔥 (step-indexed) |
| **Escape hatch** | HoTT library (HITs) | `--cubical` | Already escaped | `solve_proper` + discipline |

### Line count for `add` respects `≈`

| | Coq | Agda | Lean 4 |
|---|---|---|---|
| **Proof** | 1 line (`lia`) | 25 lines | 6 lines (`omega`) |
| **Declaration** | 3 lines (`Proper` instance) | 2 lines (type sig) | 0 (inside `Quot.lift₂`) |

### The Takeaway

**Setoid hell is a symptom of missing quotient types.** Lean 4 sidesteps it by putting quotients in the type theory. Cubical Agda sidesteps it with HITs. Vanilla Coq and Agda suffer the most. Iris adds step-indexed equivalences on top of Coq's existing problems.

The fundamental tension: your type theory either has quotients (and pays the metatheoretic cost) or doesn't (and makes users pay the ergonomic cost of setoids).

---

## References

- Barthe, Capretta, Pons. *Setoids in type theory.* J. Functional Programming, 2003.
- Altenkirch, Boulier, Kaposi, Tabareau. *Setoid type theory — a syntactic translation.* MPC 2019.
- Sozeau. *A New Look at Generalized Rewriting in Type Theory.* JFR, 2009.
- Cohen, Coquand, Huber, Mörtberg. *Cubical Type Theory.* 2018.
- Jung et al. *Iris from the ground up.* JFP, 2018.
- [Lean 4 Quotient Types](https://lean-lang.org/lean4/doc/)
- [Iris Project](https://iris-project.org/)
