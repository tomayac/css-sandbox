---
title: WIP Scope Explainer
manual_toc: true
eleventyNavigation:
  key: scope-explainer
  title: Scope Explainer
  parent: scope
---

_Based on the
[TAG Explainer Template](https://github.com/w3ctag/w3ctag.github.io/blob/master/explainers.md)_

## Authors

- Miriam Suzanne

## Participate

CSSWG Issues:
- TBA...

## Table of Contents

<!-- generated by VSCode Markdown All-In-One extension -->

- [Authors](#authors)
- [Participate](#participate)
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Goals](#goals)
  - [The namespace problem](#the-namespace-problem)
  - [The nearest-ancestor "proximity" problem](#the-nearest-ancestor-proximity-problem)
  - [The lower-boundary, or "ownership" problem (aka "donut scope")](#the-lower-boundary-or-ownership-problem-aka-donut-scope)
  - [Popular tooling for modular CSS](#popular-tooling-for-modular-css)
- [Non-goals](#non-goals)
- [Proposed Solution](#proposed-solution)
  - [Re-introducing the `@scope` rule](#re-introducing-the-scope-rule)
  - [The (existing) `:scope` pseudo-class](#the-existing-scope-pseudo-class)
  - [Scope "proximity" in the cascade](#scope-proximity-in-the-cascade)
- [Key scenarios](#key-scenarios)
- [Detailed design discussion & alternatives](#detailed-design-discussion--alternatives)
  - [Is there a global `:scope` selector?](#is-there-a-global-scope-selector)
  - [Are scope attributes useful in html?](#are-scope-attributes-useful-in-html)
  - [Should we be building on Shadow DOM?](#should-we-be-building-on-shadow-dom)
  - [Do we need complex selectors to define scope?](#do-we-need-complex-selectors-to-define-scope)
  - [Can scoped selectors reference external context?](#can-scoped-selectors-reference-external-context)
  - [Where does scope fit in the cascade?](#where-does-scope-fit-in-the-cascade)
- [Spec History & Context](#spec-history--context)
  - [CSS Scoping](#css-scoping)
  - [CSS Selectors - Level 4](#css-selectors---level-4)
  - [CSS Cascade - Level 4](#css-cascade---level-4)
- [References & acknowledgements](#references--acknowledgements)

## Introduction

There are many overlapping
and sometimes contradictory features
that can live under the concept of "scope" in CSS --
but they divide roughly into two approaches:

1. Total isolation of a component DOM subtree/fragment from the host page,
   so that no selectors get in or out
   unless explicitly requested.
2. Lighter-touch component namespacing,
   and prioritization of "proximity"
   when resolving the cascade.

That has lead to a wide range of proposals over the years,
including a [scope specification][initial-spec]
that was never implemented.
Focus moved to Shadow-DOM,
which is mainly concerned with approach #1 -- full isolation.
Meanwhile authors have attempted to handle approach #2
through convoluted naming conventions (like [BEM][])
and JS tooling
(such as [CSS Modules][], [Styled Components][], & [Vue Scoped Styles][]).

This document is proposing a native CSS approach
for what many authors are already doing
with those third-party tools & conventions.

[initial-spec]: https://www.w3.org/TR/css-scoping-1/
[BEM]: http://getbem.com/
[CSS Modules]: https://github.com/css-modules/css-modules
[Styled Components]: https://styled-components.com/
[Vue Scoped Styles]: https://vue-loader.vuejs.org/guide/scoped-css.html

## Goals

### The namespace problem

All CSS Selectors are global,
matching against the entire DOM.
As projects grow,
or adapt a more modular "component-composition" approach,
it can be hard to track what names have been used,
and avoid conflicts.

To solve this,
authors rely on
convoluted naming conventions (BEM)
and JS tooling (CSS Modules & Scoped Styles)
to "isolate" selector matching inside a single "component".

### The nearest-ancestor "proximity" problem

Ancestor selectors allow us to
filter the "scope" of nested selectors
to a sub-tree in the DOM:

```css
/* link colors for light and dark backgrounds */
.light-theme a { color: purple; }
.dark-theme a { color: plum; }
```

But problems show up quickly
when you start thinking of these as modular styles
that should nest in any arrangement.

```html
<div class="dark-theme">
  <a href="#">plum</a>

  <div class="light-theme">
    <a href="#">also plum???</a>
  </div>
</div>
```

Our selectors appropriately have the same specificity,
but they are not weighted by
"proximity" to the element being styled.
Instead we fallback to source order,
and `.dark-theme` will always take precedence.

There is no selector/specificity solution
that accurately reflects what we want here --
with the "nearest ancestor" taking precedence.

This was one of the
[original issues highlighted by OOCSS][oocss-proximity]
in 2009.

[oocss-proximity]: https://www.slideshare.net/stubbornella/object-oriented-css/62-CSS_WISH_LIST

### The lower-boundary, or "ownership" problem (aka "donut scope")

While "proximity" is loosely concerned with nesting styles,
the problem comes into more focus
with the concept of modular components --
which can be more complex.

To use BEM terminology,
Components are generally comprised of:

- An outer "block" or component wrapper
- Inner "elements" that belong to that block explicitly
- Occasional "donut holes" or "slots" where sub-components can be nested

In html templating languages,
and JS frameworks,
this can be represented by an "include"
or "single file component".

BEM attempts to convey this "ownership" in CSS:

```css
/* any title inside the component tree */
.component .title { /* too broad */ }

/* only a title that is a direct child of the component */
.component > .title { /* too limiting of DOM structures */ }

/* just the title of the component */
.component__title { /* just right? */ }
```

Nicole Sullivan coined the term
["donut" scope][donut] for this issue in 2011 --
because the scope can have a hole in the middle.
It would be useful for authors
to express this DOM-fragment
"ownership" more clearly in native HTML/CSS.

[donut]: http://www.stubbornella.org/content/2011/10/08/scope-donuts/

### Popular tooling for modular CSS

CSS Modules, Vue, Styled-JSX, and other tools
often use a similar pattern
(with slight variations to syntax) --
where "scoped" selectors only apply to
the locally described DOM fragment,
and not descendants.

In Vue single file components,
authors can write html templates
with "scoped" style blocks:

```html
<!-- component.vue -->
<template>
  <section class="component">
    <div class="element">...<div>
    <sub-component>...</sub-component>
  </section>
</template>

<style scoped>
.component { /* ... */ }
.element { /* ... */ }
.sub-component { /* ... */ }
</style>

<!-- sub-component.vue -->
<template>
  <section class="sub-component">
    <div class="element">...<div>
  </section>
</template>

<style scoped>
.sub-component { /* ... */ }
.element { /* ... */ }
</style>
```

While the language is similar to shadow-DOM in many ways,
the output is quite different --
and much less isolated.
The components remain part of the global scope,
and only the explicitly "scoped" styles are contained.
That's often achieved by automatically adding unique attributes
to each element based on the component(s) it belongs to:

```html
<section class="component" scope="component">
  <div class="element" scope="component">...<div>

  <!-- nested component "shell" is in both scopes -->
  <section class="sub-component" scope="component sub-component">
    <div class="element" scope="sub-component">...<div>
  </section>
</section>
```

And matching attributes are added to each selector:

```css
/* component.vue styles after scoping */
.component[scope=component] { /* ... */ }
.element[scope=component] { /* ... */ }
.sub-component[scope=component] { /* ... */ }

/* sub-component.vue styles after scoping */
/* note that both style `.element` without any overlap or naming conflicts */
.sub-component[scope=sub-component] { /* ... */ }
.element[scope=sub-component] { /* ... */ }
```

- The donut is achieved by selectively adding attributes
- Proximity-weight is achieved only through limiting the donut of scope,
  so that outer values are less likely to "bleed" in
- Added attribute gives scoped styles _some_ (but very little)
  extra specificity weight in the cascade

## Non-goals

There is a more extreme isolation use-case.
It's mostly used for "widgets" that will appear unchanged
across multiple projects --
but sometimes also in component libraries
on larger projects.

Full isolation blocks off a fragment of the DOM,
so that it _only_ accepts styles that are
explicitly scoped.
General page styles do not apply.

I don't think this is the most common concern for authors,
but it has received the most attention.
Shadow DOM is entirely constructed around this behavior.

I have not attempted to address that form of scope in my proposal --
it feels like a significantly different approach
that already has work underway.

See Yu Han's proposals for
[building on shadow DOM](#building-on-shadow-dom)
below.

## Proposed Solution

### Re-introducing the `@scope` rule

_This would likely belong in
the [CSS Scoping Module](https://drafts.csswg.org/css-scoping/)._

In the long-standing
["Bring Back Scope"](https://github.com/w3c/csswg-drafts/issues/3547)
issue-thread,
Giuseppe Gurgone
[suggests a syntax](https://github.com/w3c/csswg-drafts/issues/3547#issuecomment-524206816) building on the original un-implemented `@scope` spec,
but adding a lower boundary:

```css
@scope (from: .carousel) and (to: .carousel-slide-content) {
  p { color: red }
}
```

I think that's a good place to start.
In my mind, the first ("from") clause should be required,
and may not need explicit labeling.
It would accept a single (complex) selector:

```css
@scope (.media-block) {
  img { border-radius: 50%; }
}
```

In terms of selector-matching,
this would be the same as
`.media-block img`,
but with slightly different cascade implications
(see cascade section).

The second ("to") clause would be optional,
and accept a list of selectors
that represent lower-boundary "slots" in the scope.
The lower-boundary elements are included in the scope,
but their descendants are not:

```css
@scope (.media-block) to (.content) {
  img { border-radius: 50%; }
}
```

Which would only match `img`
inside `.media-block`
_if there is no intervening `.content`
between the scope root and selector target_:

```html
<div class="media-block">
  <img src="..."><!-- this img is in the .media-block scope -->
  <div class="content">
    <img src="..."><!-- this img is NOT in scope -->
  </div>
</div>
```

This approach keeps scoping confined to CSS
(no need for an HTML attribute),
flexible
(scopes can overlap as needed),
and low-impact
(global styles continue to work as expected).
Existing tools would still be able to
provide syntax sugar for single-file components --
automatically generating the from/to clauses --
but move the primary functionality into CSS.

### The (existing) `:scope` pseudo-class

A `:scope` selector has already been defined,
to select the root of any scope.
In existing CSS,
this is the same as `:root`,
since there is no way to scope elements.
However, it is used by JS APIs
to refer to the base element of e.g. `element.querySelector()`.

This can be used by authors to style the root of any scope --
as defined by the `@scope`-rile selector:

```css
@scope (.media-block) {
  /* this selects only the scope-root .media-block  */
  :scope { display: grid; }

  /* this would select nested media-blocks */
  .media-block { background: gray; }
}
```

### Scope "proximity" in the cascade

_This would likely belong in
[CSS Cascading & Inheritance](https://drafts.csswg.org/css-cascade/)._

The syntax above solves the issue of
naming-conflicts, with lower-boundaries/ownership.
But the issue of _scope proximity_ requires changes in the Cascade.
My sense is that scope proximity
should override _source order_,
but otherwise cascade layers & specificity
should take precedence.
This is a big point of departure from the original spec.

Given the same origin & importance, layering, and specificity --
inner "more proximate" scope would take precedence
over outer/global "less proximate" scopes:

```css
@scope (.light-theme) {
  p { color: purple; }
}

@scope (.dark-theme) {
  p { color: plum; }
}
```

```html
<div class="dark-theme">
  <a href="#">plum</a>

  <div class="light-theme">
    <a href="#">purple</a>
  </div>
</div>
```

Given the same proximity of multiple scopes,
source order would continue to be the final cascade filter:

```css
@scope (.light-theme) {
  p { color: purple; }
}

@scope (.special-links) {
  p { color: maroon; }
}
```

```html
<div class="special-links light-theme">
  <a href="#">maroon</a>
</div>
```

## Key scenarios
## Detailed design discussion & alternatives

### Is there a global `:scope` selector?

From [Selectors Level 4](https://www.w3.org/TR/selectors-4/#the-scope-pseudo)

> “Specifications intending for this pseudo-class to match specific elements
> rather than the document’s root element
> must define either a scoping root (if using scoped selectors)
> or an explicit set of :scope elements.”

Those two options seem exclusive to me.
Either `:scope` should select the root of each scope
(where it matches `:root` when in the global scope),
or it can refer to _all scope roots_ --
but allowing both would be problematic.

I don't see much use for the latter definition,
but find it confusing that both are allowed.

### Are scope attributes useful in html?

Yu Han's proposal (like many others)
includes a scope attribute in HTML.
That's required for the full-isolation use-case --
where elements need to opt-in or out of global page styles up-front.

But is it required or useful for less isolated use-cases?
It does come up regularly,
as a form of syntax sugar.
From [Sebastian](https://github.com/w3c/csswg-drafts/issues/3547#issuecomment-693022720):

```css
p { color: blue; }

@scope main {
  p { color: green; }
}
@scope note {
  p { color: gray; }
}
```

```html
<p>This text is blue</p>
<section scope="main">
  <p>This text is green</p>
  <div scope="note">
    <p>This text is gray</p>
  </div>
</section>
<div scope="note">
  <p>This text is gray</p>
</div>
```

The two possible advantages are:

- Scope selectors have their own namespace
- Lower boundaries could be implicit when nesting
  (which also has a flexibility downside)

It seems to me
authors could achieve the same goals manually,
without much extra code:

```css
p { color: blue; }

@scope ([data-scope=main]) to ([data-scope]) {
  p { color: green; }
}
@scope ([data-scope=note]) to ([data-scope]) {
  p { color: gray; }
}
```

```html
<p>This text is blue</p>
<section data-scope="main">
  <p>This text is green</p>
  <div data-scope="note">
    <p>This text is gray</p>
  </div>
</section>
<div data-scope="note">
  <p>This text is gray</p>
</div>
```

So I'm not convinced we need any scope attribute.

### Should we be building on Shadow DOM?

Yu Han has
an [interesting proposal](https://docs.google.com/document/d/1hhjmuQE6BTTnAyKP3spDr8sU6lpXArh8LDfihZh78hw/edit?usp=sharinghttps://docs.google.com/document/d/1hhjmuQE6BTTnAyKP3spDr8sU6lpXArh8LDfihZh78hw/edit?usp=sharing)
in two parts,
designed to build on top of existing shadow DOM logic:

1. Allow shadow-DOM elements to opt-in to global styles
2. Allow light-DOM elements to opt-in to style isolation

This would require a new `scoped` HTML attribute,
because that sort of isolation has to be defined
before any styles are applied.

I think those would be good to consider,
but out-of-scope (🤷🏻‍♀️) for this proposal.

My initial instinct is that --
while the target of a selector must be in-scope to match --
scoped selectors should be _otherwise unaware_ of the scope.
That would allow scoped selectors
to reference ancestor context
that is out of scope:

```css
@scope (.my-component) {
  .my-host-page .my-component-part {
    /* selector matches when both... */
    /* - `.my-host-page .my-component-part` matches globally */
    /* - the targeted element is inside the `.my-component` scope */
  }
}
```

However, it may also work
to restrict the syntax so the entire selector is in-scope.
Additional context in the scoping selector
could be used to achieve the same goal:

```css
@scope (.my-host-page .my-component) {
  .my-component-part { /* ... */ }
}
```

### Do we need complex selectors to define scope?

Given a syntax of `@scope <selector>` are we placing any restrictions on the `<selector>`?

```css
/* In most cases we expect a single/simple selector to work */
@scope [data-component=tabs] { /* ... */  }

/* Not sure there are strong use-cases for context selectors */
@scope .page-context [data-component=tabs] { /* ... */ }
@scope [data-component=tabs].horizontal { /* ... */ }

/* modifiers could be handled internally */
@scope [data-component=tabs] {
  .page-context :scope { /* ... */ }
  :scope.horizontal { /* ... */ }
}
```

Counter-point: is there a reason to dis-allow complex selectors?

### Can scoped selectors reference external context?

My initial instinct is that --
while the target of a selector must be in-scope to match --
scoped selectors should be _otherwise unaware_ of the scope.
That would allow scoped selectors
to reference ancestor context
that is out of scope:

```css
@scope (.my-component) {
  .my-host-page .my-component-part {
    /* selector matches when both... */
    /* - `.my-host-page .my-component-part` matches globally */
    /* - the targeted element is inside the `.my-component` scope */
  }
}
```

However, it may also work
to restrict the syntax so the entire selector is in-scope.
Additional context in the scoping selector
could be used to achieve the same goal:

```css
@scope (.my-host-page .my-component) {
  .my-component-part { /* ... */ }
}
```

### Where does scope fit in the cascade?

The original scope specification
had scope override specificity in the cascade.
Un-scoped styles are treated as-though scoped to the document-root,
and the layering was importance-relative:

> For normal declarations the inner scope's declarations override,
> but for ''!important'' rules outer scope's override.

But I think those semantics should not be coupled with scope,
and will soon be provided by
[cascade layers][https://drafts.csswg.org/css-cascade/].
Both specificity & layers can be used in-conjunction with scope
to control weighting when desired.

Another idea that I considered
was to combine the specificity of the scope selector
to the specificity of nested rule-blocks:

```css
@scope [data-component=tabs] {
  /* `[data-component=tabs] .tab-item`: 0,2,0 */
  .tab-item { /* ... */ }
}

@scope #tabs {
  /* `#tabs .tab-item`: 1,1,0 */
  .tab-item { /* ... */ }
}
```

That seems like a sensible solution,
but I think it also makes the
weight of a selector unnecessarily complicated
for authors.

## Spec History & Context

Besides the tooling that has developed,
there are several current & former specs
that are relevant here...

### CSS Scoping

- [First Public Working Draft][initial-spec]
- [Editors Draft](https://drafts.csswg.org/css-scoping/)

There is often pushback to the question of scope,
since the initial specification was never implemented,
and Shadow DOM was seen as a path forward.
While the current editors draft
is primarily concerned with Custom Elements & Shadow DOM,
this spec initially contained a full set of scoping features
that have since been removed:

A `<style scoped>` attribute,
which would apply styles
scoped to a particular DOM sub-tree.
This had a few limitations:

- Authors need to repeat styles in the DOM for every instance of the scope
- Those style need to live in distinct stylesheets

The use-cases that necessitate that approach
are now being handled by shadow encapsulation,
which frees us up to consider different use-cases now.

The spec also included `@scope` blocks in CSS,
which would help alleviate both issues.
Scoping has two primary effects:

1. The selector of the scoped style rule
   is restricted to match only elements within a subtree of the DOM
2. The cascade prioritizes scoped rules over un-scoped ones,
   regardless of specificity
3. Important declarations would flip the cascade order of scopes

Point 1 is limited by the need for lower scope boundaries,
or "donut scope".

Points 2 & 3 give scope _significant_ power in the cascade --
power that we now plan to provide through Cascade Layers.
While there are instances where the semantics of
layering, scoping, and containment
might reasonably overlap --
I think all three features are better off
with their own syntax.

In my proposal,
scope is only given a _minimal_ role in the cascade,
and mostly acts as a protection from naming conflicts.

### [CSS Selectors - Level 4](https://www.w3.org/TR/selectors-4/)

- [Scoped Selectors](https://www.w3.org/TR/selectors-4/#scoping),
  which only refer to a subtree or fragment of the document
- [Reference Element](https://www.w3.org/TR/selectors-4/#the-scope-pseudo)
  (`:scope`) pseudo-class ":scope elements" or the root of any scope
  (currently used in JS APIs only)

### [CSS Cascade - Level 4](https://www.w3.org/TR/css-cascade-4/)

- [Removes](https://www.w3.org/TR/css-cascade-4/#change-2018-drop-scoped)
  “scoping” from the cascade sort criteria,
  because it has not been implemented.
- Adds [encapsulation context](https://www.w3.org/TR/css-cascade-4/#cascade-context)
  to the cascade, for handling Shadow DOM
  - Outer context wins for *normal* layer conflicts
  - Inner context wins for `!important` layer conflicts

## References & acknowledgements

There are many others who have been working on this,
from various angles...

In the CSSWG GitHub issues:

- [Bring Back Scope](https://github.com/w3c/csswg-drafts/issues/3547):
  - [@scope with lower-bounds](https://github.com/w3c/csswg-drafts/issues/3547#issuecomment-524206816)
  - [@scope with name & attribute](https://github.com/w3c/csswg-drafts/issues/3547#issuecomment-693022720)
- [Selector Boundaries](https://github.com/w3c/csswg-drafts/issues/5057)
- [CSS Namespaces](https://github.com/w3c/csswg-drafts/issues/270)
  - And stated [priorities](https://github.com/w3c/csswg-drafts/issues/270#issuecomment-231586786)

Many thanks for valuable feedback and advice from:

- Anders Hartvoll Ruud
- Ian Kilpatrick
- Mason Freed
- Nicole Sullivan
- Rune Lillesveen
- Tab Atkins
- Una Kravets
- Yu Han