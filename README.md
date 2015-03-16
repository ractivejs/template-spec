# Ractive.js template specification


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
  template: {v:3,t:[{t:7,e:'h1',f:['Hello ',{t:2,r:'name'},'!']}]},
  data: { name: 'world' }
});
```

This document details the format of these parsed templates.



## Notes

The format is designed to be compact, not human readable, since in some cases parsed templates will be sent from server to client (or inlined in app code, if parsing happens as a build step).

Since the parsed template is a POJO (plain old JavaScript object), it can be stored as JSON. We will use standard JavaScript notation (without all the fussy double-quoting JSON insists on) in the examples below, together with examples of each item written in Mustache.

Some terminology:

* **fragment** - an array of *items*. There is a single *root fragment*. Certain items, such as elements, may have fragments of their own.
* **item** - a node in the abstract syntax tree, to use traditional parsing jargon. 'Item' is preferred since the word 'node' is used frequently in Ractive to refer to DOM nodes.
* **reference** - data binding targets *refer* to the data they wish to bind to. References are context dependent - the `bar` reference in `{{#foo}}{{bar}}{{/foo}}` may *resolve* to `foo.bar` at runtime, if `foo` turns out to be an object with a `bar` property.
* **expression** - a data binding target may use an expression instead of a reference - e.g. `{{foo + bar}}`. These expressions contain references which are evaluated at runtime - the expression as a whole is re-evaluated when the data those references point to change. See the [section on expressions](#expressions).
* **reference expression** - some expressions, such as `{{foo[bar]}}`, will resolve to references (in this example, whenever the value of `bar` changes). These are treated differently to normal expressions, which allows them to be used for two-way binding. See the [section on reference expressions](#reference-expressions).
* **directive** - instructions on how an element should render and behave, such as `on-click='activate'` or `intro='fade'` look like attributes, but are not rendered to the DOM. They are treated differently by the parser depending on the directive type. See the [section on directives](#directives).


## Version

This is **version 3** of the specification, as shown by the `v: 3` in the [structure](#structure) section below. This ensures that your build tools, or server-side compile steps, are speaking the same language as the copy of Ractive running in your app in the browser.


## Structure

```js
{
  v: 3, /* Template spec version */
  t: [  /* items in the main template */ ],
  p: {  /* Optional hash of partials */
    foo: [ /* items in the 'foo' partial */ ]
  }
}
```



## Type code reference

Each item in the template (other than text items, which are represented as strings) has a `t` property indicating its type. These codes are as follows:

```js
[
  INTERPOLATOR   : 2,
  TRIPLE         : 3,
  SECTION        : 4,
  ELEMENT        : 7,
  PARTIAL        : 8,
  COMMENT        : 9,
  YIELDER        : 16,
  DOCTYPE        : 18
]
```


## Format

### Text

Plain text is represented as a string, whether it will become a text node or an element attribute.

```js
// Before
'I am some text'

// After
'I am some text'
```

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
  rx: {
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
'<div id="box" class="type-{{foo}}" intro="fade:{delay:500}" on-click="log:{{hello}}">...</div>'

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
```

Attributes can exist inside mustache sections:

```js
// Before
'<div {{#if active}}class="active"{{/if}}>...</div>'

// After
{
  t: 7,
  e: "div",
  m: [
    {
      t: 4,
      f: [ 'class="active"' ],
      n: 50,
      r: "active"
    }
  ],
  f: [
    "..."
  ]
}
```

Components can have *inline partial definitions* - these are passed to the component constructor. If the component's template includes a [partial](#partial) or a [yielder](#yielder), the matching partial definition is used.

```js
// Before
`<widget>
  some content
  {{#partial foo}}this is foo{{/foo}}
</widget>`

// After
{
  t: 7,
  e: 'widget',
  f: [ 'some content' ],
  p: {
    foo: [ 'this is foo' ]
  }
}
```


### Partial

Partials are markers for places where template snippets should be inserted.

```js
// Before
'{{>foo}}''

// After
{
  t: 8,
  r: 'foo'
}
```

A partial may use an expression instead of a partial name:

```js
// Before
'{{>partials[type]}}'

// After
{
  t: 8,
  rx: {
    r: "partials",
    m: [
      {
        t: 30,
        n: "type"
      }
    ]
  }
}
```

It may also have a *context*, which determines the value of `this` inside the partial. In effect, it is converted internally to a `{{#with...}}` block:

```js
// Before
'{{>foo items[i]}}'

// Equivalent to
'{{#with items[i]}}{{>foo}}{{/with}}'

// After
{
  t: 4,
  n: 53,
  rx: {
    r: "items",
    m: [
      {
        t: 30,
        n: "i"
      }
    ]
  },
  f: [
    {
      t: 8,
      r: "foo"
    }
  ]
}
```


### Yielder

Yielders are similar to partials, except that the content 'belongs' to the parent rather than the component:

```js
// Before
{{yield foo}}

// After
{
  t: 16,
  r: 'foo'
}
```


### Comment

With the standard parser, comments are stripped unless the `stripComments` option is false.

```js
// Before
'<!-- This is a comment -->'

// After
{
  t: 9,
  c: ' This is a comment '
}
```


### Doctype declaration

If you're using Ractive to render templates on the server (or as part of a build step), your template may feature a `<!DOCTYPE ...>` declaration.

```js
// Before
'<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">'

// After
{
  t: 18,
  a: ' html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd"'
}
```


## Expressions

While Mustache brands itself as a 'logic-less' templating language, Ractive templates can feature JavaScript expressions. See the [docs](http://docs.ractivejs.org/latest/expressions) for more detail.

Ractive's built-in parser is therefore capable of parsing JavaScript expressions to an abstract syntax tree (AST). However the AST is not transported as part of the template; instead, it is used to extract references from the expression (for data-binding purposes) and then flattened to a string representation.

Only a subset of the infinite range of JavaScript expressions is supported (albeit an infinite subset...) - those which do not use assignment operators (such as `=` or `++`), function literals, or `new`/`delete`/`void` operators. This is to ensure security and prevent side-effects.


## Reference expressions

A subset of expressions fall into the category of *reference expressions* - those expressions which, when their references have been resolved and they can be evaluated, will resolve to references. For example the `foo[bar]` expression, once we know that `bar === 'qux'`, is equivalent to `foo.qux`. Treating these expressions differently allows us to perform certain tricks that would otherwise be impossible, such as two-way data binding. This turns out to be incredibly useful when building interactive tabular interfaces, for example.

Reference expressions are denoted by the property name `rx`, for reasons we won't go into.

We'll use a horribly contrived reference expression to illustrate how they are represented in parsed templates:

```js
// Before
'{{one[two]["three"].four[five+6]}}'

// After
{
  t: 2,
  rx: {
    r: 'one'      // the 'base reference'
    m: [          // the 'members'
      { t: 30, n: 'two' },
      { r: [], s: 'three' },
      'four',
      { r: ['five'], s: '${0}+6' }
    ]
  }
}
```

Type `30` (the first member) is used to denote a reference. The second and final members are expressions in their own right. The third is a property name, which is stored as a string.


## Directives

As described earlier, directives look like attributes but aren't rendered to the DOM as attributes. They can be **transition** directives (`intro`, `outro`, or `intro-outro`), **decorator** directives, or **event directives** (`on-click`, `on-mouseover` and so on). There are some rules:

* There can be zero or one `intro` (denoted in the template as `t1`) and zero or one `outro` (`t2`). `intro-outro` (`t0`) counts as one of each.
* There can be zero or one `decorator` (`o`).
* There can be as many event directives as you like, but only one of each kind. Different events that shared a handler can be chained: `on-change-input='update'`.

In their simplest form, decorators are represented as strings:

```js
// Before
'<div on-click="activate">...</div>'

// After
{
  t: 7,
  e: 'div',
  v: {
    click: 'activate'
  }
}
```

However a directive may have arguments, in which case the directive template is an object rather than a string. If the arguments are static (no data-binding), they are converted to objects at parse time. The directive name is denoted by `n`, and the arguments are denoted by `a`.

```js
// Before
'<div on-click="activate:{foo:1,bar:2},42">...</div>'

// After
{
  t: 7,
  e: 'div',
  v: {
    click: {
      n: 'activate',
      a: [
        { foo: 1, bar: 2 },
        42
      ]
    }
  }
}
```

If the arguments are dynamic, they are denoted by `d` instead, which is a fragment. This instructs Ractive's renderer to set up the appropriate data binding.

```js
// Before
'<div on-click="activate:{message:{{message}}}">...</div>'

// After
{
  t: 7,
  e: 'div',
  v: {
    click: {
      n: 'activate',
      d: [
        '{message:',
        { t: 2, r: 'message' },
        '}'
      ]
    }
  }
}
```

This fragment will be parsed whenever the event fires.
