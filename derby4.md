# Derby 0.6: Introduction to components

## General information

In the context of Derby 0.6, components are derby templates moved out into a separate scope. This is an index.html file from one of the previous examples (TODO list). 

<div class='code-caption'>index.html</div>

```html
<Body:>

  <h1>Todos:</h1>
  
  <view name="new-todo"></view>
  
  <!--list output -->
  
<new-todo:>  
  <form>
    <input type="text">
    <button type="submit">Add Todo</button>
  </form>
```  

`Body:` and `new-todo:` are templates. Let’s make `new-todo:` a component. In order to do that derby application has to register it: 

```js
app.component('new-todo', function(){});
```

So, we need to pass a function as a second argument, which is going to handle this component. 
Let’s bind input to a reactive variable and create an on-submit event handler. That is how the code would look like if there were no components:

```html
<new-todo:>  
  <form on-submit="addNewTodo()">
    <input type="text" value="{{_page.new-todo}}">
    <button type="submit">Add Todo</button>
  </form>
```  

```js
app.proto.addNewTodo = function(){
  //...
}
```

Since there are no components, certain problems occur:
 
1. global scope is bloated
2. addNewTodo function is added to app.proto -- we do NOT want spaghetti code in a big application

And here `new-todo:` is a component: 

```html
<new-todo:>  
  <form on-submit="addNewTodo()">
    <input type="text" value="{{todo}}">
    <button type="submit">Add Todo</button>
  </form>
```  

```js
app.component('new-todo', NewTodo);  

function NewTodo(){}

NewTodo.prototype.addNewTodo = function(todo){
  // note that the model is "scoped"
  // it can see only local variables (not global)
  var todo = this.model.get('todo');
  //...
}
```

Now, there is a separate scope in the `new-todo:` template: all global collections and `_page` are invisible. `todo` is local and it is not accessible in the global scope. Encapsulation is a great thing. As for `addNewTodo` (handler function), it is inside `NewTodo` class and this way it doesn’t bloat the application. 

Derby components are ui-elements, which are used to hide the implementation details of the specific visual block. Note that components are not implied to load data. Data has to be loaded on a controller level.
 
What is the interface of components? How do we pass arguments to the components and how do we get results?

We pass arguments just like to a regular template via attributes and as an enclosed html content.
Results are returned by the use of events. Let’s set a class to the component and give a placeholder to the input. We are going to get the input data via the event handler:

<div class='code-caption'>index.html</div>

```html
<Body:>

  <h1>Todos:</h1>
  
  <view 
    name="new-todo" 
    placeholder="Input new Todo" 
    inputClass="big"
    on-addtodo="list.add()">
  </view>
  
  <view name="todos-list" as="list"></view>

<new-todo:>  
  <form on-submit="addNewTodo()">
    <input type="text" value="{{todo}}" placeholder="{{@plaсeholder}}"
           class="{{@inputClass}}">
    <button type="submit">Add Todo</button>
  </form>

<todos-list:>
  <!-- list output -->
```

```js
app.component('new-todo', NewTodo); 
app.component('todos-list:', TodosList); 

function NewTodo(){}

NewTodo.prototype.addNewTodo = function(todo){
  var todo = this.model.get('todo');
  // create an event which is accessible externally
  // (where we call the component)
  this.emit('addtodo', todo);
}

function TodosList(){};

TodosList.prototype.add = function(todo){
  // That’s how the event got from one component
  // to another. That’s right, the component which 
  // handles the list is going to add new elements
}
```

The component takes 2 arguments: `placeholder` and `inputClass` and returns `addtodo`. `addtodo` is an event which we redirect to `todos-list` component, where it is handled by `TodosList.prototype.add`. Note that when we created a component instance, we assigned alias list to it using a keyword `as`. This way we could write `list.add()` in `on-addtodo` handler. 
As a result, `new-todo` is isolated but still todos-list component handles todos list. This way, the responsibilities are separated. 

## Interface component

Ability to pass arguments to components is inherited from templates, so most of functionality is similar. Templates (like components) in derby html files are similar to functions. We can also call templates from other templates. 

Template (component) declaration syntax:
 
```html 
<name: ([element="element"] [attributes="attributes"] 
        [arrays="arrays"])>
```

`attributes`, `element` and `arrays` attributes are optional.
 
### element attribute

By default template declaration and template call looks like this: 

```html
<!-- template declaration -->
<nav-link:>
  <!-- current page url is in  $render.url -->
  <li class="{{if $render.url === @href}}active{{/}}">
    <a href="{{@href}}">{{@caption}}</a>
  </li>  

<!-- template call from Body: template -->
<view name="nav-link" href="/" caption="Home"></view>
```

