# Understanding the Handlebars (`.hbs`) templates

The preferred way to write the renderers for UI5 Web Components (and supported directly by the build tools) is to use standard Handlebars templates with some additional custom syntax.

## Table of contents
1. [Handlebars compilation](#compilation)
2. [Design goals](#design_goals)
3. [The context in `.hbs` files](#context)
	- [Global context](#context_global)
	- [Context in loops](#context_loops)
	- [Accessing the global context from loops](#context_loops_accessing)
4. [The `.hbs` syntax](#syntax)
	- [Bindings](#syntax_bindings)
	- [Conditions](#syntax_conditional)
	- [Loops](#syntax_loops)
	- [Property assignment (the `.` prefix)](#syntax_dot)
	- [Boolean attribute assignment (the `?` prefix)](#syntax_question_mark)
	- [Event handlers assignment (the `@` prefix)](#syntax_at)
	- [Style maps](#syntax_style_maps)
	- [Class maps](#syntax_class_maps)
	- [Partials](#syntax_partials)
	- [Include](#syntax_include)
5. [Using the `slot` element](#slots)
	- [Rendering slots](#slots_rendering)
	- [Individual slots](#slots_individual)


## 1. Handlebars compilation <a name="compilation"></a>

[Handlebars](https://handlebarsjs.com/guide/#simple-expressions) templates (`.hbs`) are compiled during build/development to [lit-html](https://lit.dev/docs/v1/lit-html/introduction/) templates (`.lit.js`) and the lit templates are what's actually executed during runtime.

Example:

The following `src/Demo.hbs` template

```html
<button>{{text}}</button>
```

will be compiled to `dist/generated/templates/DemoTemplate.lit.js` with the following content:

```js
import { html, svg, repeat, classMap, styleMap, ifDefined, unsafeHTML, scopeTag } from "@ui5/webcomponents-base/dist/renderer/LitRenderer.js";
const block0 = (context, tags, suffix) => html`<button>${ifDefined(context.text)}</button>`;
export default block0;
```

and later tree-shaken by the bundler and bundled along with the rest of the component's code.

Therefore, the `.hbs` file is there just for convenience, the end result will always be optimized lit-html.

## 2. Design goals of the Handlebars templates <a name="design_goals"></a>

 - **Declarative**: write HTML in a form as close as possible to what will eventually be in the DOM (rather than writing template functions directly).
 - **Abstract**: the template could be compiled to other formats in the future (not just lit-html) so it should only use universal concepts and no lit-specific features.
 - **Separation of concerns**: the template must be as simple as possible with no complex expressions or calculations - variables that control structures (for example, `{{#if` statements) should be precalculated. 
 
For these reasons, we would suggest you use `.hbs` templates and have them compiled to lit-html, instead of directly writing `lit-html` renderers, although that's also possible if you prefer so.

## 3. The context in `.hbs` files <a name="context"></a>

### 3.1 Global context <a name="context_global"></a>

The context in the `.bhs` file is the **web component instance**, and you do not have to write the `this` keyword (although you can).
Therefore, you can directly use metadata entities (property, slot, event names) or any other Javascript property on the component directly:

In the `Demo.js` file:

```js
this.age = 30;
this.fullName = `${this.name} ${this.lastName}`;
```

In the `Demo.hbs` file you can just use them directly:

```html
<p>{{fullName}}</p>
<p>{{age}}</p>
```

The following code will have exactly the same result:

```html
<p>{{this.fullName}}</p>
<p>{{this.age}}</p>
```

but `this` is optional, so it's almost never used.

### 3.2 Context in loops  <a name="context_loops"></a>

In a loop the context is always the current item, and not the component itself. 

Example:

In the `Demo.js` file:

```js
this.items = [
	{
		id: "item1",
		posinset: 1,
		setsize: 5,
		text: "Item 1"
	},
	{
		id: "item2",
		posinset: 2,
		setsize: 5,
		text: "Item 2"
	}
]
```

In the `Demo.hbs` file:

```html
{{#each items}}
	<div id="{{id}}"
		 role="option"
		 aria-posinset="{{posinset}}"
		 aria-setsize="{{setsize}}"
	>{{text}}</div>
{{/each}}
```

Again, you can use the `this` keyword, but it's not necessary. The following code will be the same as the one above:
```html
{{#each items}}
	<div id="{{this.id}}"
		 role="option"
		 aria-posinset="{{this.posinset}}"
		 aria-setsize="{{this.setsize}}"
	>{{this.text}}</div>
{{/each}}
```

The only use case where you must use the `this` keyword is when you want to refer to the looped over item directly (and not its properties).

Example:

```html
{{#each items}}
	<div id="{{id}}"
		 .item="{{this}}"
	>{{text}}</div>
{{/each}}
```

Here, each `div` inside the loop gets assigned an `item` property that points to the respective item from the array we're looping over.

Here's another example for the `this` keyword:

In the `Demo.js` file:

```js
this.numbers = [
	[1, 2, 3],
	[4, 5, 6]
];
```

In the `Demo.hbs` file:

```html
{{#each numbers}}
	<div>
		{{#each this}}
			<span>{{this}}</span>
		{{/each}}
	</div>
{{/each}}
```

The result in the DOM would be:

```html
<div><span>1</span><span>2</span><span>3</span></div>
<div><span>4</span><span>5</span><span>6</span></div>
```

In this example, the first usage of `this` (in the nested `#each`) is the nested array (for example, `[1, 2, 3]`), and the second usage of `this` inside the `span` is the number itself.

### 3.3 Accessing the global context from loops  <a name="context_loops_accessing"></a>

You can access the global context inside loops with the "one-level-up" expression: `../`

Example:

In the `Demo.js` file:

```js
this.name = "John Smith";
this.items = [
	{
		id: "item1"
	},
	{
		id: "item2"
	}
]
```

In the `Demo.hbs` file:

```html
{{#each items}}
	<div id="{{id}}">{{../name}}</div>
{{/each}}
```

In this example, even though we're looping over an item from the array, we can still access the global context and use the `name` property of the web component instance.


## 4. The `.hbs` syntax <a name="syntax"></a>

You can use the following features when writing `.hbs` templates:

### Bindings <a name="syntax_bindings"></a>

You can access any property from the context (generally the web component instance) in your `.hbs` template.

In the `Demo.js` file:

```js
this.tooltip = "Some tooltip";
this.txt = "Some text";
```

In the `Demo.hbs` file:

```html
<button title="{{tooltip}}">{{txt}}<button/>
```

*Note:* You must always create valid HTML, so you can only use bindings for attribute values or text nodes.

For example, the following is **not allowed**:

```html
<{{tag}} {{attr}}="Hello">This will not compile</{{tag}}>
```

You can access object properties:

In the `Demo.js` file:

```js
this.person = {
	name: "John",
	lastName: "Smith"
}
```

In the `Demo.hbs` file:

```html
<p>{{person.name}} {{person.lastName}}</p>
```

but you cannot use expressions inside `.hbs` templates. The following is **not allowed**:

```html
<p>{{person.name + " " + person.lastName}}</p>
```

Instead, you should precalculate the required value in the `.js` file and use it directly in the template:

In the `Demo.js` file:

```js
get fullName() {
	return `${this.person.name} ${this.person.lastName}`;
}
```

In the `Demo.hbs` file:

```html
<p>{{fullName}}</p>
```


### Conditions <a name="syntax_conditional"></a>

You can use `if`, `else` and `unless` to create conditions.

Examples:

```html
{{#if hasText}}
	<label class="ui5-badge-text"><bdi><slot></slot></bdi></label>
{{/if}}
```

or

```html
{{#if hasText}}
	<label class="has-text"><span>{{text}}</span></label>
{{else}}
	<label class="empty-label"></label>
{{/if}}
```

or

```html
{{#unless _isPhone}}
	<p>Some content</p>
{{/unless}}
```

You can chain if-else-if, as follows:

```html
{{#if hasImage}}
	<slot></slot>
{{else if icon}}
	<ui5-icon class="ui5-avatar-icon" name="{{icon}}" accessible-name="{{accessibleNameText}}"></ui5-icon>
{{else if initials}}
	<span class="ui5-avatar-initials">{{validInitials}}</span>
{{/if}}
```

Again, you cannot use expressions, so the following is **not allowed**:

```html
{{#if person.access === "admin" }}
	<p>Show admin functionality</p>
{{/if}}
```

Instead, you should have a precalculated value in your `.js file`, for example:

In `Demo.js`:

```js
get isAdmin() {
	return this.person.access === "admin";
}
```

and then use this value in `Demo.hbs`:

```html
{{#if isAdmin }}
	<p>Show admin functionality</p>
{{/if}}
```

### Loops <a name="syntax_loops"></a>

You can use `each` to loop over arrays.

In the `Demo.js` file:

```js
this.items = [
	{
		id: "item1",
		posinset: 1,
		setsize: 5,
		text: "Item 1"
	},
	{
		id: "item2",
		posinset: 2,
		setsize: 5,
		text: "Item 2"
	}
]
```

In the `Demo.hbs` file:

```html
{{#each items}}
	<div id="{{id}}"
		 role="option"
		 aria-posinset="{{posinset}}"
		 aria-setsize="{{setsize}}"
	>{{text}}</div>
{{/each}}
```

See the previous section (especially the "Context in loops" part) for more examples and the meaning of the `this` keyword in loops.

### Property assignment (the `.` prefix) <a name="syntax_dot"></a>

The `.` prefix allows you to bind by property, rather than by attribute.

Consider the following example:

```js
this.id = "myId";
this.someString = "Some data";
this.item = {
	a: 1,
	b: 2
};
this.text = "Some text";
```

```html
<div
		id="{{id}}"
		data-info="{{someString}}"
		.item="{{myItem}}"
>
	{{text}}
</div>
```

While `data-info` is set as an attribute (default assignment), `item` is set as a property due to the `.` used.
The result in the DOM would be:

```html
<div id="myId" data-info="Some data">Some text</div>
```

There would be no `item` in the DOM at all, but the following code:

```js
document.getElementById("myId").item
```

would return the `item` object because it was set as a property.

### Boolean attribute assignment (the `?` prefix) <a name="syntax_question_mark"></a>

The `?` prefix signifies that an attribute must not be set in DOM at all, if the bound value is falsy.

Consider the following example:

```js
this._id = "myCB";
this.checked = false;
this.readonly = false;
this.disabled = false;
```

```html
<input
	id="{{_id}}-CB"
	type='checkbox'
	?checked="{{checked}}"
	?readonly="{{readonly}}"
	?disabled="{{disabled}}"
	tabindex="-1"
	aria-hidden="true"
	data-sap-no-tab-ref
/>
```

Since the `checked`, `readonly`, and `disabled` attributes are all `Boolean`, they must not be in the DOM if we want the `<input>` to be interactive.

The output in DOM would be:

```html
<input
	id="myCB-CB"
	type='checkbox'
	tabindex="-1"
	aria-hidden="true"
	data-sap-no-tab-ref
/>
```

All attributes that had the `?` prefix and were bound to a falsy value are gone from DOM.

However, if you did not use the `?` prefix:

```html
<input
	id="{{_id}}-CB"
	type='checkbox'
	checked="{{checked}}"
	readonly="{{readonly}}"
	disabled="{{disabled}}"
	tabindex="-1"
	aria-hidden="true"
	data-sap-no-tab-ref
/>
```

even though `checked`, `readonly`, and `disabled` are equal to `false`, the resulting DOM would be:

```html
<input
	id="myCB-CB"
	type='checkbox'
	checked=""
	readonly=""
	disabled=""
	tabindex="-1"
	aria-hidden="true"
	data-sap-no-tab-ref
/>
```

which is not what we want, since boolean HTML attributes don't need to have a value at all to be considered set, only their presence is required.
Therefore, always bind boolean attributes with `?`. 

### Event handlers assignment (the `@` prefix) <a name="syntax_at"></a>

You can bind events as follows:

In the `Demo.js` file:

```js
this.onClick = event => {};
```

In the `Demo.hbs` file:

```html
<button @click="{{onClick}}"></button>
```

### Style maps <a name="syntax_style_maps"></a>

Style maps are an easy and useful tool to apply multiple styles to an element dynamically.

In order to use a style map in your `.hbs` template you must bind a `styles` property (or as in the next example, a getter called `styles`).
Any binding to a `styles` object on a `style` attribute will be treated as a style map.

In the `Demo.js` file:

```js
get styles() {
	return {
		root: {
			display: this.isBlock ? "block" : "inline",
			width: `${this.x}px`,
			height: `${this.y}px`
		},
		footer: {
			backgroundColor: this.bgColor 
		}
	}
}
```

In the `Demo.hbs` file:

```html
<div style="{{styles.root}}">
	Some content
	<footer style="{{styles.footer}}"></footer>
</div>
```

After the following code is run, both the `div` and the `footer` will have the respective CSS styles applied to them.

**Important:** do not build styles manually. Always use style maps as they are CSP-compliant and they will not build style strings and assign them,
but will use Javascript APIs to apply each style/CSS variable separately.

The following is an anti-pattern and is **not allowed** in the latest version of the handlebars-to-lit compiler:

```js
this.display = "block";
this.styles = "display: none; visibility: hidden";
```

```html
<div style="display: {{display}}"></div>
<div style="{{styles}}"></div>
```

In the first example we build a style value manually, and in the second example we pass hard-coded styles as a string. None of these are CSP-compliant.
The correct way would be to pass objects (as in the first example), in which case a style map will be used.

### Class maps <a name="syntax_class_maps"></a>

Class maps are an easy tool to set multiple classes to an element - either conditionally, or unconditionally.

In order to use a class map in your `.hbs` template you must bind a `classes` property (or as in the next example, a getter called `classes`):

```js
get classes() {
	return {
		main: {
			"ui5-demo-main": true,
			"ui5-demo-mobile": isPhone()
		},
		content :{
			"ui5-content-wide": this.width > 1024	
		},
		section: {
			"ui5-section": true,
			"ui5-section-with-items": this.items.length > 0,
			"ui5-section-desktop": !isPhone() && !isTablet()
		}
	}
}
```

```html
<article class="{{classes.main}}">
	<div class="{{classes.content}}"></div>
	<section class="{{classes.section}}"></section>
</article>
```

Here, all 3 HTML elements will have their classes applied based on the conditions in the definition of the class map. Some entries in the class map
are unconditional (`ui5-demo-main` and `ui5-section`) so these classes will always be set, however the rest are going to be set only if certain criteria are met.

### Partials <a name="syntax_partials"></a>

You can use partials to reuse code in `.hbs` templates:

You can define a partial with `{{#*inline "NAME"}}` and use it with `{{>NAME}}` where `NAME` is the name of the partial.

Consider the following example:

```html
<div>
	{{>valueStateMessage}}
</div>
<p>Some more content</p>
<div>
	{{>valueStateMessage}}
</div>

{{#*inline "valueStateMessage"}}
	{{#if msg}}
		<span>{{msg}}</span>
	{{/if}}
{{/inline}}
```

Here we define some common code in the `valueStateMessage` partial and use it twice within the template.

Partials are very often used to define **hooks** - extension points for other components.

Example:

In `Demo.hbs`:

```html
<section>
	<span class="first-fe" data-ui5-focus-trap tabindex="0" @focusin={{forwardToLast}}></span>

	{{> beforeContent}}

	<div style="{{styles.content}}" class="{{classes.content}}"  @scroll="{{_scroll}}" part="content">
		<slot></slot>
	</div>

	{{> afterContent}}

	<span class="last-fe" data-ui5-focus-trap tabindex="0" @focusin={{forwardToFirst}}></span>
</section>


{{#*inline "beforeContent"}}{{/inline}}

{{#*inline "afterContent"}}{{/inline}}
```

Here we define two empty partials (`beforeContent` and `afterContent`) for others to implement.

### Include <a name="syntax_include"></a>

You can include other `.hbs` files with `{{>include "PATH_TO_FILE"}}` where `PATH_TO_FILE` is a relative or absolute path to the `.hbs` file you want to include.

Example:

```html
{{>include "./Demo.hbs"}}
```

Paths to `.hbs` files from other `node_modules/` libraries are also supported.

Example:

```html
{{>include "@ui5/webcomponents/src/Popup.hbs"}}
```

The most common use case for `{{>include}}` is to include an `.hbs` file that has extension points (hooks) and implement them. Given the example from the previous section (about Partials), consider the following:

In `Demo2.hbs`:

```html
{{>include "./Demo.hbs"}}

{{#*inline "beforeContent"}}
	<span>Implementation here</span>
{{/inline}}

{{#*inline "afterContent"}}
	<span>Another implementation here</span>
{{/inline}}
```

Then the `Demo2` component will use the `.hbs` file of the `Demo` component, however with its own version of its partials.

## 5. Using the `slot` element <a name="slots"></a>

### Rendering slots <a name="slots_rendering"></a>

The [slot](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/slot) element allows you to render children, nested in your web component, in a desired place in the shadow DOM.
You should render each slot, defined in your component's metadata (see [Understanding UI5 Web Components Metadata](./03-understanding-components-metadata.md)), somewhere in the `.bhs` template.

To render the default slot simply render a `slot` tag:

```html
<slot></slot>
```

and to render a named slot:

```html
<slot name="tabs"></slot>
```

Here's a real-world example of a "page" component:

In `Page.js` (metadata object):

```js
slots: {
	header: {
		type: HTMLElement
	},
	"default": {
		type: HTMLElement,
		propertyName: "content"
	},
	footer: {
		type: HTMLElement
	}
}
```

In `Page.hbs`:

```html
<div class="ui5-page-root">
	<header class="ui5-page-header-root" id="ui5-page-header">
		<slot name="header"></slot>
	</header>

	<section part="content" class="ui5-page-content-root" style="{{styles.content}}">
		<slot></slot>
	</section>

	<footer class="ui5-page-footer-root" style="{{styles.footer}}">
		<slot name="footer"></slot>
	</footer>
</div>

```

We render 3 `slot` elements - a default slot (unnamed) and 2 named slots - respectively with `name` equal to `header` and `footer`.

All children, passed to the component, with no `slot` attribute will then be rendered by the browser where the default `<slot></slot>` is,
and all children with attributes `slot="header"` / `slot="footer"` will be rendered where the respective named `slot` is.

### Individual slots <a name="slots_individual"></a>

All children, assigned to a certain `slot`, are rendered by the browser next to each other in the exact order in which they were passed to the component.
Sometimes, however, each child must be placed separately in the shadow root, potentially wrapped in other HTML elements, to satisfy the UX design of the component. 

The `individualSlots` slot metadata configuration setting (see [Understanding UI5 Web Components Metadata](./03-understanding-components-metadata.md)) allows you to have a separate physical slot for each child belonging to a certain slot.

However, setting `individualSlots: true` in the metadata configuration only creates an `_individualSlot` property on each element belonging to the slot, but does not create any slots automatically.
The individual slots must be explicitly rendered by the developer in the `.hbs` template.

To do so, simply render a `slot` with a `name` property equal to the `_individualSlot` value for each child.

Here's an example:

In `Demo.js` (metadata object):

```js
{
	slots: {
		"default": {
			type: HTMLElement,
			propertyName: "items",
			individualSlots: true
		}
	}
}
```

Since `propertyName` is set to `items`, the children of the default slot will be accessible on the web component instance with `this.items`;
and since `individualSlots` is set to `true`, every child in `this.items` (every child slotted in the default slot) will have an `_individualSlots` property created by the framework. 

In `Demo.hbs` you must render a slot for each child with `name` equal to the `_individualSlot` property value for this child:

```html
{{#each items}}
	 <div class="item-wrapper">
		<slot name="{{_individualSlot}}"></slot>
	</div>
{{/each}}
```

The resulting DOM from the loop above will look like this:

```html
<div class="item-wrapper"><slot name="items-1"></slot></div>
<div class="item-wrapper"><slot name="items-2"></slot></div>
<div class="item-wrapper"><slot name="items-3"></slot></div>
```

This allows you to have arbitrary DOM around each child and implement complex UX design, otherwise impossible if all children were just normally rendered next to each other in a single slot.
