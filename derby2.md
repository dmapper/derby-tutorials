# Tutorial. Learning Derby 0.6. TODO list app

![TODO list application](/img/tutorials/derby2/1.png)

We are going to create a TODO list (from TodoMVC project). It’s based on the example created on Angular and we’re going to recreate the functionality on Derby. 
Ok, so let’s see what we’ve got in the example created on  Angular and figure out how it works:
1. we type the new task in the upper text field and it appears in the list after clicking “enter”; 
2. we can delete any task in the list by clicking the “close” button on the right hand side of the task (it appears when we hover the cursor over the task);
3. tasks can be marked as “completed” by clicking  “check” button on the left side of the task (you can also uncheck it);
4. when double-clicking the task it switches to edit mode -  we can edit it, and then click “enter” to update and save;
5. if  a task is completed, the “clear completed” button appears at the bottom on the right - if we click on it completed tasks will be deleted;
6. completed and active tasks are counted (the number is displayed in the status bar at the bottom);
7. there are also 3 links in the status bar (all, active, completed changing url to `#/`, `#/completed` accordingly); by clicking the link we are changing tasks filter: all tasks are displayed, active or completed.

## 1. CSS and HTML file

Since our objective is to learn more about derby.js, we aren’t going to write styles - this has been done - styles are already implemented in TodoMVC. Let’s have a look at the [.css file](http://todomvc.com/architecture-examples/angularjs/bower_components/todomvc-common/base.css). It’s clear that we are going to need a picture for background [bg.png](http://todomvc.com/architecture-examples/angularjs/bower_components/todomvc-common/bg.png). We are also going to use Angular generated html as a framework. I copied it with the help of browser tools and cleaned it a bit from Angular folders.

<div class='code-caption'>Defalut html code</div>

```js
<section id="todoapp">
  <header id="header">
    <h1>todos</h1>
    <form id="todo-form">
      <input id="new-todo" placeholder="What needs to be done?" autofocus">
    </form>
  </header>
  <section id="main">
    <input id="toggle-all" type="checkbox">
    <label for="toggle-all">Mark all as complete</label>
    <ul id="todo-list">
      <li>
        <div class="view">
          <input class="toggle" type="checkbox">
          <label>hello</label>
          <button class="destroy"> </button>
        </div>
        <form >
          <input class="edit">
        </form>
      </li>
    </ul>
  </section>
  <footer id="footer">
    <span id="todo-count"><strong>0</strong>
      <span>items left</span>
    </span>
    <ul id="filters">
      <li><a href="/" class="selected">All</a></li>
      <li><a href="/active">Active</a></li>
      <li><a href="/completed">Completed</a></li>
    </ul>
    <button id="clear-completed">Clear completed (0)</button>
  </footer>
</section>
```

As we can see, our html code consists of 3 basic blocks: 
1. header &mdash; this has the main input (we need it for adding new tasks)
2. main &mdash; main block which contains the task list itself
3. footer &mdash; status bar switching between filter and “Clear completed” button

## 2. Project structure

So, what’s going to be in our project? We are going to have styles file, html templates and at least two more files - server side and derby application itself. We are also going to need a server to serve static data (background picture). Here’s the file structure which suits our purposes):

```bash
public/
  bg.png
app             # derby application
  views/
    index.html
  css/
    index.css
  index.js      # derby application code
server.js       # server side derby
package.json
```

Note that the .css file is inside the `app` folder (not inside `public`), because derby works a bit differently with styles. As a result they are going to be inserted directly into `<head>` (they are going to be inside the style tag). According to Derby creators this is the best way to arrange style files.
 
So, the content of the app folder is an isomorphic derby application. I don’t really like the word “isomorphic” so I’m going to avoid using it. I’m just going to say **derby application** as contrasted to **server side derby**. The point is that these files (all the files in app) are going to be served as a single piece, that’s why I put them together.

For future reference we can divide the project into a few derby applications, for example, client side and admin panel. It makes sense because a) this way unwanted data isn’t served (templates, styles, code) b) and cohesion is reduced. So, the project has: server side and a few derby applications (in this case we have two).

There are going to be 2 modules - derby@0.6.0-alpha5 and derby-starter (as dependencies) in package.json file. 

## 3. Creating file structure

Let’s create the file structure. Download the picture and styles clicking the links mentioned above. Create package.json running npm init command.
The edit the html. Just like in the previous example 

1. it has to be in the predefined template Body;  
2. header, main and footer have to be moved out into separate derby templates.

<div class='code-caption'>index.html (code that was added is highlighted)</div>

