# Tutorial. Learning Derby 0.6. Introduction to Derby.js

For the last few months I’ve been taking part in a few projects developed in Derby (http://derbyjs.com/), a reactive fullstack javascript framework. Some of them are working quite successfully in production, others are going to be launched in the short run. 

While learning this technology I was struck by a few thoughts. Firstly, very little information can be found about it. The documentation is pretty poor, and the articles which you can find are mostly written by people who spent a day or two learning Derby. Secondly, there is a small group of people who know this technology pretty well and are using it in their projects, successfully solving  all the problems beginners run into. 

The idea is to share the acquired knowledge with those who are interested in it. I’d like to show you a few examples from the derby-examples project and examine them in detail. Alternatively, I can either reconstruct them from the ground up, explaining logic from an expert viewpoint, or using examples I can explain the issues which were left uncovered in previous examples.

## 1. Derby v0.6

We are going to start learning Derby v0.6. Currently version 0.6 alpha 5 has been released. It’s stable enough for us to start migrating all our projects from v0.5 to v0.6. New projects are created only on v0.6. For example, [Type4Fame](http://type4fame.com) is my friend’s hobby project created on 0.6.

As compared to 0.5, we can see that things have changed for the better. The code of the framework is well-structured and understandable and rendering is accelerated. Besides, it has finally become possible to use arbitrary expressions in views. The new system of components allows to successfully structure large applications and divide them into parts with low cohesion. Well, I’ve got to say that I’m very much impressed. 

It is fair to say that it isn’t alpha release and the creators have a certain set of unimplemented TODOs and there are some bugs too (however, they are being fixed).
 
## 2. Level of competence 

Here’s the list of things you need to know and understand to catch on Derby:

1. basic knowledge of web development (html, css, javascript);
2. nodejs &mdash; you need to know how commonjs-modules and npm work, understand how to run standard http server; 
3. expressjs &mdash; derby applications are built on top of express applications, so it would be   useful to have basic knowledge about Express and server-side javascript (plugging in modules, processing requests, etc.);

There is a specific set of technologies used by Derby and it can be very helpful to get to know them (although it is optional, we are going to find out about them):

1. reactive programming
2. isomorphic javascript applications
3. operational transformations - technology developed by google for resolving conflicts  when editing google docs simultaneously  
4. browserify
5. mongodb

## 3. Platform

We are going to need Linux or Mac. Windows may cause problems - v0.5 worked fine even on Windows (but I had to compile redis v2.6 myself) and  v0.6 is on the way toward it, however there are a few pull-requests, which aren’t in the main repository yet.

Before getting started, let’s make sure that you have these installed on your computer:
 
1.   nodejs
2.   mongodb
3.   redis 2.6 (note that v2.4 doesn’t suit us) - upd redis is not required anymore

They are installed with settings automatically. Mongo and Redis have to be launched before running the application.

## 4. Getting started

Let’s look at the example  c. It is very simple, still there are going to be lots of explanations. You can copy the example and examine it or write it from scratch. I’d recommend the second option since the odds are that you are going to remember it a lot better doing it this way. Anyways, I’m going to explain you both variants. 

Copy this chunk of code from my repository: 

```bash
# copy
git clone https://github.com/zag2art/derby-example-hello.git

# enter project directory
cd derby-example-hello

# install dependencies(there are two packages: derby и derby-starter)
npm install

# to run the app you need to execute the following command and open http://localhost:3000/ in browser
npm start
```

We are going to need to  create all the files manually (also we have created a directory in advance and entered it). We are creating `package.json` using `npm init` (specifying that server.js is a startup file) and adding 2 modules also using npm:

```bash
# installing the latest version of derby from v0.6 branch (without specifying the version npm is going to get us v0.5) and save it to package.json
npm install derby@~0.6 -S

# installing derby-starter
npm install derby-starter -S
```

It’s necessary to say few words about derby-starter. You are probably never going to use it in your projects. Have a look at its code (https://github.com/codeparty/derby-starter/blob/master/lib/server.js). Most importantly, standard expressjs application (derby attaches to it as a middleware) is initialized here. Also, server side of derby is set up here and connecting derby to data (mongo and redis) is configured here too.

Normally, this code can be found inside the derby application, because it needs to be edited. For example, you are going to need to plug in authorization via passport, restrict access to data via racer-access, or if you feel like processing part of requests by express but not derby(e.g. if you need a replicated restful-api). Anyway, we don’t need it for now and thanks to this module our server side is reduced to one line:

<div class='code-caption'>server.js</div>

```js
require('derby-starter').run(__dirname+'/index.js');
```

We are launching server side derby passing the path to index.js (which contains our Derbyjs application) as an argument. 
So, the application consists of 2 files: 

<div class='code-caption'>index.js</div>

```js
var app = module.exports = require('derby').createApp('hello', __filename);
app.loadViews(__dirname);

// the route is being rendered both on server and client side
app.get('/', function(page, model) {
  // subscription enables data synchronization
  model.subscribe('hello.message', function() {
    page.render();
  });
});
```

<div class='code-caption'>index.html</div>

```html
<Body:>
  <!-- in the template you write html and define bindings -->
  Holler: <input value="{{hello.message}}">
  <h2>{{hello.message}}</h2>
```

Firstly, let’s create these files, launch the application, play with it a bit and figure out how it works. So, one directory contains all the files: `package.json`, `server.js`, `index.js` and `index.html`. Modules `derby` and `derby-starter` are installed and can be found in subfolder node_module. Mongo and redis are installed and running. 
Let’s launch the application (`npm start` or directly &mdash; `node server.js`):

```bash
zag2art@laptop:~/work/derby-example-hello$ npm start

> derby-example-hello@0.0.0 start /home/zag2art/work/derby-example-hello
> node server.js

Master pid  11291
11293 listening. Go to: http://localhost:3000/
```

Open your browser and go to http://localhost:3000/. Now write something in the input, e.g.:

![Hello World example](/img/tutorials/derby1/1.png)

We can see that the text we typed in appeared in h2 block below the input. Reactive bindings are handling it. 

Here’s one more experiment. Let’s open one more tab with the same address (http://localhost:3000/). We can see the text we had typed before. So, the data is synchronized between the tabs. If our application were hosted somewhere in the Internet, it would be synchronized among all the clients. Now let’s change the data.  We will be able to see the changes on the second tab instantly without any blinks. When data is updated  the part of the page (where the data is changed) is recreated on the client side. Now let’s have a look at the page sourcecode on the second tab:

![Hello World source code](/img/tutorials/derby1/2.png)

## 5. Derby features

Now let’s find out about the Derby features and the sourcecode. The first thing we should talk about is the system requirements.
 
1. Derby is designed to create SPA (complex web-applications with lots of elements) applications with constant re-rendering on the page.
2. For better usability the application should allow to create bookmarks for its current state, which means that when a significant change occurs the url changes.
3. The application has to be indexed, therefore when designing application architecture the developer has to decide which application states should be indexed and think about the corresponding urls. 

So, Derby creators have understood that if the page is requested from the server, it has to be generated there and served fully rendered. And if the user is working with the application in the browser he can’t ask server for html every time the url is changed and wait till the screen blinks. Also, we don’t want to duplicate the page rendering code. It means that we need to be able to render the page on the server and client side. Front-end and back-end are both written in javascript which makes things easier. But is it possible to do so? 

We need the code responsible for pages rendering (templating, html-related stuff), routing (working with url when changing address on the client side) to work the same way on front-end and back-end. We also need to arrange data access the same way on both sides. The code working the same way on server side and client side is called isomorphic. 
You can find more about it [here](http://venturebeat.com/2013/11/08/the-future-of-web-apps-is-ready-isomorphic-javascript/).
It is implemented in Derby. 

So, there are two parts in Derby: 
1. Part related to server (all sorts of settings, plugging modules, integration with expressjs). Note that there is no rendering and no routing. In this case it’s server.js file with derby-starter module. 
2. Isomorphic application itself. Its code is executed on server side when some page is requested from server and then the code is executed on client side. In this case it’s index.js and index.html files.   

Let me tell you about the second part in more detail. Imagine that we’ve created a derby application working with 3 pages:

1. `/` - website homepage 
2. `/about` - page containing information about the developer 
3. `/projects` - page with projects

All the pages have links on each other. How is search engine going to work with this website? It is going to make 3 requests (one to each page) and index the pages properly. How is the user going to interact with it? 

The user enters the site (by clicking a link or typing in the address in the address bar) and opens one of these three pages. This will be the only request to server, because the user receives a fully rendered page and the whole isomorphic application (so called bundle which includes everything we wrote; that is, packed js files and the part of derby which allows it all to work). 

So, for example, when requesting the page `/about` the user gets a fully rendered SPA application. And now if he clicks `/` or `/projects` link on `/about` page there will be no more requests to the server (there should be an asterisk and I’m going to explain this part later). The page is rendered directly on the client side and the url is going to remain unchanged in the browser. 

And so on, and so forth. The user can jump from one page to another without any requests to server. 
Now, let’s have a look at `index.js` code:

<div class='code-caption'>index.js</div>

```js
var app = module.exports = require('derby').createApp('hello', __filename);
app.loadViews(__dirname);

// the route is being rendered both on server and client side
app.get('/', function(page, model) {
  // subscription enables data synchronization
  model.subscribe('hello.message', function() {
    page.render();
  });
});
```

We should point out that `index.js` is a common.js module. It means that we can easily plug in any module we like in it and they can work both on client and server side. Here’s the example: `var _ = require('underscore');`
Derby is included and initialized in the first line.

```js
app.loadViews(__dirname);
```

Here were are showing where the files with templates are. In this case we’ve pointed to the directory which has index.js in it. Derby will look up there and find index.html. Our application is pretty small since it serves educational purposes, that is why all our files are found in a single directory. However, we normally find templates in subdirectory views.  So, if we had css styles we would find here something like this:
 
```js
// here’s an example from a bigger application
app.loadViews (path.join(__dirname, '../views'));
app.loadStyles(path.join(__dirname, '../styles')); 
```

For example, index.css could be found there (or `index.styl`, or `index.less` - Derby supports them). Of course there can be includes with their structure of subdirectories.
 
The code of processing url comes next. Schematically, if we created the application which I had described (the one with 3 pages) there would be:

```js
app.get('/', function(page, model) {
  ...
});

app.get('/about', function(page, model) {
  ...
});

app.get('/projects', function(page, model) {
  ...
});
```

Hope, it’s clear. For everything that we call a separate page there has to be a handler (or someone may say a controller). By the way, this stuff is similar to expressjs and we can have arguments here, for example:

```js
app.get('/users/:user', getUser);
```

For more information go [here](http://derbyjs.com/#routes).

## 6. Handlers

What are we doing inside the handler? Similarly with `Express.js` we need to prepare data for page rendering and call render function with a corresponding template and data as an argument. It is the same scheme but it has certain peculiarities. 

First of all, let’s talk about data. Model (passed as an argument to the handler) is used to work with data. Derby is a reactive framework, which is why we have 2 options when working with data. The first option is to receive data as it currently is and give it to template engine. The 2nd option is to receive data and subscribe for updates. So, what does this actually mean?
 
Imagine a page with some list. If we simply receive and output data the page will be static. But if we subscribe for updates the list on the page is going to be updated (if, for example, another user who has access to this list changes it). That’s what the code is going to look like:
 
```js
// getting data without subscribing to updates
model.fetch('list', function(){
  // Here we got the data 
  // We can call render
});

// getting data subscribing to updates
model.subscribe('list', function(){
  // Here we got the data 
  // Subscription for future updates is registered 
  // We can call render 
});
``` 

Here’s the code from our example: 

```js
model.subscribe('hello.message', function() {
  page.render();
});
```

We are subscribing for `hello.message` and calling render in a callback function. Ok, so what is `hello.message`? Why does render have no arguments (no template, no data)?

In `Derby` module `Racer` is responsible for reactive data. All access to data in `Racer` happens with the help of so called “paths”. So, `hello.message` is a path, where `hello` is a name of a collection in Mongo (in sql there would be a table) and `message` is an id of the document in the collection. We have subscribed for a single document in `hello` collection. This data is going to become a value of our input in the template. 
You can find more information about working with data and paths [here](http://derbyjs.com/#models).

We haven’t passed anything to render. The reason for this is that html for url `/` is specified in the main html file (`index.html` is a root file, there could have been a few more files connected to it via import) and it’s going to be used by default. And we haven’t passed data to render because we can access paths in the templates. 
Here’s the file with the templates: 

<div class='code-caption'>index.html</div>

```html
<Body:>
  <!-- in templates you write html and set up data bindings -->
  Holler: <input value="{{hello.message}}">
  <h2>{{hello.message}}</h2>
```

## 7. Templates

So, the file itself consists of a few sections (here we have only one section &mdash; `Body`). In Derby terminology they are called “templates”. We have a templates file which has `Body` template in it. And everything you put in `Body` template is going to appear in the resulting html body. I’m not going to explain the whole system of templating (including namespaces, inheritance, etc.) in Derby using this single example, but I still need to mention a few more things. 

Between the double curly braces we can see bindings to data (we called them paths earlier). To set page title, for example, we could use another predefined template `Title`:

```html
<Title:>
  My first Derby application
  
<Body:>
  <!-- we’ve got html in the template and bindings to data -->
  Holler: <input value="{{hello.message}}">
  <h2>{{hello.message}}</h2>
```

Template engine looks pretty much like handlebars. I’m going to give you a few more examples for clarity:

```html
<Body:>
  {{if _page.login}}
    <h3> Hello, friend! </h3>
  {{/}}  

  {{each _page.todos as #todo, #index}}
    <p>{{#todo.text}}</p>
    {{if #index % 5 === 0}}
        <!-- 0, 5, 10, 15 -->
        <b>important</b>
    {{/}}
  {{/}}
```

And here’s the example of binding to events:

```html
<a href="#" on-click="{{removeTopic(topic[#index])}}">Ok</a>
```

We’re a bit ahead of time, but I think it’s reasonably necessary. In our next tutorial we are going to create a TODO-list and I’ll give you the explanations so that you can figure out how it works.
 
For now, let’s play with the code. Add something to the template, change title (note that Derby supports live reloading, so as soon as you have changed data in html file the browser will reload the page automatically). Try to plug in styles. 

**P.S.** 
If you feel like reading standard Derby documentation, note that it’s written for v0.5 and some parts are outdated. However, it would be useful to read about controllers and models. However,  I recommend not reading: views, templates, components. These parts have changed so reading them will probably confuse you. 