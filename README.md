Livemarkup.js
=============

Special HTML templating language made for [Backbone.js].

**Status:** some code has been written, but is still not feature-complete.  (ie, 
    you can't use Livemarkup.js yet.) This document outlines the intended 
feature set that the initial release will have.

Getting started
---------------

First, make some templates. They're plain HTML, except with directives in `{{ 
  ... }}` and special `l-` attributes.

(This is an inline template example. Templates don't have to be inline, however)

``` html
<script type='text/lm-template' id='album_show'>
  <article class='music-album'>
    <h2 l-onchange='title'>
      {{ model.get('title') }}
    </h2>

    <div l-onchange='artist'>
      Artist:
      {{ model.get('artist') }}
    </div>
  </article>
</script>
```

Let's initialize Livemarkup.js and integrate it with Backbone.js.

``` javascript
LM.useView(Backbone.View);
LM.register($("script[type='text/lm-template']"));
/* (These are explained later) */
```

Now in your views, just define a `templateName` attribute:

``` javascript
var AlbumView = Backbone.View.extend({
  templateName: 'album_show',
});
```

...and initialize your view with a model.

``` javascript
var album = new Album({ ... });  /* Assume a Backbone model */
var view = new AlbumView({ model: album });
view.$el.appendTo("body");
```

Voila! Your view now has automatic model binding (DOM will update as models are 
    updated).

Features
--------

 * Simple one-way and two-way binding automatically updates your DOM when your 
 model changes (and vice versa).

 * Collections are listened to, too, and the DOM will update when collections 
 do.

 * Partials are a feature of the library, rather than implemented using 
 workarounds.

 * Helpers are also a feature of the library.

 * You can manage Backbone sub-views easily.

 * You can use Livemarkup.js with [fancy HTML syntax 
 compilers](#fancy-syntax-compilers).

 * The directives in `{{ ... }}` are written in JavaScript, not some custom-made 
 DSL. That means you can do whatever it is you want.

 * Made for Backbone.js, but can be used without it. (not sure why you'd wanna, 
     though)

Syntax
------

### Text nodes

Show any text by enclosing a JavaScript expression in `{{ ... }}`.

``` html
Hello {{ model.get('name') }}

Hello {{ view.getName() }}
```

### HTML

To display raw HTML (without escaping), use `{{{ ... }}}`:

``` html
Hello {{{ view.getHTML() }}}
```

### Model binding

To update a certain node and its descendants when a model event happens:

``` html
<span l-listen="change:first_name change:last_name">
  Hello {{ model.getFullName() }}
</span>
```

If you need to listen to the view using the `l-listen="object#event"` syntax:

``` html
<span l-listen="view#refresh">
  ...
</span>
```

Since most events you'll listen to are `change` events in the `model`, there's a 
shorthand you can use called `l-onchange`. The example above can be rewritten as:

``` html
<span l-onchange="first_name last_name">
  Hello {{ model.getFullName() }}
</span>
```

This means when model attributes are changed, the DOM will update automatically:

``` javascript
/* JavaScript */
model.set({ first_name: 'John', last_name: 'Smith' });
```

### Attributes

You can use `{{ ... }}` in attributes:

``` html
<a href="{{ model.getPermalink() }}">
```

As with all directives, you can have it listen to updates on a given event or 
attribute:

``` html
<a href="{{ model.getPermalink() }}" l-onchange='permalink'>
```

Another example:

``` javascript
/* in the model: */
  getCssClass: function() {
    return this.isDone() ? 'done' : 'not-done';
  }
```

``` html
<div class='todo-item {{ model.getCssClass() }}'>
```

### Two-way binding

You can make a form DOM element update a model when it's changed, and have it 
change then a model updates. The result is that the form element and the model 
will be in sync.

This example below will update use `.set('description', value)` to update the 
model, and `.get('description')` to update the form:

``` html
<label>Description:</label>

<textarea l-bindto='description'>
</textarea>
```

Works with checkboxes, too. This one will set a boolean value (ie, 
    `set('active', true|false)`):

```
<input type='checkbox' l-bindto='active'>
```

Or radios:

``` html
<label>
  <input type='radio' name='fruit' l-bindto='fruit' value='apple'>
  Apple
</label>
<label>
  <input type='radio' name='fruit' l-bindto='fruit' value='orange'>
  Orange
</label>
```

Or simplify that as:

``` html
<div l-each='model.getFruitOptions() as name, value'>
  <label>
    <input type='radio' name='fruit' l-bindto='fruit' l-attr-value='value'>
    {{ name }}
  </label>
</div>
```

### Loops

To loop through collections or arrays, use `l-each='___ as ___'`:

``` html
<ul l-each="model.people as person">
  <li>
    <b>{{ person.name }}</b>
  </il>
</ul>
```

Supports Backbone collections, and plain arrays.

If you pass a collection, it will automatically listen to collection `add` 
`reset`, and `sort` events.

``` javascript
// In JavaScript
model.people.sort();    /* DOM will update */

model.people.add({ name: "Jim McBean" });

model.people.remove(...);
```

to loop though objects, use `l-each='____ as __, __'` to iterate through each 
key/value:

``` html
<div l-each="model.synonyms as key, value">
  <dl>
    <dt>{{ key }}</dt>
    <dd>{{ value }}</dd>
  </dl>
