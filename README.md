Ractive.js template specification
=================================


Typically, a developer would create a Ractive instance like so:

```js
var ractive = new Ractive({
  el: 'body',
  template: '<h1>Hello {{name}}!</h1>',
  data: { name: 'world' }
});
```

In this example we are passing a string as the `template` option - a string containing a template written in Ractive's default templating language, which is a variant of [Mustache](http://mustache.github.io/).

But we could have passed the parsed template instead:

```js
var ractive = new Ractive({
  el: 'body',
  template: [{t:7,e:'h1',f:['Hello ',{t:2,r:'name'},'!']}],
  data: { name: 'world' }
});
```

This document details the format of these parsed templates. **It is a living standard**, which is developer-speak for 'it might change without notice'. The hope is that documenting it here will at least make those changes visible and transparent (is that an oxymoron?).



Notes
-----

The format is designed to be compact, not human readable, since in some cases parsed templates will be sent from server to client (or inlined in app code, if parsing happens as a build step).

Since the parsed template is a POJO (plain old JavaScript object), it can be stored as JSON. We will use standard JavaScript notation (without all the fussy double-quoting JSON insists on) in the examples below, together with examples of each item written in Mustache.

Some terminology:

* **fragment** - an array of *items*. There is a single *root fragment*. Certain items, such as elements, may have fragments of their own.
* **item** - a node in the abstract syntax tree, to use traditional parsing jargon. 'Item' is preferred since the word 'node' is used frequently in Ractive to refer to DOM nodes.
* **reference** - data binding targets *refer* to the data they wish to bind to. References are context dependant - the `bar` reference in `{{#foo}}{{bar}}{{/foo}}` may *resolve* to `foo.bar` at runtime, if `foo` turns out to be an object with a `bar` property.
* **expression** - a data binding target may use an expression instead of a reference - e.g. `{{foo + bar}}`. These expressions contain references which are evaluated at runtime - the expression as a whole is re-evaluated when the data those references point to change. See the [section on expressions](#expressions).
* **reference expression** - some expressions, such as `{{foo[bar]}}`, will resolve to references (in this example, whenever the value of `bar` changes). These are treated differently to regular expressions, which allows them to be used for two-way binding. See the [section on reference expressions](#reference-expressions).
* **directive** - instructions on how an element should render and behave, such as `on-click='activate'` or `intro='fade'` look like attributes, but are not rendered to the DOM. They are treated differently by the parser depending on the directive type. See the [section on directives](#directives).



Type code reference
-------------------

Each item in the template (other than text items, which are represented as strings) has a `t` property indicating its type. These codes are as follows:

```js
[
  INTERPOLATOR : 2,
  TRIPLE       : 3,
  SECTION      : 4,
  ELEMENT      : 7,
  PARTIAL      : 8,
  COMMENT      : 9,
  COMPONENT    : 15
]
```


Format
------

### Text

```js
// Before
'I am some text'

// After
'I am some text'
```

Plain text is represented as a string, whether it will become a text node or an element attribute.

### Interpolator

The interpolator is the atomic unit of data-binding.

```js
// Before
'{{foo.bar}}'

// After
{
  t: 2,
  r: 'foo.bar'    // the reference
}
```

As an alternative to a reference, an interpolator may have an *expression*...

```js
// Before
'{{foo + bar}}'

// After
{
  t: 2,
  x: {                   // the expression
    r: ['foo', 'bar'],   // the references used in the expression
    s: '${0}+${1}'       // a string representing the expression
  }
}
```

...or a *reference expression*, which at runtime will resolve to a reference, and can thus be used for two-way binding:

```js
// Before
'{{foo[bar]}}'

// After
{
  t: 2,
  kx: {
    r: 'foo',            // the base reference
    m: [                 // the 'members' of the reference expression
      {
        t: 30,           // the type of member (see below)
        n: 'bar'         // the name of the member
      }
    ]
  }
}
```

See the sections on [expressions](#expressions) and [reference expressions](#reference-expressions) for more information.

Expressions and reference expressions can also be used with [triples](#triple) and [sections](#section). For brevity, they are omitted from the examples below.

### Triple

```js
// Before
'{{{foo}}}'

// After
{
  t: 3,
  r: 'foo'    // the reference
}
```

### Section

```js
// Before
'{{#foo}}...{{/foo}}'

// After
{
  t: 4,
  r: 'foo',
  f: ['...']  // the section fragment
}
```

Sections can be 'inverted' - in other words, they only render when their condition (`foo` in the example) is falsy:

```js
// Before
'{{^foo}}...{{/foo}}'

// After
{
  t: 4,
  r: 'foo',
  f: ['...'],
  n: 1        // 'n' means 'inverted'. `1` is used instead of `true`, to save space
}
```

Note: this will likely change in the near future. In regular Mustache, lists, objects and conditional sections are all treated similarly, and differentiated at render time. Only inverted sections look different. Some templating languages, e.g. [Handlebars](http://handlebarsjs.com/), allow the template author to specify the type of section, which is useful.


### Element

An 'element' may in fact be a component - this is determined at render time. As far as the parser is concerned, they are the same.

```js
// Before
<div id='box' class='type-{{foo}}' intro='fade:{delay:500}' on-click='log:{{hello}}'>...</div>

// After
{
  t: 7,
  e: 'div',
  a: {               // attributes (should only exist if there are one or more attributes)
    id: 'box',       // static attributes are stored as strings
    class: [         // dynamic attributes are stored as fragments
      'type-',
      { t: 2, r: 'foo' }
    ]
  },
  t1: {              // t1 is `intro`, t2 is `outro`, t0 is `intro-outro`
    n: 'fade',       // directive name
    a: [{            // directive arguments - `a` - are parsed
      delay: 500
    }]
  },
  v: {               // proxy event directives - a map of name:definition pairs
    click: {
      n: 'log',
      d: [{          // this time, the directive has dynamic arguments - `d` instead of `a`
        t: 2,
        r: 'hello'
      }]
    }
  },
  f: ['...']         // element children fragment
}

TODO the rest of this document