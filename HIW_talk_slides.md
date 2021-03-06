---
author: Sam Derbyshire, Tweag
title: Rewriting type families with GHC type-checking plugins
subtitle: HIW 2021
date: August 22nd, 2021
---

## Introduction

<!--
Compile the HTML slides with Pandoc:

pandoc -t revealjs -s -o index.html HIW_talk_slides.md -V revealjs-url=reveal.js -V theme=tweag_theme -V controlsTutorial=false -V controlsLayout=edges -V slideNumber='c/t' -V transition=slide

Custom theme in reveal.js/dist/themes/tweag_theme.css
-->

::: notes

Hi everyone!

This summer, I've been working on GHC with Richard Eisenberg.

I've made some improvements to type-checking plugins which I'd like to share.

  - native support for rewriting type families,
  - improvements to the API, with the `ghc-tcplugin-api` library.

:::

## Talk overview
<br />

- When to reach for a type-checking plugin?
- Lightning fast review of type-checker plugins.
- New functionality: rewriting type family applications.
- Illustration: intrinsically typed System F.
- Presentation of a library for writing type-checking plugins.

## When to reach for a type-checking plugin?

::: notes
Solving typeclasses instances in a custom way (SMT solver, graphs,...).
Type family reduction (focus of this talk).
:::

## Equational theory: Peano arithmetic

<br />

GHC doesn't understand basic arithmetic.

:::{.element: class="fragment"}
```haskell

type (+) :: Nat -> Nat -> Nat
type family a + b where
  Zero   + b = b
  Succ a + b = Succ (a + b)
```
:::

:::{.element: class="fragment"}
GHC can only reduce `a + b` when it knows what `a` is:  
is it `Zero` or is it `Succ i` for some `i`?
:::

<br />

:::{.element: class="fragment"}
Can't reason about `(+)` parametrically:

```
Could not deduce n + Succ n ~ Succ (n + n)
```
:::

::: notes

Christiaan Baaij's `ghc-typelits-natnormalise` type-checker plugin can handle this.

:::

## Equational theory: row types

<br />