It is not always convenient to do it this way. For example, we may want to call the template with a specific name not via view tag  but using template name as a tag name. In this case, we are going to need an element attribute: 

```html
<!-- declare a template allowing it to be called as a nav-link tag -->
<nav-link: element="nav-link">
  <li class="{{if $render.url === @href}}active{{/}}">
    <a href="{{@href}}">{{@caption}}</a>
  </li>  

<!-- call nav-link from Body: template -->
<nav-link href="/" caption="Home"></nav-link>
```

Or we can do it this way: 

```html
<nav-link href="/" caption="Home"/>
```

As you can see, there is no closing tag because it has no contents. What does this mean?
It is an inexplicit content parameter. 

When we call a template, we use either view tag or a tag named by element attribute:

```html
<!-- this way -->
<view name="nav-link" href="/" caption="Home"></view>
<!-- or this way -->
<nav-link name="nav-link" href="/" caption="Home"></nav-link>

<!-- template declaration -->
<nav-link: element="nav-link">
  <li class="{{if $render.url === @href}}active{{/}}">
    <a href="{{@href}}">{{@caption}}</a>
  </li>
```    

It turns out that when calling a template we can put some text or enclosed html (which will be passed inside the template by an inexplicit parameter `@content`) between the opening and closing tag. Let’s replace caption with `@content`:

```html
<!-- this way -->
<view name="nav-link" href="/">Home</view>
<!-- or this way -->
<nav-link name="nav-link" href="/">Home</nav-link>
<!-- or even this way -->
<nav-link name="nav-link" href="/">
  <span class="img image-home">
    Home
  </span>
</nav-link>

<!-- template declaration -->
<nav-link: element="nav-link">
  <li class="{{if $render.url === @href}}active{{/}}">
    <a href="{{@href}}">{{@content}}</a>
  </li>  
```

It is very convenient because it allows to hide the details and simplify top level code. 

### attributes attribute

Imagine we have a task: html code which is passed to the template inside the template can’t be a solid block inserted into a specific place. Suppose, there is a widget which has a header, a footer and the main content. That’s how we call it: 

```html
<widget>
  <header><-- contents --></header>
  <footer><-- contents --></footer>
  <body><-- contents --></body>
</widget>
```

Inside the widget template there is some complex html and we need to be able to insert these 3 blocks separately (that is, header, footer and body). To do this we need attributes: 

```html
<widget: attributes="header footer body">
   <!-- complex html -->
   <!-- complex html -->
     {{@header}}
   <!-- complex html -->
   <!-- complex html -->
     {{@body}}
   <!-- complex html-->
     {{@footer}}
   <!-- complex html →
```

Instead of body we could have used content since everything that is listed in attributes goes to the content. 

```html
<Body:>
  <widget>
    <h1>Hello<h1>
    <header><-- contents --></header>
    <footer><-- contents --></footer>
    <p>text</text>
  </widget>

<widget: attributes="header footer">
   <!-- complex html -->
   <!-- complex html -->
     {{@header}}
   <!-- complex html -->
   <!-- complex html -->
     
     {{@content}}  <!-- tags h1 and p get here -->
     
   <!-- complex html -->
     {{@footer}}
   <!-- complex html -->
```

Note that everything that was listed in attributes must appear in the internal block (which we insert into the template) only once. Let’s use template attribute arrays: 

```html
<dropdown: arrays="option/options">
  <!-- complex html -->
  {{each @options}}
    <li class="{{this.class}}">
      {{this.content}}
    </li>
  {{/}}
  <!-- complex html -->
```

We set 2 names when declaring the template: `arrays=”option/options”`:

1. option &mdash; name of html element inside dropdown on component call
2. options &mdash; name of the array with the elements inside the template (the elements inside this array are going to be objects, and all the option attributes will be fields inside these objects, whereas the contents of the object will be a content field). 

## javascript part of component

The template becomes a component if there is a function-constructor registered for this component. 

```html
<new-todo:>  
  <form on-submit="addNewTodo()">
    <input type="text" value="{{todo}}">
    <button type="submit">Add Todo</button>
  </form>
```  

```js
app.component('new-todo', NewTodo);  

function NewTodo(){}

NewTodo.prototype.addNewTodo = function(todo){

  var todo = this.model.get('todo');
  // ...
}
```

The component has predetermined functions which are called at certain points, that is, create, init and destroy. 

### init()

init function is called both on server and client side before rendering the component. Its purpose is to initialize the internal model of the component, set default values, create references.

