---
title: Switch Function
eleventyNavigation:
  key: switch-function
  title: Switch Function
  parent: rwd
---

The `switch()` function would provide conditional logic in CSS values,
similar to the Sass `if()` function,
but with access to some essential client-side values for comparison:
primarily `available-inline-size`.

## Resources

_From [Brian Kardell](https://bkardell.com/) & [Igalia](https://www.igalia.com/)_

- [Towards Responsive Elements](https://bkardell.com/blog/TowardResponsive.html)
- [A `switch()` function in CSS](https://gist.github.com/bkardell/e5d702b15c7bcf2de2d60b80b916e53c)
- [All Them Switches](https://bkardell.com/blog/AllThemSwitches.html)

## Background

The [query-block](query/) approach
relies on [CSS Containment](https://drafts.csswg.org/css-contain/)
to avoid infinite loops -
and specifically [single-axis size containment](https://github.com/w3c/csswg-drafts/issues/1031)
which _might not be possible_.

The `switch()` proposal avoids those issues by limiting:

- The properties that an author is able to switch,
  to ensure they never impact container size
  (no `font-size`, `width`, etc)
- The values that can be used inside a switch,
  to avoid calculations that have to run at layout-time
  (no `attr()`, `calc()`, etc)

Those limits come from the internal architecture of browser engines,
and make it possible to implement `switch()`
without relying on other hypothetical CSS features.

Igalia has already developed a working prototype:

<figure data-ratio>
<iframe width="560" height="315" src="https://www.youtube.com/embed/8QFST9MvjyA" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</figure>

## Syntax

The initial proposal suggests:

```css
.foo {
  display: grid;
  grid-template-columns: switch(
    (available-inline-size > 1024px) 1fr 4fr 1fr;
    (available-inline-size > 400px) 2fr 1fr;
    (available-inline-size > 100px) 1fr;
    default 1fr;
   );
}
```

The implemented prototype is a bit different,
but seems like a temporary way to fit existing CSS syntax:

```css
.foo {
  display: grid;
  grid-template-columns: switch(
    auto /
     (available-inline-size > 1000px) 1fr 2fr 1fr 2fr /
     (available-inline-size > 500px) auto 1fr /
   );
}
```

There is also a comment from
[Fantasai](https://gist.github.com/bkardell/e5d702b15c7bcf2de2d60b80b916e53c#gistcomment-3295085)
with a proposal to make the syntax more efficient:

```css
.foo {
  grid-template-columns: switch( available-inline-size ?
    (? > 1024px) 1fr 4fr 1fr;
    (? > 400px) 2fr 1fr;
    (? > 100px) 1fr;
    1fr;
  );
}
```

## Thoughts

This is not a full solution to Container Queries
because of the built-in limitations and single-property approach,
but it could quickly help address
several of the most common and most basic layout use-cases --
especially as demonstrated with grid layout.

The `switch()` function is also very flexible,
and could solve many other context-responsive styling issues --
like adjusting `<em>` styles based on the inherited `font-style`,
or toggling values based on a variable.
That seems promising on multiple fronts.

Any inline-contitional syntax is bound to take up space,
and add clutter to CSS.
I'm not sure that's avoidable,
but it would be good to try and make the syntax
as light-weight as reasonable.

That becomes more of an issue if this were the _only_ way
to write conditional CSS.
Many languages provide both inline & block conditionals,
for good reason --
there are cases for each --
and I think that might also be the way to go in CSS.