:::{.element: class="fragment"}
**Row**: unordered association map, field name ⇝ type, e.g.  
`( "field1" :: Int, "field2" :: Bool )`.  
Can be used to model extensible records (order doesn't matter).
:::

<br />

:::{.element: class="fragment"}
To account for re-ordering, we need:

```haskell
Insert k v (Insert l w r) ~ Insert l w (Insert k v r)
```

when `k` and `l` are distinct field names that don't appear in the row `r`.
:::

:::{.element: class="fragment"}
Problem: in GHC, we can't have type-families on the LHS of a type family equation:
:::

:::{.element: class="fragment"}
```
• Illegal type synonym family application ‘Insert l w r’ in instance:
    Insert k v (Insert l w r)
```
:::

## Non-confluence

<br />

:::{.element: class="fragment"}
GHC can't allow type-families on the LHS of a type family equation in general:
no guarantee type-family reduction is confluent.
:::

:::{.element: class="fragment"}
```haskell

type F :: Type -> Type
type family F a where
  F (F a) = a       -- F[0]
  F Bool  = Float   -- F[1]
  F a     = Maybe a -- F[2]
```
:::

:::{.element: class="fragment"}
```haskell

F (F Bool) ~F[0]~> Bool
F (F Bool) ~(F F[1])~> F Float ~F[2]~> Maybe Float
```

:::

::: notes

The F[i] notation just means the i-th type-family equation
(branched coercion axiom), counting from 0.

Confluence means that the result of rewriting doesn't depend on the order
in which we perform reductions: diamond property.

Authors of type-checking plugins should be careful not to introduce non-confluence.

:::

## Type-checking plugins to the rescue

:::{.element: class="fragment"}
<p align="center">
<img src=GHC_Tc_state_machine_HIW_1.svg height="550px" />
</p>
:::

::: notes

We can hook into GHC's typechecker with typechecking plugins.

LHS: GHC typechecker state machine. Start, type-checking loop, stop.

RHS: type-checking plugin (TcPlugin datatype we are about to review).

Review the existential `s` environment/state; plugin authors will already be familiar with this.
Initialisation gives us `s`, which we pass to the different stages (we only do this once).
This initialises the various stages so that they are ready to be called by GHC.

:::

## Rewriter plugins

:::{.element: class="fragment"}
```haskell

data TcPlugin = forall s. TcPlugin
  { tcPluginInit    :: TcPluginM s
  , tcPluginSolve   :: s -> TcPluginSolver
  , tcPluginRewrite :: s -> UniqFM TyCon TcPluginRewriter -- new!
  , tcPluginStop    :: s -> TcPluginM ()
  }
```
:::

:::{.element: class="fragment"}
```haskell

type TcPluginRewriter
  =  [Ct]   -- Givens
  -> [Type] -- Type family arguments
  -> TcPluginM TcPluginRewriteResult

data TcPluginRewriteResult
  = TcPluginNoRewrite
  | TcPluginRewriteTo
    { tcPluginReduction    :: !Reduction
    , tcRewriterNewWanteds :: [Ct]
    }

data Reduction = Reduction Coercion !Type
```
:::

::: notes

We supply rewriters for each type-family with a map: `UniqFM TyCon TcPluginRewriter`.

`Reduction`: the stored coercion is left-to-right.  
Plugin authors can pass a `UnivCo (PluginProv ...) lhs rhs` ("just believe me") as usual.

Can emit new constraints: useful for custom type errors. This is quite different
from how GHC usually works: it never emits constraints when rewriting.

:::

##

<p align="center">
<img src=GHC_Tc_state_machine_HIW_2.svg height="550px" />
</p>

::: notes

What calls `tcPluginRewrite`? Mainly constraint canonicalisation,
but also type normalisation in the pattern match checker.

:::

## Stephanie Weirich's challenge

<br />

Stephanie Weirich posed the challenge of implementing
a strongly-typed representation of System F.
[[Talk]](https://www.youtube.com/watch?v=j2xYSxMkXeQ)
[[GitHub]](https://github.com/sweirich/challenge/)

:::{.element: class="fragment"}
```haskell

-- | A term, in the given context, of the given type.
data Term ϕ a where
  -- | Function application.
  (:$) :: Term ϕ (a :-> b) -> Term ϕ a -> Term ϕ b
  -- | Type application.
  (:@) :: Term ϕ (Forall ty) -> Sing a -> Term ϕ (SubOne a ty)
  -- Elided: variables, literals, Lambda abstraction, Big Lambda.
```
:::

::: notes

Just giving the flavour for now; full code linked at the end.

Standard System F AST, except we annotate terms
by a context and a type, to ensure things are well-scoped and
well-typed.

Uses de Bruijn indices, but has two different successors:
for kind and type extension, respectively.

:::

## Type-level substitution

:::{.element: class="fragment"}
```haskell

type ApplySub ::
    forall (kϕ :: KContext) (kψ :: KContext) (k :: Kind).
    Sub kϕ kψ -> Type kϕ k -> Type kψ k
```
:::

```haskell
-- | Apply a substitution to a type.
type family ApplySub s ty where {..}

-- | Substitute a single type.
type SubOne a b = ApplySub ('Extend 'Id a) b
```

::: notes

Explain that we have a declarative language of renamings and substitutions:

  - identity
  - composition
  - extend a composition with a type
  - rename when creating a new binder
  - substitute under a binder
:::

## Problem

```haskell

ApplySub s (ApplySub t a) ~ ApplySub ('Compose s t) a
```

:::{.element: class="fragment"}
Type family argument on the LHS of a type family equation.
:::

<br />

:::{.element: class="fragment"}
Might also need to use Givens:

```haskell
[G] ApplySub t a ~ f

ApplySub s f
  ~~>
  ApplySub ('Compose s t) a

```
:::

## Rewriting plugin

:::{.element: class="fragment"}
```haskell

pluginRewrite :: PluginDefs -> UniqFM TyCon TcPluginRewriter
pluginRewrite defs@( PluginDefs { applySubTyCon } ) =
  listToUFM
    [ ( applySubTyCon, rewriteApplySub defs ) ]

rewriteApplySub
  :: PluginDefs
  -> [ Ct ]     -- Givens
  -> [ Type ]   -- Arguments to ApplySub
  -> TcPluginM TcPluginRewriteResult
rewriteApplySub defs givens [ kϕ, kψ, k, sub, sub_arg ]
  = ...
```
:::

::: notes

PluginDefs is this plugin's choice of `s` (from before).

Review UniqFM: a map from a type-family TyCon to its rewriting function.

Extra arguments: invisible arguments. They correspond to the forall in the
standalone kind signature of `ApplySub`.

Take a moment to recall how it used to be done: have to traverse
all the constraints to find type-family applications, and then
figure out how to rewrite each overall constraint over an inner
type family reduction.

:::


##

```haskell

rewriteApplySub defs givens [ kϕ, kψ, k, sub, sub_arg ]
  = ...
```

:::{.element: class="fragment"}
```haskell

[G] ApplySub kϕ0 kϕ l t a ~ sub_arg

rewriteApplySub defs givens [ kϕ, kψ, k, sub, sub_arg ]
  ~~>
  TcPluginRewriteTo
    { tcPluginReduction =
        mkPluginNomRedn "ApplySub s (ApplySub t a)" $
          mkTyConApp composeTyCon [kϕ0, kϕ, kψ, sub, t]
    , tcRewriterNewWanteds = [] }
```
:::

::: notes

Note that this is all totally untyped: kϕ :: Type, k :: Type, sub :: Type...
Easy to mix up arguments (or pass the wrong amount).

Main tools for debugging: Core Lint, tc-trace.
Explained in more detail in my library, and upcoming blog post.

:::


## Type-checking plugins in practice?

:::{.element: class="fragment"}
<p align="center">
<svg
   version="1.1"
   id="svg56507"
   xml:space="preserve"
   width="639.56335"
   height="526.78101"
   viewBox="0 0 639.56335 526.78101"
   sodipodi:docname="tcplugins_cpp.svg"
   inkscape:version="1.1 (c68e22c387, 2021-05-23)"
   xmlns:inkscape="http://www.inkscape.org/namespaces/inkscape"
   xmlns:sodipodi="http://sodipodi.sourceforge.net/DTD/sodipodi-0.dtd"
   xmlns="http://www.w3.org/2000/svg"
   xmlns:svg="http://www.w3.org/2000/svg"><sodipodi:namedview
     id="namedview367"
     pagecolor="#505050"
     bordercolor="#ffffff"
     borderopacity="1"
     inkscape:pageshadow="0"
     inkscape:pageopacity="0"
     inkscape:pagecheckerboard="1"
     showgrid="false"
     inkscape:zoom="0.90509668"
     inkscape:cx="348.58155"
     inkscape:cy="292.23398"
     inkscape:window-width="2560"
     inkscape:window-height="1377"
     inkscape:window-x="-8"
     inkscape:window-y="464"
     inkscape:window-maximized="1"
     inkscape:current-layer="g56513"
     fit-margin-top="0"
     fit-margin-left="0"
     fit-margin-right="0"
     fit-margin-bottom="0" /><defs
     id="defs56511" /><g
     id="g56513"
     transform="translate(175.58246,-277.19226)"><rect
       style="fill:#ececec;fill-opacity:1;stroke:none;stroke-width:15.0002;stroke-linecap:round;stroke-linejoin:bevel;stroke-dashoffset:7.2;paint-order:fill markers stroke"
       id="rect87717"
       width="639.56335"
       height="526.78101"
       x="-175.58246"
       y="277.19226" /><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#833f7b;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text56561"
       x="99.361618"
       y="73.600929"><tspan
         x="99.361618 102.23846"
         y="73.600929"
         id="tspan56559"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text56753"
       x="161.44299"
       y="149.92166"><tspan
         x="161.44299 164.31984"
         y="149.92166"
         id="tspan56751"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text56825"
       x="227.84351"
       y="161.60213"><tspan
         x="227.84351 230.85954"
         y="161.60213"
         id="tspan56823"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text56939"
       x="187.52254"
       y="185.12219"><tspan
         x="187.52254 190.39938"
         y="185.12219"
         id="tspan56937"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text56963"
       x="253.92303"
       y="185.12219"><tspan
         x="253.92303 256.79987"
         y="185.12219"
         id="tspan56961"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57077"
       x="196.16318"
       y="208.64375"><tspan
         x="196.16318 199.04001"
         y="208.64375"
         id="tspan57075"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57185"
       x="184.64333"
       y="249.60422"><tspan
         x="184.64333 187.52016"
         y="249.60422"
         id="tspan57183"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57209"
       x="245.28391"
       y="249.60422"><tspan
         x="245.28391 248.16074"
         y="249.60422"
         id="tspan57207"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#4c9e16;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57263"
       x="149.92166"
       y="261.44232"><tspan
         x="149.92166 152.79849"
         y="261.44232"
         id="tspan57261"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#4c9e16;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57299"
       x="213.44295"
       y="261.44232"><tspan
         x="213.44295 216.31978"
         y="261.44232"
         id="tspan57297"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57377"
       x="204.80231"
       y="279.04333"><tspan
         x="204.80231 207.67915"
         y="279.04333"
         id="tspan57375"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57485"
       x="213.44295"
       y="314.24387"><tspan
         x="213.44295 216.31978"
         y="314.24387"
         id="tspan57483"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57497"
       x="230.88185"
       y="314.24387"><tspan
         x="230.88185 233.75868 236.63553"
         y="314.24387"
         id="tspan57495"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57551"
       x="167.20143"
       y="331.84488"><tspan
         x="167.20143 170.07826"
         y="331.84488"
         id="tspan57549"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57599"
       x="184.64333"
       y="343.52386"><tspan
         x="184.64333 187.52016"
         y="343.52386"
         id="tspan57597"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57611"
       x="201.9231"
       y="343.52386"><tspan
         x="201.9231 204.79993 207.67677"
         y="343.52386"
         id="tspan57609"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57653"
       x="172.96284"
       y="349.44443"><tspan
         x="172.96284 175.83969"
         y="349.44443"
         id="tspan57651"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57761"
       x="172.96284"
       y="384.64493"><tspan
         x="172.96284 175.83969"
         y="384.64493"
         id="tspan57759"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text57965"
       x="207.68153"
       y="454.88538"><tspan
         x="207.68153 210.55836"
         y="454.88538"
         id="tspan57963"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58043"
       x="178.72276"
       y="472.48639"><tspan
         x="178.72276 181.73882"
         y="472.48639"
         id="tspan58041"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58067"
       x="236.64326"
       y="472.48639"><tspan
         x="236.64326 239.5201"
         y="472.48639"
         id="tspan58065"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58109"
       x="201.9231"
       y="478.40698"><tspan
         x="201.9231 204.79993"
         y="478.40698"
         id="tspan58107"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#000000;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58199"
       x="167.20143"
       y="496.00647"><tspan
         x="167.20143 170.07826"
         y="496.00647"
         id="tspan58197"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#4c9e16;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58319"
       x="149.92166"
       y="548.48682"><tspan
         x="149.92166 152.79849"
         y="548.48682"
         id="tspan58317"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#4c9e16;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58355"
       x="213.44295"
       y="548.48682"><tspan
         x="213.44295 216.31978"
         y="548.48682"
         id="tspan58353"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#4c9e16;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58517"
       x="149.92166"
       y="601.28833"><tspan
         x="149.92166 152.79849"
         y="601.28833"
         id="tspan58515"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><text
       style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:4.64006px;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';writing-mode:lr-tb;fill:#4c9e16;fill-opacity:1;fill-rule:nonzero;stroke:none;stroke-width:0.014872"
       id="text58553"
       x="213.44295"
       y="601.28833"><tspan
         x="213.44295 216.31978"
         y="601.28833"
         id="tspan58551"
         style="font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code';stroke-width:0.014872" /></text><g
       id="g93461"
       transform="translate(-2.7652617)"><text
         xml:space="preserve"
         style="font-weight:bold;font-size:5.48034px;line-height:1.45;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Bold';text-align:center;letter-spacing:0px;word-spacing:0px;text-anchor:middle;stroke-width:1"
         x="148.79193"
         y="-160.09645"
         id="text5173-6"><tspan
           sodipodi:role="line"
           id="tspan60483"
           x="0"
           y="0"><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="221.33521"
             id="tspan5269-0" /></tspan><tspan
           sodipodi:role="line"
           id="tspan60485"
           x="148.79193"
           y="316.69312"><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="316.69312"
             id="tspan60487"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan60625">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan400556-2">TcPluginM</tspan>  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan401686-4">TcPluginM</tspan>, tcPluginTrace, tcPluginIO)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="324.63959"
             id="tspan5295-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan227068-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan399722-0">Type</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="332.58612"
             id="tspan5297-1">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan398604-7">Kind</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan397043-1">PredType</tspan>, eqType, mkTyVarTy, tyConAppTyCon_maybe)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="340.53259"
             id="tspan5299-4"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan227890-7">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan395019-1">TysWiredIn</tspan> (typeNatKind)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="348.47906"
             id="tspan5303-9"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan228740-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan391008-5">Coercion</tspan>   (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan393206-7">CoercionHole</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan393796-8">Role</tspan> (..), mkUnivCo)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="356.4256"
             id="tspan5305-0"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan229858-5">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan390250-7">Class</tspan>      (className)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="364.37207"
             id="tspan5307-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan230652-0">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan389340-8">TcPluginM </tspan> (newCoercionHole, tcLookupClass, newEvVar)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="372.31854"
             id="tspan5309-7"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan231590-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan385590-9">TcRnTypes </tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan386532-7">TcPlugin</tspan> (..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan387122-4">TcPluginResult</tspan>(..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="380.26508"
             id="tspan5311-0"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan233404-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan383918-2">TyCoRep</tspan>    (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan384536-8">UnivCoProvenance</tspan> (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="388.21155"
             id="tspan5313-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan234182-8">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan382660-3">TcType</tspan>     (isEqPred)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="396.15802"
             id="tspan5315-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan236128-8">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan381260-6">TyCon</tspan>      (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan380610-2">TyCon</tspan>)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="404.10455"
             id="tspan5317-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan236914-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan378072-0">TyCoRep</tspan>    (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan378922-8">Type</tspan> (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="412.05103"
             id="tspan5319-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan237788-5">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan376974-7">TcTypeNats</tspan> (typeNatAddTyCon, typeNatExpTyCon, typeNatMulTyCon,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="419.9975"
             id="tspan5321-8">                   typeNatSubTyCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="427.94403"
             id="tspan5325-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan239754-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan376080-6">TcTypeNats</tspan> (typeNatLeqTyCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="435.8905"
             id="tspan5327-6"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan240732-3">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan375414-5">TysWiredIn</tspan> (promotedFalseDataCon, promotedTrueDataCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="443.83698"
             id="tspan5331-7">#if MIN_VERSION_ghc(8,10,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="451.78351"
             id="tspan5333-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan241338-5">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan369557-9">Constraint</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="459.72998"
             id="tspan5335-8">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan365853-0">Ct</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan366663-1">CtEvidence</tspan> (..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan367641-3">CtLoc</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan427964-2">TcEvDest</tspan> (..), ctEvidence, ctEvLoc, ctEvPred,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="467.67645"
             id="tspan5337-1">   ctLoc, ctLocSpan, isGiven, isWanted, mkNonCanonical, setCtLoc, setCtLocSpan,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="475.62299"
             id="tspan5339-8">   isWantedCt)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="483.56946"
             id="tspan5341-9"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan242744-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan359872-3">Predicate</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="491.51599"
             id="tspan5343-8">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan361442-7">EqRel</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan362955-8">NomEq</tspan>), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan363813-0">Pred</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan364655-4">EqPred</tspan>), classifyPredType, getEqPredTys, mkClassPred,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="499.46246"
             id="tspan5345-5">   mkPrimEqPred, getClassPredTys_maybe)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="507.40894"
             id="tspan5347-7"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan243794-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan339964-7">Type</tspan> (typeKind)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="515.35547"
             id="tspan5349-0">#else</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="523.30194"
             id="tspan5351-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan244196-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan339270-5">TcRnTypes</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="531.24841"
             id="tspan5353-8">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan334603-4">Ct</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan335521-1">CtEvidence </tspan>(..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan336491-6">CtLoc</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan337409-0">TcEvDest</tspan> (..), ctEvidence, ctEvLoc, ctEvPred,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="539.19495"
             id="tspan5355-6">   ctLoc, ctLocSpan, isGiven, isWanted, mkNonCanonical, setCtLoc, setCtLocSpan,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="547.14142"
             id="tspan5357-3">   isWantedCt)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="555.08789"
             id="tspan5359-7"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan245802-8">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan333581-4">TcType</tspan> (typeKind)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="563.03442"
             id="tspan5361-9"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan246844-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan326287-8">Type</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="570.9809"
             id="tspan5363-3">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan325405-6">EqRel</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan329757-3">NomEq</tspan>), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan327453-4">PredTree</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan328975-1">EqPred</tspan>), classifyPredType, mkClassPred, mkPrimEqPred,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="578.92737"
             id="tspan5365-2">   getClassPredTys_maybe)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="586.8739"
             id="tspan5367-8">#if MIN_VERSION_ghc(8,4,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="594.82037"
             id="tspan5369-4"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan247538-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan323911-4">Type</tspan> (getEqPredTys)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="602.76685"
             id="tspan5371-0">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="610.71338"
             id="tspan5373-3">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="618.65985"
             id="tspan5377-9">#if MIN_VERSION_ghc(8,10,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="626.60632"
             id="tspan5379-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan248948-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan322169-0">Constraint</tspan> (ctEvExpr)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="634.55286"
             id="tspan5381-7">#elif MIN_VERSION_ghc(8,6,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="642.49933"
             id="tspan5383-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan249950-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan321463-7">TcRnTypes</tspan>  (ctEvExpr)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="650.4458"
             id="tspan5385-0">#else</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="658.39233"
             id="tspan5387-6"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan250404-5">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan320741-4">TcRnTypes</tspan>  (ctEvTerm)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="666.33881"
             id="tspan5389-5">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="674.28528"
             id="tspan5393-3">#if MIN_VERSION_ghc(8,2,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="682.23181"
             id="tspan5395-2">#if MIN_VERSION_ghc(8,10,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="690.17828"
             id="tspan5397-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan213148-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan317154-4">Constraint</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan316532-7">ShadowInfo</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan315510-1">WDeriv</tspan>))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="698.12482"
             id="tspan5399-5">#else</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="706.07129"
             id="tspan5401-0"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan211382-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan312804-4">TcRnTypes</tspan>  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan313578-6">ShadowInfo</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan314432-2">WDeriv</tspan>))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="714.0177"
             id="tspan5403-0">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="721.96436"
             id="tspan5405-4">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="729.91077"
             id="tspan5409-8">#if MIN_VERSION_ghc(8,10,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="148.79193"
             y="737.8573"
             id="tspan5411-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan210740-0">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan311690-6">TcType</tspan> (isEqPrimPred)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="745.80383"
             id="tspan5413-8">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="148.79193"
             y="753.75018"
             id="tspan5415-2">#endif</tspan></tspan></text><text
         xml:space="preserve"
         style="font-weight:bold;font-size:5.48034px;line-height:1.45;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Bold';text-align:center;letter-spacing:0px;word-spacing:0px;text-anchor:middle;stroke-width:1"
         x="-138.86426"
         y="300.41788"
         id="text5173-6-2"><tspan
           sodipodi:role="line"
           id="tspan39360"
           x="-138.86426"
           y="300.41788"><tspan
             id="tspan5171-8-2"
             style="font-style:normal;font-variant:normal;font-weight:bold;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Bold';text-align:start;text-anchor:start;fill:#c05db3;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="300.41788">-- GHC API</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="308.36435"
             id="tspan5175-5-8">#if MIN_VERSION_ghc(9,0,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="316.31082"
             id="tspan5177-6-4"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan143497-9-3">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan253305-9-9">GHC.Builtin.Names</tspan> (knownNatClassName, eqTyConKey, heqTyConKey, hasKey)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="324.25735"
             id="tspan5179-1-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan155023-9-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan254851-9-1">GHC.Builtin.Types</tspan> (promotedFalseDataCon, promotedTrueDataCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="332.20383"
             id="tspan5181-3-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan167597-4-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan256157-2-8">GHC.Builtin.Types.Literals</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="340.15033"
             id="tspan5183-2-4">  (typeNatAddTyCon, typeNatExpTyCon, typeNatMulTyCon, typeNatSubTyCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="348.0968"
             id="tspan5185-0-6">#if MIN_VERSION_ghc(9,2,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="356.0433"
             id="tspan5187-6-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan168979-9-7">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan256995-0-7">GHC.Builtin.Types</tspan> (naturalTy)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="363.98981"
             id="tspan5189-3-9"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan170361-3-7">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan258053-5-0">GHC.Builtin.Types.Literals</tspan> (typeNatCmpTyCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="371.93631"
             id="tspan5191-0-5">#else</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="379.88281"
             id="tspan5193-2-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan172347-0-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan260615-9-7">GHC.Builtin.Types</tspan> (typeNatKind)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="387.82928"
             id="tspan5195-4-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan173065-0-2">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan261933-2-8">GHC.Builtin.Types.Literals</tspan> (typeNatLeqTyCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="395.77579"
             id="tspan5197-6-4">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="403.72229"
             id="tspan5199-2-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan174715-6-2">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan263107-7-3">GHC.Core</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan265761-1-2">Expr</tspan> (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="411.66876"
             id="tspan5201-6-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan175525-9-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan266695-2-7">GHC.Core.Class</tspan> (className)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="419.61523"
             id="tspan5203-2-0"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan176083-9-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan270763-7-1">GHC.Core.Coercion</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan271489-0-4">CoercionHole</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan268293-8-2">Role</tspan> (..), mkUnivCo)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="427.56174"
             id="tspan5205-4-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan185941-0-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan271947-5-6">GHC.Core.Predicate</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="435.50824"
             id="tspan5207-5-5">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan273597-3-0">EqRel</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan274555-0-2">NomEq</tspan>), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan275229-3-6">Pred</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan276295-2-9">EqPred</tspan>), classifyPredType, getEqPredTys, mkClassPred,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="443.45474"
             id="tspan5209-1-8">   mkPrimEqPred, isEqPred, isEqPrimPred, getClassPredTys_maybe)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="451.40121"
             id="tspan5211-2-4"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan186795-5-7">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan277093-3-9">GHC.Core.TyCo.Rep</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan277959-1-9">Type</tspan> (..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan278665-2-1">UnivCoProvenance</tspan> (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="459.34772"
             id="tspan5213-9-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan187633-0-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan284333-4-9">GHC.Core.TyCon</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan285243-3-7">TyCon</tspan>)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="467.29422"
             id="tspan5215-9-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan189747-4-0">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan287113-5-6">GHC.Core.Type</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="475.24069"
             id="tspan5217-7-9">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan287811-2-2">Kind</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan288813-7-2">PredType</tspan>, eqType, mkTyVarTy, tyConAppTyCon_maybe, typeKind)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="483.18719"
             id="tspan5219-0-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan190549-9-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan289547-0-9">GHC.Driver.Plugins</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan290469-9-5">Plugin</tspan> (..), defaultPlugin, purePlugin)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="491.1337"
             id="tspan5221-5-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan191175-0-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan291343-3-0">GHC.Tc.Plugin</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="499.08017"
             id="tspan5223-2-1">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan292001-3-7">TcPluginM</tspan>, newCoercionHole, tcLookupClass, tcPluginTrace, tcPluginIO,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="507.02667"
             id="tspan5225-2-2">   newEvVar)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="514.97314"
             id="tspan5227-4-1">#if MIN_VERSION_ghc(9,2,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="522.91968"
             id="tspan5229-0-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan192957-0-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan294191-7-4">GHC.Tc.Plugin</tspan> (tcLookupTyCon)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="530.86615"
             id="tspan5231-5-8">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="538.81268"
             id="tspan5233-2-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan193891-1-8">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan295045-9-8">GHC.Tc.Types</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan295895-8-3">TcPlugin</tspan> (..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan296481-9-2">TcPluginResult</tspan> (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="546.75916"
             id="tspan5235-7-9"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan197178-7-7">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan300586-9-6">GHC.Tc.Types.Constraint</tspan></tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="554.70563"
             id="tspan5237-4-3">  (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan302344-1-4">Ct</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan303714-8-3">CtEvidence</tspan> (..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan304908-7-2">CtLoc</tspan>, <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan305814-1-3">TcEvDest</tspan> (..), <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan306760-8-9">ShadowInfo</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan307306-3-7">WDeriv</tspan>), ctEvidence,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="562.65216"
             id="tspan5239-0-5">   ctLoc, ctLocSpan, isGiven, isWanted, mkNonCanonical, setCtLoc, setCtLocSpan,</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="570.59863"
             id="tspan5241-4-2">   isWantedCt, ctEvLoc, ctEvPred, ctEvExpr)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="578.54517"
             id="tspan5243-2-3"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan199096-2-5">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan343299-8-1">GHC.Tc.Types.Evidence</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan344201-3-0">EvTerm</tspan> (..), evCast, evId)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="586.49164"
             id="tspan5245-4-1">#if MIN_VERSION_ghc(9,2,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="594.43811"
             id="tspan5247-1-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan199942-9-7">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan346047-2-1">GHC.Data.FastString</tspan> (fsLit)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="602.38464"
             id="tspan5249-0-4"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan202348-1-6">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan347337-0-1">GHC.Types.Name.Occurrence</tspan> (mkTcOcc)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="610.33112"
             id="tspan5251-8-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan203250-5-4">import</tspan><tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan348823-1-8"> GHC.Unit.Module</tspan> (mkModuleName)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="618.27759"
             id="tspan5253-9-3">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="626.22412"
             id="tspan5255-6-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan204012-3-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan350381-0-5">GHC.Utils.Outputable</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan350919-9-5">Outputable</tspan> (..), (<tspan
   style="fill:#4c9e16;fill-opacity:1;stroke-width:1"
   id="tspan414456-3-0">&lt;+&gt;</tspan>), (<tspan
   style="fill:#4c9e16;fill-opacity:1"
   id="tspan28095-4">&lt;&gt;</tspan>), text)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="634.17059"
             id="tspan5257-3-0">#else</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="642.11707"
             id="tspan5259-5-6">#if MIN_VERSION_ghc(8,5,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="650.0636"
             id="tspan5261-8-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan206442-9-1">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan353157-1-4">CoreSyn</tspan>    (Expr (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="658.01007"
             id="tspan5263-9-5">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="665.95654"
             id="tspan5265-9-1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan207024-9-0">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan353995-8-1">Outputable </tspan>(<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan354801-5-8">Outputable</tspan> (..), (<tspan
   style="fill:#4c9e16;fill-opacity:1;stroke-width:1"
   id="tspan422136-7-8">&lt;+&gt;</tspan>), (<tspan
   style="fill:#4c9e16;fill-opacity:1"
   id="tspan28505-7">&lt;&gt;</tspan>), text)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="673.90308"
             id="tspan5267-4-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan207846-8-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan355787-2-7">Plugins</tspan>    (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan356509-8-7">Plugin</tspan> (..), defaultPlugin)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="681.84955"
             id="tspan5269-0-1">#if MIN_VERSION_ghc(8,6,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="689.79602"
             id="tspan5271-3-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan215398-4-4">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan404870-2-4">Plugins</tspan>    (purePlugin)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="697.74255"
             id="tspan5273-8-1">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="705.68903"
             id="tspan5275-0-2"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan220752-7-9">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan405772-8-4">PrelNames</tspan>  (hasKey, knownNatClassName)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="713.6355"
             id="tspan5277-6-5"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan219718-2-8">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan406594-0-8">PrelNames</tspan>  (eqTyConKey, heqTyConKey)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="721.58203"
             id="tspan5279-9-8"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan218852-8-5">import </tspan><tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan407660-9-5">TcEvidence</tspan> (<tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan408654-6-3">EvTerm</tspan> (..))</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="729.5285"
             id="tspan5281-3-8">#if MIN_VERSION_ghc(8,6,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="737.47498"
             id="tspan5283-7-6"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan222494-6-0">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan409388-1-5">TcEvidence</tspan> (evCast, evId)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="745.42151"
             id="tspan5285-6-5">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="753.36798"
             id="tspan5287-7-6">#if !MIN_VERSION_ghc(8,4,0)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="761.31445"
             id="tspan5289-1-6"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan223716-7-8">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan402428-0-6">TcPluginM</tspan>  (zonkCt)</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:600;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Semi-Bold';text-align:start;text-anchor:start;fill:#bc7a00;fill-opacity:1;stroke-width:1"
             x="-138.86426"
             y="769.26099"
             id="tspan5291-9-9">#endif</tspan><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="777.20746"
             id="tspan5293-4-3"><tspan
               style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
               id="tspan400556-2-3"></tspan></tspan></tspan><tspan
           sodipodi:role="line"
           id="tspan39362"
           x="-138.86426"
           y="308.36438"><tspan
             style="font-style:normal;font-variant:normal;font-weight:500;font-stretch:normal;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Medium';text-align:start;text-anchor:start;stroke-width:1"
             x="-138.86426"
             y="777.20746"
             id="tspan39364"><tspan
               style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
               id="tspan39502"></tspan></tspan></tspan><tspan
           sodipodi:role="line"
           x="-138.86426"
           y="316.31085"
           id="tspan40265" /></text></g><g
       id="g90173"
       class="fragment"
       transform="matrix(1.9206515,0,0,0.69687341,-328.56361,262.65944)"
       style="stroke-width:0.864368"><path
         style="fill:none;stroke:#ac1010;stroke-width:8.64368;stroke-linecap:round;stroke-linejoin:miter;stroke-miterlimit:4;stroke-dasharray:none;stroke-opacity:1"
         d="M 354.54513,89.276229 121.08809,714.6508"
         id="path89790" /><path
         style="fill:none;stroke:#ac1010;stroke-width:8.64368;stroke-linecap:round;stroke-linejoin:miter;stroke-miterlimit:4;stroke-dasharray:none;stroke-opacity:1"
         d="M 121.08809,89.276229 354.54513,714.6508"
         id="path90169" /></g><g
       id="g102727"
       class="fragment"
       transform="translate(-109.15537,133.57576)"><rect
         style="fill:#ffffff;fill-opacity:1;stroke:#000000;stroke-width:6;stroke-linecap:round;stroke-linejoin:bevel;stroke-miterlimit:4;stroke-dasharray:none;stroke-dashoffset:7.2;stroke-opacity:1;paint-order:fill markers stroke"
         id="rect102335"
         width="449.28748"
         height="81.64949"
         x="12.710854"
         y="370.85504" /><text
         xml:space="preserve"
         style="font-weight:bold;font-size:28.0228px;line-height:1.45;font-family:'Fira Code';-inkscape-font-specification:'Fira Code Bold';text-align:center;letter-spacing:0px;word-spacing:0px;text-anchor:middle;stroke-width:1"
         x="237.19649"
         y="421.93695"
         id="text91575"><tspan
           id="tspan91573"
           x="237.19649"
           y="421.93695"
           style="stroke-width:1"><tspan
   style="fill:#ad1616;fill-opacity:1;stroke-width:1"
   id="tspan96171">import</tspan> <tspan
   style="fill:#1c72c4;fill-opacity:1;stroke-width:1"
   id="tspan98997">GHC.TcPlugin.API</tspan></tspan></text></g></g></svg>

</p>
:::

::: notes
Type-checking plugins manipulate GHC's internal representation of types.

We have access to pretty much of all of GHC!
This means we need to know a lot about how GHC works.

GHC's typechecker changes quite regularly.
For plugin authors, this usually means a large amount of CPP macros.
The library should absorb the impact instead.

:::

## Typed holes

```haskell

import GHC.TcPlugin.API

-- Want to use a 'Coercion' as equality constraint evidence.
help :: Coercion -> EvTerm
help = _
```

:::{.element: class="fragment"}
```

    * Found hole: _ :: Coercion -> EvTerm
    * Valid hole fits include
        evCoercion :: Coercion -> EvTerm
```
:::


## Typed holes (2)

```haskell

import GHC.TcPlugin.API

-- Want to apply a 'TyCon' to arguments.
help :: TyCon -> [Type] -> Type
help = _
```

:::{.element: class="fragment"}
```

    * Found hole: _ :: TyCon -> [Type] -> Type
    * Valid hole fits include
        mkTyConApp :: TyCon -> [Type] -> Type
```
:::

## Hackage documentation

:::{.element: class="fragment"}
<p align="center">
<img src=ghc-tcplugin-api-hackage.png height="250px" />
</p>
:::

:::{.element: class="fragment"}
<p align="center">
<img src=ghc-tcplugin-api-hackage-2.png height="250px" />
</p>
:::

::: notes

Snippets like these help users get started.
Crucial middle-ground between high-level overviews and real-world code.

:::

## Custom type errors

<br />

Helpers for custom type errors.

:::{.element: class="fragment"}
```haskell

myErrMsg :: Type -> TcPluginErrorMessage
myErrMsg ty =
   Txt "Couldn't handle type " :|: PrintType ty :|: Txt "."
  -- Same process as with the 'ErrorMessage' data kind.

errorWantedCt :: MonadTcPluginWork m => CtLoc -> Type -> m Ct
errorWantedCt loc ty =
   newWanted loc $ mkTcPluginErrorTy (myErrMsg ty)
```
:::

::: notes

This is just one quality-of-life improvement offered by the library.

It's much easier to iterate and experiment with a library,
rather than the API being tied to GHC and hard to change.

Much more can be done! Suggestions welcome!
:::


## Available now!

<br />

:::{.element: class="fragment"}
Get your hands dirty: [`ghc-tcplugin-api`](https://hackage.haskell.org/package/ghc-tcplugin-api) on Hackage.  
:::

<br />

:::{.element: class="fragment"}
Type-family rewriting functionality in GHC 9.4.  
Compatibility layer in `ghc-tcplugin-api` for GHC 9.0 and 9.2.
:::

<br />

:::{.element: class="fragment"}
[`ghc-tcplugin-api` on GitHub](https://github.com/sheaf/ghc-tcplugin-api)
contains the System F example.  
Slides: [github.com/sheaf/HIW-tcplugins](github.com/sheaf/HIW-tcplugins)
:::

<br />

:::{.element: class="fragment"}
Upcoming Tweag blog post: writing and debugging a type-checking plugin.  
In depth review of GHC constraint solving and type family reduction.
:::

::: notes

I extend my thanks to Richard Eisenberg, who generously provided
me with a lot of help and high quality feedback.
It's been an absolute pleasure working with him.

:::