</div>
```

### If blocks

Make certain blocks appear only when a given condition is fulfilled:

``` html
<div l-if="model.isPremium()">
  Premium Member
</div>
```

You can combine this with `l-listen` or `l-onchange`:

``` html
<div l-if="! model.get('is_published')" l-onchange="is_published">
  <h3>Publish options</h3>

  <label>
    Publish date:
    <input type='text' name='date'>
  </label>
</div>
```

### Partials

For complex templates, you can make sub-templates (called partials). Use
`{{% render partial }}` to render another template within a template.

``` html
<ul l-each="model.getPeople() as person">
  {{% render partial='people/person_card' person='person' }}
</ul>
```

### Subviews

A view may be used to manage a collection of other views. Use
```{{% render subview }}``` to automatically manage these views.
  
``` html
<ul l-each="model.getPeople() as person">
  {{% render subview='PersonCardView' person='person' }}
</ul>
```

Then in your view, populate `this.subviewClasses`:

``` javascript
PeopleView = Backbone.View.extend({
  subviewClasses: {
    PersonCardView: PersonCardView
  }
});
```

Then you can use `view.subviews` to access them. It is a hash with the keys 
being the model `cid`s, and the values being the view objects.

``` javascript
// In your view code:
_.each(this.subviews, function(view, cid) {
});
```

### Helpers

Register helpers in the `LM.Helpers` object.

``` javascript
_.extend(LM.Helpers, {
  money: function(amount) {
    return "$" + amount;
  }
});
```

You can the use it in your templates:

``` html
<div class='account-balance' l-onchange='balance'>
  Your current balance is
  {{ money(model.get('balance')) }}
</div>
```

### Coffeescript

Don't like JavaScript? Use CoffeeScript syntax is your markup if you like.
(Under debate: this probably isn't needed)

``` html
{{% set lang='coffeescript' }}
```

Registering templates
---------------------

For you to use templates, you must first register them using `LM.register()`.
This works like the `JST` object commonly used in Rails or Jammit. The reasons
for this is:

 * So LM knows where to look for partials
 * Uh... don't know what else

Register templates using `LM.register(name, markup);`.

``` javascript
LM.register('people/person_card',
  "<div>" +
  "Hello {{ model.get('name') }}" +
  "</div>"
);
```

Better with CoffeeScript, actually, since it supports multi-line strings:

``` coffeescript
LM.register 'people/person_card', '''
  <div>
    Hello {{ model.get('name' }}
  </div>
'''
```
  
### Inline templates

Or you can use inline templates by defining them in
`<script type='text/lm-template' id='____'>`. The ID will be the name of the template.

``` html
<script type='text/lm-template' id='person_card'>
  <div>
    Hello {{ model.get('name') }}
  </div>
</script>
```

``` javascript
// Register that template:
LM.register($("#person_card"));

// Or register all templates:
LM.register($("script[type='text/lm-template']"));
```

Backbone.js integration
-----------------------

Integrates automatically with Backbone views.

Simply call `LM.extendView()` to inject it into `Backbone.View`, or any view
class you wish.

``` javascript
LM.extendView(Backbone.View);
```

This provides `Backbone.View` and it's subclasses the following conveniences:

 * Enable LM use in a view by defining `templateName` attribute. This is the
 name of the LM template to use (registered via `LM.register()`).
 * The view's element is automatically updated with the contents of the
 rendered template.
 * When views are instantiated, LM automatically sets `this.template` to a
 template object.
 * In the template, the view is automatically available as `view`.
 * If the view has a model attribute, it's available in the template as `model`.

### Example

``` javascript
LM.register('books/show', [
  '<article class="book">'
    '<h2 l-onchange="title">'
      '{{ model.get("title") }}'
    '</h2>'
  '</article>'
].join(''));
```

``` javascript
var BookView = Backbone.View.extend({
  templateName: 'books/show'
});

var book = new Book({ title: "A Study In Pink", author: "Sir ACD" });
var view = new BookView({ model: book });
view.render();

view.$el   //=> "<article class='book'>...</article>"
```

### Overriding render()

If you're to override `render()`, be sure to call the `renderTemplate()` method yourself.

``` javascript
var BookView = Backbone.View.extend({
  templateName: 'books/show',

  render: function() {
    this.renderTemplate();
  }
});
```

or in CoffeeScript, calling `super` might be simpler:

``` coffeescript
class BookView extends Backbone.View
  render: ->
    super()
```

Bare API
--------

LiveMarkup can be used outside Backbone.js.

When using outside Backbone.js, you can use `LM.get()` and template API:

``` javascript
var view = /* ... */;
var model = /* ... */;

var template = LM.get('people/person_card');
template.bind('view', view);
template.bind('model', model);

template.appendTo($("#parent_div"));

template.render();
```

Fancy syntax compilers
----------------------

LiveMarkup's syntax is intentionally made using HTML attributes and HTML text 
nodes so compile-to-HTML markup languages can be used to generate LM templates.  
To illustate with [Jade]:

``` coffeescript
LM.register 'books/show', jade.compile
  '''
  article.book
    label Book title:
    {{ title }}

    div(l-if='model.isBestseller()')
      strong Best seller!
  '''
```

[Jade]: http://jade-lang.com
[Backbone.js]: http://backbonejs.org