```html
|+<Body:>
|+  <section id="todoapp">
|+    <view name="header"/>
|+    <view name="main"/>
|+    <view name="footer"/>
|+  </section>
  
|+<header:>
  <header id="header">
    <h1>todos</h1>
    <form id="todo-form">
      <input id="new-todo" 
             placeholder="What needs to be done?" autofocus">
    </form>
  </header>

|+<main:>
  <section id="main">
    <input id="toggle-all" type="checkbox">
    <label for="toggle-all">Mark all as complete</label>
    <ul id="todo-list">
      <li>
        <div class="view">
          <input class="toggle" type="checkbox">
          <label>hello</label>
          <button class="destroy"> </button>
        </div>
        <form >
          <input class="edit">
        </form>
      </li>
    </ul>
  </section>

|+<footer:>
  <footer id="footer">
      <span id="todo-count"><strong>0</strong>
        <span>items left</span>
      </span>
    <ul id="filters">
      <li><a href="/" class="selected">All</a></li>
      <li><a href="/active">Active</a></li>
      <li><a href="/completed">Completed</a></li>
    </ul>
    <button id="clear-completed">Clear completed (0)</button>
  </footer>
```

## 4. Calling templates

We can call our own templates with the help of the view tag (we are setting the name of the template in the name attribute).
For starters, let’s create some minimal working code in order to see the result in the browser and be able to add functionality. 
Server.js (file from the previous example) is extended so that it can take into consideration project file structure and serve static files. 

<div class='code-caption'>server.js</div>

```js
var server = require('derby-starter');

var appPath = __dirname + '/app';

var options = {
  static: __dirname + '/public'
};

server.run(appPath, options);
```