```js
// https://github.com/codeparty/d-d3-barchart/blob/master/index.js 
function BarChart() {}

BarChart.prototype.init = function() {
  var model = this.model;
  model.setNull("data", []);
  model.setNull("width", 200);
  model.setNull("height", 100);

  // ...
};
```

### create()

It is called only on client side before rendering the component. We need it to register event handlers, plug in client libraries, subscribe for data (but it’s antipattern), run reactive functions of the component, etc. 

```js
// https://github.com/codeparty/d-d3-barchart/blob/master/index.js 
function BarChart() {}

BarChart.prototype.init = function() {
  var model = this.model;
  model.setNull("data", []);
  model.setNull("width", 200);
  model.setNull("height", 100);

  // ...
};
```

### destroy()

It is called when the component is being destroyed. We need it for final clean-up:

1. release memory 
2. stop reactive functions
3. remove client libraries. 

this inside the component handlers has: `model`, `app`, `dom` (except `init`), all aliases to dom-elements and components created within the component, parent-reference to parent-component and everything we put inside prototype function-constructor of the component. 

So, `this.model` in the component has model of this particular component only. Here, model is a scoped one. If you need to address the global scope, use `this.model.root` or `this.app.model`.

`app` is a derby application instance and we can do a lot of things with it, for example: 

```js
MyComponent.prototype.back = function(){
  this.app.history.back();
}
```

Using dom we can add handlers to DOM-events, e.g.:

```js
// https://github.com/codeparty/d-bootstrap/blob/master/dropdown/index.js
Dropdown.prototype.create = function(model, dom) {
  // Close on click outside of the dropdown
  var dropdown = this;
  dom.on('click', function(e) {
    if (dropdown.toggleButton.contains(e.target)) return;
    if (dropdown.menu.contains(e.target)) return;
    model.set('open', false);
  });
};
```

To understand this example you need to understand that this.toggleButton and this.menu are aliases to DOM-elements, which were declared in the template with the help of as. [Look here](https://github.com/codeparty/d-bootstrap/blob/master/dropdown/index.html#L4-L11)
 
All dom functions (`on`, `once`, `removeListeners`) can take 4 parameters: `type, [target], listener, [userCapture]`. Target is an element which is being handled. If target is not specified, it equals to document.The remaining 3 parameters are similar to parameters of `addEventListener` (`type, listener[, useCapture]`)

Aliases to dom-elements are set inside the template with the keyword as: 

```html
<main-menu:>
  <div as="menu">
    <!-- ... -->
  </div>
```

```js
MainMenu.prototype.hide = function(){
  // for example
  $(this.menu).hide();
}
```

## Moving component outside the application into a separate module

We need to create a folder for the component. We put js, html, css files in this folder. The component is registered in the application with `app.component` function (which has one parameter - function constructor): 

```js
app.component('dropdown', Dropdown);
```

Let’s look at the example:
 
<div class='code-caption'>tabs/index.js</div>

```js
module.exports = Tabs;
function Tabs() {}
Tabs.prototype.view = __dirname;

Tabs.prototype.init = function(model) {
  model.setNull('selectedIndex', 0);
};

Tabs.prototype.select = function(index) {
  this.model.set('selectedIndex', index);
};
```

<div class='code-caption'>tabs/index.html</div>

```html
<index: arrays="pane/panes" element="tabs">
  <ul class="nav nav-tabs">
    {{each @panes as #pane, #i}}
      <li class="{{if selectedIndex === #i}}active{{/if}}">
        <a on-click="select(#i)">{{#pane.title}}</a>
      </li>
    {{/each}}
  </ul>
  <div class="tab-content">
    {{each @panes as #pane, #i}}
      <div class="tab-pane{{if selectedIndex === #i}} active{{/if}}">
        {{#pane.content}}
      </div>
    {{/each}}
  </div>
```  

Take a look at this line:

```js
Tabs.prototype.view = __dirname;
```

Here derby gets component name (it is not specified in the template since index is used). The algorithm is simple: the last segment of the path is taken.
 
Suppose, `_dirname` is `/home/zag2art/work/project/src/components/tabs`. It means that we can access this component as `tabs`, e.g.:

```html
<Body:>
  <tabs selected-index="{{widgets.data.currentTab}}">
    <pane title="One">
      Stuff'n
    </pane>
    <pane title="Two">
      More stuff
    </pane>
  </tabs>
```  

This is how we plug the component in the application: 

```js
app.component(require('../components/tabs'));
```

It is very convenient tо write components as separate modules, e.g.: 
http://www.npmjs.org/package/d-d3-barchart