Let me remind you that we are using `derby-starter` module since this project serves educational purposes. If we look inside it we can see that serving static files is a classic usage of express static-middleware. 
Look at [this](https://github.com/codeparty/derby-starter/blob/master/lib/server.js#L72-L81).

<div class='code-caption'>index.js</div>

```js
var derby = require('derby');
var app = module.exports = derby.createApp('todos', __filename);

// we made app global for now to have access to it in the console
global.app = app;

app.loadViews (__dirname+'/views');
app.loadStyles(__dirname+'/css');

app.get('/', getTodos);

function getTodos(page, model){
  page.render();
}
```

Ok, lets run `npm start` (or directly `node server.js`). We can see the result in the browser: 

![TODO list example on Derby.js](/img/tutorials/derby2/2.png)

html and css are working. That’s the way to go!

## 5. Designing url

In the previous lesson I told you that derby developer should start development by dividing project into url addresses. This is due to derby’s ability to generate pages both on servers side and client side, which is great for search engines. So, examining the example written in Angular we’ve noticed that there are 3 links in the footer that change the url and the respective task filter. We can see that we need 3 handlers of get-requests in the application, for example:

```js
app.get('/', getAllTodos);
app.get('/active', getActiveTodos);
app.get('/completed', getCompletedTodos);
```

This would be necessary if all these pages were different. But the only difference between these pages is the filter, that’s why we’ll try not to duplicate code.
 
## 6. Designing data

### 6.1. Collections

The tasks will be stored in todos collection. Every task will have 2 fields: 

1. `text` &mdash; task description
2. `completed` &mdash; boolean, it means that the task is completed

Besides, each task has id field - derby will add it automatically when adding an element to the collection.
According to Derby methodology we need to prepare data before calling render in the controller (a function which processes request to url) and register subscriptions for data updates. So, schematically the handler should look like this: 

```js
function getTodos(page, model){
  model.subscribe('todos', function(){
    page.render();
  });
}
```

However, before moving further (to make one controller for all 3 requests and leave task filters different) we need to learn a few things about derby models:
 
1. **paths** start with `_` (e.g., `_session`, `_page`, etc.)
2. what is peculiar about `_page`?
3. what is **filters** in the context of Derby
4. what is **ref** to specific data in the collection

In the previous lesson I told you about so-called **paths**. We use them in operation with models. For instance, when you subscribe for data &mdash; `model.subscribe("path")`, `get` or `set` data in model. 
Here are the examples of paths:

1. `todos` &mdash; referencing the whole todos collection
2. `users.42` &mdash; referencing a document in the users collection with id = 42

So, the first part of the path is a name of the collection. It can start either with a letter or with one of these characters: `$` or `_`. Collections starting with `$` or `_` are special because they are not synchronized with the server (they are local within the model and we can create only one model in derby application). Collections starting with `$` are reserved by Derby for its own purposes. And collections starting with `_` are used by the developers. 

Let’s perform a small experiment. Open console in the browser, type in `app.model.get()` and have a look at the output.
 
Among `_`-collections there is a special one &mdash; `_page`. It resets every time url is changed which makes it very convenient for storing working data. You’re going to see more examples in this lesson. 

### 6.2. Filters

Now, let’s move on to filters. If you’ve read the documentation (the part concerning models) you probably know that in Derby there are various mechanisms which make working with reactive data a lot easier. I’m talking about reactive functions, subscriptions for different events, filter functions and sort functions.
  
Let’s talk about filters. That’s what we should do to implement a filter which indicates active tasks: we are registering a filter function with a name (it’s required for serialization in bundle). According to documentation we must register it in `app.on("model")`. 

```js
app.on('model', function(model) {
  model.fn('completed', function(item) { 
    return  item.completed;
  });
});
```

And then we’re using this filter in controller for filtering `todos` collection:

```js
function getPage(page, model){
  model.subscribe('todos', function() {
    var filter = model.filter('todos', 'completed')
    filter.ref('_page.todos');
    page.render();
  });
}
```

Take a look at `filter.ref('_page.todos');`. Here filtered `todos` becomes accessible through `_page.todos`. Here’s the code with filters and controllers:

```js
app.on('model', function(model) {
  model.fn('all',       function(item) { return true; });
  model.fn('completed', function(item) { return  item.completed;});
  model.fn('active',    function(item) { return !item.completed;});
});

app.get('/',          getPage('all'));
app.get('/active',    getPage('active'));
app.get('/completed', getPage('completed'));

function getPage(filter){
  return function(page, model){
    model.subscribe('todos', function() {
      model.filter('todos', filter).ref('_page.todos');
      page.render();
    });
  }
}
```

You’ve probably noticed that in order to unify everything we had to create a false filter `all`, but I think it not that bad since it allowed us not to repeat the code. 

## 7. Add and display tasks

Here’s how data input html looks like:

```html
<form id="todo-form">
  <input id="new-todo" placeholder="What needs to be done?" autofocus>
</form>
```

Reactive bindings is a classic pattern in Derby (like in many other frameworks). Let’s bind input value to some path in `_page` and register `submit` event handler to handle clicking enter.

```html
<form id="todo-form" on-submit="addTodo(_page.newTodo)">
  <input id="new-todo" placeholder="What needs to be done?" 
         autofocus value="{{_page.newTodo}}">
</form>
```

We could use `on-click`, `on-keyup`, `on-focus` instead of `on-submit`. We insert the handler into `app.proto` (when we discuss derby components we’ll see that each component has its handlers in it):

```js
app.proto.addTodo = function(newTodo){

  if (!newTodo) return;

  this.model.add('todos', {
    text: newTodo,
    completed: false
  });

  this.model.set('_page.newTodo', '');
};
```

Let’s check whether it’s an empty string or not, add the task in the collection and clear input. You might have noticed that there was only one argument in the handler. If we needed links to an event object or html element itself for some reasons we would need to write this stuff in html: 

```js
on-submit="addTodo(_page.newTodo, $event, $element)"
```

`$event` and `$element` are arguments which are automatically filled by derby.
 
Now, let’s get to the filtered task list and edit ul element. 

```js
<ul id="todo-list">
  {{each _page.todos as #todo, #index}}
  <li class="{{if #todo.completed}}completed{{/}}">

    <div class="view">
      <input class="toggle" type="checkbox" 
             checked="{{#todo.completed}}">
      <label>{{#todo.text}}</label>
      <button class="destroy"> </button>
    </div>
    <form>
      <input class="edit">
    </form>

  </li>
  {{/each}}
</ul>
```

To Summarize, here’s what we did: 

1. ran through each task (already filtered) and create `li` elements for them
2. output task description in `label`
3. bound checkbox to `todo.completed`
4. set class `completed` to `li` tag if the task is completed

## 8. Delete elements

It’s pretty simple:

```html
<button class="destroy" on-click="delTodo(#todo.id)"></button>
```

```js
app.proto.delTodo = function(todoId){
  this.model.del('todos.' + todoId);
};
```

There can be even less code:

```html
<button class="destroy" on-click="model.del('todos.' + #todo.id)">
</button>
```

Deleting all completed tasks is similar (“Clear completed” button is on the right on the bottom):

```html
<button id="clear-completed" on-click="clearCompleted()">
  Clear completed (0)
</button>
```

```js
app.proto.clearCompleted = function(){
  var todos = this.model.get('todos');

  for (var id in todos) {
    if (todos[id].completed) this.model.del('todos.'+id);
  }
}
```

## 9. Edit elements

When we double click on task it switches to edit mode. According to html we’ll need to add editing class to according li element when switching to this edit mode. 
We’ll also need to get rid of selection which occurs when we double click and to set focus on the input we need.
 
Let’s store the information about the task we’re editing by this address: `_page.edit`. We’re going to store `id` (of the task we’re editing) and the `text`. 
So, here’s our code: 

```html
<ul id="todo-list">
  {{each _page.todos as #todo}}
    <li class="{{if #todo.completed}}completed{{/}} 
               {{if _page.edit.id === #todo.id}}editing{{/}}">

      <div class="view">
        <input class="toggle" type="checkbox" 
               checked="{{#todo.completed}}">
        <label on-dblclick="editTodo(#todo)">{{#todo.text}}</label>
        <button class="destroy" on-click="delTodo(#todo.id)"></button>
      </div>
      
      <form on-submit="doneEditing(_page.edit)">
        <input id="{{#todo.id}}" value="{{_page.edit.text}}" 
               class="edit" on-keyup="cancelEditing($event)">
      </form>

    </li>
  {{/each}}
</ul>
```

```js
app.proto.editTodo = function(todo){

  this.model.set('_page.edit', {
    id: todo.id,
    text: todo.text
  });

  window.getSelection().removeAllRanges();
  document.getElementById(todo.id).focus()
}

app.proto.doneEditing = function(todo){
  this.model.set('todos.'+todo.id+'.text', todo.text);
  this.model.set('_page.edit', {
    id: undefined,
    text: ''
  });
}

app.proto.cancelEditing = function(e){
  // 27 = ESQ-key
  if (e.keyCode == 27) {
    this.model.set('_page.edit.id', undefined);
  }
}
```

Double clicking triggers editTodo function, and we are setting `_path.edit` in it, deselecting and switching focus to the input we need. 

When we’re done editing we click either enter or esq. One of these two handlers &mdash; doneEditing and cancelEditing triggers accordingly. 

## 10. Reactive functions. Counting active and completed tasks

So, **reactive function** is a function that triggers every time data is changed. What we need to do is to specify that this particular reactive function is going to trace changes of particular data. Data serves as an argument in the function. The function calculates something and returns the result. The result is bound to a particular **path**.

Ok, that’s quite abstract and may be a bit difficult to understand. Let’s look at the example. We have todos collection with active and completed tasks. It would be great for us to have access to the counters of active and completed tasks. So, we need something like this:

```js
_page.counters = {
  active: 2,
  completed: 3
}
```

Then we would be able to feed data into footer. 
One of the options to get data is to use reactive functions. They are registered just like filters:

```js
app.on('model', function(model) {
  model.fn('all',       function(item) { return true; });
  model.fn('completed', function(item) { return  item.completed;});
  model.fn('active',    function(item) { return !item.completed;});

  model.fn('counters', function(todos){
    var counters = { active: 0, completed: 0 };
    for (var id in todos) {
      if(todos[id].completed) counters.completed++; 
      else counters.active++;
    }
    return counters;
  })
});
```

That’s how we’ve registered counters function, but that’s not all we need to do. 
We also need to run it at a certain time and bind it to paths. This is done in controller with the help of `model.start`: 

```js
model.subscribe('todos', function () {
  model.filter('todos', filter).ref('_page.todos');
  model.start('_page.counters', 'todos', 'counters');
  page.render();
});
```

Now we have access to all the counters in the templates. Let’s work on footer: 

```html
<footer:>
  <footer id="footer">
    
    <span id="todo-count"><strong>{{_page.counters.active}}</strong>
      <span>items left</span>
    </span>
    
    <ul id="filters">
      <li><a href="/" 
             class="{{if $render.url==='/'}}selected{{/}}">
        All
      </a></li>
      
      <li><a href="/active" 
             class="{{if $render.url==='/active'}}selected{{/}}">
        Active
      </a></li>
      
      <li><a href="/completed"
             class="{{if $render.url==='/completed'}}selected{{/}}">
        Completed
      </a></li>
    </ul>
    
    <button id="clear-completed" on-click="clearCompleted()" 
            class="{{if _page.counters.completed==0}}hidden{{/}}">
      Clear completed ({{_page.counters.completed}})
    </button>
    
  </footer>
```

We’ve displaying the counters, and hiding “Clear completed” button if there are no more completed tasks. We’ve added selected class to the link which is active, using information we got when examining `app.model.get()` in console in the browser. The reserved collection `$render` contains useful information, for instance, `url` (which was rendered). Look into console one more time.

Let’s play with what we have now. Open a few tabs and check if data is synchronized.
This project is on github so you can [check it out](https://github.com/zag2art/derby-example-todo) to compare. 
