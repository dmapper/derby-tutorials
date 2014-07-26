# Learning Derby 0.6, example #3

This tutorial consists of 2 parts:
 
1. server side derby review
2. derby-auth usage (for registration/authorization)

derby-auth is a module which we will use as a wrapper for passportjs. 

![Derby auth github authentication](/img/tutorials/derby3/1.png)

## I. Server side  

In previous examples we  used derby-starter as server side. As we know, debry application is built on top of standard Express.js application and derby-starter module conceals the details of backend configuration e.g. database settings and express.js middlewares usage. So, this time we won’t use derby-starter for better understanding of backend. 

### Basic application

To get started, copy derby-boilerplate repository. This is a minimal working application similar to one in the previous example, however server side is not in derby-starter now, but in the project itself (by the way, expressjs 4.0 is used in this application and we don’t need to use redis anymore). So,  run this command: 

```bash
git clone https://github.com/derbyparty/derby-boilerplate.git
```

Let’s have a look at project structure:

```bash
src/
  app/
  server/
styles/
views/
index.js
```

Our derby application is going to be in the app folder. derby-starter analogue is going to be in the server folder. Actually, I copied and pasted the code from derby-starter and edited it.
 
Styles are in the styles folder and templates are in the views folder. index.js file is similar to what we had in previous examples &mdash; there are a few lines of code which launch the whole thing. 

### Derby application server side

Note that derby is built on top of expressjs application so if you don’t know anything about sessions in express, routing, express-middleware or know very little about this stuff you should study these subjects first and have a good understanding of them. 

Server folder contains: 

```bash
error/
index.js
server.js
```

There is error handler in the error folder. It’s expressjs-middleware which triggers when there is no handler for the url or some errors occur in the process.
 
The main purpose of the `index.js` file is to pick up express (which is set in `server.js`) and run server on a specified port.

<div class='code-caption'>Initial code</div>

```js
var derby = require('derby');

exports.run = function (app, options, cb) {

  options = options || {};
  var port = options.port || process.env.PORT || 3000;

  derby.run(createServer);

  function createServer() {
    if (typeof app === 'string') app = require(app);

    var expressApp = require('./server.js').setup(app, options);

    var server = require('http').createServer(expressApp);
    server.listen(port, function (err) {
      console.log('%d listening. Go to: http://localhost:%d/', process.pid, port);
      cb && cb(err);
    });
  }
}
```

We are approaching the most important and interesting part - setting expressjs with derby in it. I’m using expressjs v4.0. The main difference between v4.0 and  v3.0 is that there is no standard middleware installed in express package by default and we need to install it manually. 

That’s how server.js would look like without derby in it (a regular expressjs application): 

```js
var express             = require('express');

// in express v4.0 middleware is in separate modules
// so, we need to require each module separately
var session             = require('express-session');

// sessions are stored in mongo
var MongoStore          = require('connect-mongo')(session);

// error handler is in the separate folder
var midError            = require('./error');

var MongoClient = require('mongodb').MongoClient;

exports.setup = function setup(app, options) {

  var mongoUrl = process.env.MONGO_URL || process.env.MONGOHQ_URL || 'mongodb://localhost:27017/derby-app';

  // initializing connection to database
  MongoClient.connect(mongoUrl);

  var expressApp = express()

  if (options && options.static) {
    expressApp.use(require('serve-static')(options.static));
  }

  expressApp.use(require('cookie-parser')());
  expressApp.use(session({
    secret: process.env.SESSION_SECRET || 'YOUR SECRET HERE',
    store: new MongoStore({url: mongoUrl})
  }));

  // if there were some express routes they would be here

  // the default route - generating 404
  expressApp.all('*', function(req, res, next) { next('404: ' + req.url); });

  // error handler
  expressApp.use(midError());

  return expressApp;
}
```

Let me remind you how this code works:

1. plugging in modules
2. plugging in expressjs middleware (via expressApp.use).
 
Basically, middleware is a set of functions which is called for each request (that is sent to the server) in the order in which they are registered. Each of this middleware functions can either answer the request and end the chain of processing (the remaining middleware functions will not run) or  perform some intermediate actions with the request (e.g., parse cookies) and pass control to other middleware functions.  The order of plugging middleware functions is very important &mdash; that’s the order of calling functions for each request. 

Here’s the `server.js` file with derby (derby application):

<div class='code-caption'>server.js</div>

```js
// express v4.0
var express             = require('express');

// in express v4.0 middleware is in separate modules
// so, we need to require each module separately
var session             = require('express-session');

// sessions are stored in mongo
var MongoStore          = require('connect-mongo')(session);

// error handler is in the separate folder
var midError            = require('./error');

var derby               = require('derby');

// BrowserChannel (socket.io equivalent) is a transport protocol 
// used in derby to pass data from the browser to the server

// liveDbMongo (mongo driver for derby) can update data reactively 
var racerBrowserChannel = require('racer-browserchannel');
var liveDbMongo         = require('livedb-mongo');

// plugging a module which creates browserify bundles
derby.use(require('racer-bundle'));

exports.setup = function setup(app, options) {

  var mongoUrl = process.env.MONGO_URL || process.env.MONGOHQ_URL ||
      'mongodb://localhost:27017/derby-app';

  // initializing connection to database
  var store = derby.createStore({
    db: liveDbMongo(mongoUrl + '?auto_reconnect', {safe: true})
  });

  var expressApp = express()

  // here the application returns its “bundle” 
  // (requests to "/derby/…" are processed here)
  expressApp.use(app.scripts(store));

  if (options && options.static) {
    expressApp.use(require('serve-static')(options.static));
  }

  // client script browserchannel is added to bundle here
  // and middleware which process client requests are returned
  // (browserchannel is based on long-polling which means that 
  // requests to /channel are processed here
  expressApp.use(racerBrowserChannel(store));

  // add getModel() method which allows express controllers to read 
  // and write from/to database
  expressApp.use(store.modelMiddleware());

  expressApp.use(require('cookie-parser')());
  expressApp.use(session({
    secret: process.env.SESSION_SECRET || 'YOUR SECRET HERE',
    store: new MongoStore({url: mongoUrl})
  }));

  expressApp.use(createUserId);

  // dery application controllers are registered here
  // they get triggered whenever the user gets pages from the server
  expressApp.use(app.router());

  // if there were some express routes they would be here

  // the default route - generating 404
  expressApp.all('*', function(req, res, next) { 
    next('404: ' + req.url); 
  });

  // error handler
  expressApp.use(midError());

  return expressApp;
}

// throwing user id from session to derby model
// if there is no id in the session we generate random id
function createUserId(req, res, next) {
  var model = req.getModel();
  var userId = req.session.userId;
  if (!userId) userId = req.session.userId = model.id();
  model.set('_session.userId', userId);
  next();
}
```

Spend some time figuring out how this code works. The main thing here is that we use browserify to return a so-called “bundle” &mdash; a block of data which contains all javascript files of the derby application and templates (css scripts won’t be here) to client in the browser. 

It is important to understand that bundle is not returned to the browser when there is a request of some page by client. We receive a fully rendered page with styles and some initial data. It makes sense because speed is really important. And then this page loads “bundle” &mdash; request is made by address `/derby/:bundle-name`.  

Since server side derby can correspond to several derby applications, there is going to be a bundle for each of these applications. It makes sense, because this way  we can separate client side from admin-section and omit passing redundant data. That’s what we need to write: 

```js
expressApp.use(clientApp.scripts(store));
expressApp.use(adminApp.scripts(store));
```

The same applies to routes handling. If there are  2 derby applications we will need to plug in 2 application routers:

```js
expressApp.use(clientApp.router());
expressApp.use(adminApp.router());
```

instead of plugging in one:

```js
expressApp.use(app.router());
```

In addition to route handlers of derby application we can add expressjs handlers implementing any RESTfull API. When derby application client router can’t find the appropriate route, it sends request to the server to handle it.
 
The next step is to connect browserchannel (an analogue of socket.io). It is a transport protocol which is responsible for synchronizing the application with the server in real time. At some point the client script browserchannel is added to bundle as well as handling requests from this module (it is based on long-polling, so, we need to process those requests &mdash; this module binds it to path `/channel`).
 
Note that we do not use redis in the basic setup of the example. Now we can avoid using redis (after the last derby update), but we need it when we do horizontal scaling &mdash; launch a few derby servers at the same time to maximize performance.
 
So, at this point you should understand that the structure allows to add stuff to the application very flexibly. We can add basically anything: expressjs-middleware, derby plugins, derby applications, express handler, etc. 

Let’s talk about sessions. As you can see from the example the sessions we use are express sessions. Cookies are used for authentication (which will be stored in the client browser), the sessions are stored in mongodb in sessions collection.
 
If we look at createUserId function 

```js
function createUserId(req, res, next) {
  var model = req.getModel();
  var userId = req.session.userId;
  if (!userId) userId = req.session.userId = model.id();
  model.set('_session.userId', userId);
  next();
}
```

we can see that in this function userId is taken from the session. If there is no userId it is generated randomly (`model.id()`) and set in the model on path `_session.userId` (this data is serialized and passed to the client side) and in the session. 

There can be several variants of requests:

1. The user visits the website for the first time and requests the main page &mdash; new cookie is generated on the server and the new session (with the new random userId) is generated &mdash; the main page is rendered in the browser and if we look at the model of the client side, we will see the newly generated id on `_session.userId` path. When visiting different pages of the application there will be no more requests to the server &mdash; the client side always has its Id and if it sends requests to the server, we will be able to see it. 
2. The user didn’t sign up and visited the website in a week. The cookie was saved and he will be assigned the same userId. 
3. The user visited the website from a different browser (the same as in the 1st paragraph). 
 
Let’s examine registration/authorization in the context of the server. Suppose, the client filled in registration data which is going to be sent to the server where we would create a document in the users collection. We don’t need to worry about the id &mdash; it’s already created.
 
As for the authorization, the client visits the websites, gets a random id, goes to authorization page, logs in, passes the data to the server, the server recognizes the client and generates a new userId in the session and in the  `_session.userId`.

This collection will probably store username, email, passwordHash, and some static data. If there were authorization (registration) on our website via social networks, social network keys would be stored in the collection. We can also store user settings here.
  
Of course, we would like a part of this collection to be hidden on the client side. For now, there is no such problem. Recently so-called projection (ability to subscribe for the collection, subscribe for the certain fields of the document) have appeared in derby. A few weeks ago this functionality didn’t exist and derby-auth module (we’ll talk about this module later) has no projections. Projections allow us to divide user data into 2 parts:
 
1. `auths` collection (private data only)
2. `users` collection (accessible public data)
 
## II. Derby-auth
 
### Authorization in derby application (derby-auth module)

There are two modules which handle authorization in derby:
 
1. `derby-auth`
2. `derby-passport`

Both of these modules are passportjs wrappers. Current versions of derby-auth and derby-passport are for derby v0.5. Although they aren’t updated for derby v0.6, we still use them in derby v.0.6 (html forms templates are not available, though, but it’s fine because we use custom templates).
 
I opted for derby-auth because I have some experience working with it. 
Let me remind you that user data  is divided into two parts. Public data is stored in users collections, private is stored in auths. 
We can create projections - subscribe for certain fields in the collection. 

### Update 

`derby-login` is a module which handles authorization. This module came out a few days after this article had been written. So, `derby-login` supports derby v0.6, allows to make projections and has ready-to-use components for registration, logging in, changing password. It’s great and I’d recommend to use this module. 

### passportjs

`PassportJs` is a middleware that handles node.js authorization. It is a module that abstracts authorization &mdash; it allows your application to use simple log in/out authorization via almost any service which supports `Oauth`, `Oauth 2`. 

Types of authorization are called strategies. For example, log in/password authorization is called LocalStrategy (local authorization strategy) and GitHub authorization is called GithubStrategy. Each strategy is in the separate module, so plugging authentication looks like this: 

```js
var passport       = require('passport');
var LocalStrategy  = require('passport-local').Strategy;
var GithubStrategy  = require('passport-github').Strategy;
```

### Strategies

There are certain features of using  strategies of authorization via social networks. Our application can use this type of authorization when it

1. is registered in social network
2. got 2 arguments:
 
    1. appId (or clientId -- a unique id of the application in this particular social network)
    2. Secret (a secret key which we need to authenticate ourselves)

We also need to specify redirectUrl (social networks redirect to this url after authorization) to sign up.  In the example redirectUrl is `localhost:3000/auth/github/callback`.
 
In the example I’m going to use github authorization. Firstly, we need to have a user account to create the application in github account settings.
 
![Derby auth github authentication](/img/tutorials/derby3/2.png) 

### Plugging `derby-auth` to our application

Install derby-auth directly from git-repository. Master branch is for derby v0.3: 

```bash
npm install git://github.com/cray0000/derby-auth#0.5 -S
```

derby-auth automatically installs `passport` and `passport-local`. Install `passport-github`:

```bash
npm install passport-github -S
```

### Configuration

(according to the [instruction](https://github.com/lefnire/derby-auth/tree/0.5))

#### Step 1

Plug derby-auth and configure strategies (normally, configurations are specified in config file or environment variables).

```js
// Plugging derby-auth 
var auth = require('derby-auth');

var strategies = {
  github: {
    strategy: require("passport-github").Strategy,
    conf: {
      clientID: 'eeb00e8fa12f5119e5e9',
      clientSecret: '61631bdef37fce808334c83f1336320846647115'
    }
  }
}
```  

#### Step 2

Set options.

```js
var options = {
  passport: {
    failureRedirect: '/login',
    successRedirect: '/'
  },
  site: {
    domain: 'http://localhost:3000',
    name: 'Derby-auth example',
    email: 'admin@mysite.com'
  },
  smtp: {
    service: 'Gmail',
    user: 'zag2art@gmail.com',
    pass: 'blahblahblah'
  }
}
```

`successRedirect` &mdash; user is redirected here after successful authorization;
`failureRedirect` &mdash; user is redirected here if authorization fails;
We need information about the website to pass it to strategies (e.g., it can be used by social networks if some fields weren’t filled during the registration). Email settings are required to reset the password. 

#### Step 3

Initialize the storage: 

```js
auth.store(store, false, strategies);
```

`derby-auth` adds access control to data. The user won’t have access to other users’ documents in the collection. The second argument is not used at the moment. 

#### Step 4

Let’s make sure that middleware which handles storing sessions and `body-parser` are plugged to express. Body-parser isn’t plugged yet &mdash; install 

```bash
npm i body-parser -S
```

and plug it:

```js
expressApp.use(require('cookie-parser')());
expressApp.use(session({
  secret: process.env.SESSION_SECRET || 'YOUR SECRET HERE',
  store: new MongoStore({url: mongoUrl})
}));

expressApp.use(require('body-parser')())
// we also need to add method-override 
// so that we can access PUT in the controllers
expressApp.use(require('method-override')())
```

Plug derby-auth middleware:

```js
expressApp.use(auth.middleware(strategies, options));
```

You can see that some expressjs controllers (which handle request to specific urls) are registered. Here is a short summary about controllers: 

route | method | arguments | purpose
--- | --- | --- | ---
`/login` | post | username, password | sign in using login and password
`/register` | post | username, email, password | sign up using login and password
`/auth/:provider`<br>`/callback` | get | - | the user is redirected to authorization page in the social network<br>(path example: `/auth/github`)
`/auth/:provider`<br>`/callback` | get | - | social network returns control here after authorization &mdash; `derby-auth` handles it and redirects us to `failureRedirect` or `successRedirect` url.<br>(path example: `/auth/github/callback`)
`/logout` | get | - | when we make this kind of request we<br>1) log out<br>2) are redirected to `/`  page
`/password-reset` | post | email | new password is sent to user email (this has to be an AJAX request)
`/password-change` | post | uid, oldPassword, newPassword | resetting user password (this has to be an AJAX request)

There is one more step in the instruction (optional) &mdash; plugging components (authorization/registration html and forms styles, etc.). These components are for derby v0.5 and aren’t updated for derby v0.6 yet. 

### Client’s model

Derby-auth passes the data to the client. This data contains specific information: user id, additional information which describes problems which occur during the registration and whether the user managed to authenticate or not: 

path | description
--- | ---
`_session.loggedIn` | a boolean 
`_session.userId` | user id
`_session.flash.error` | array of strings &mdash; additional messages about errors (all kinds of errors which occur during authorization/registration)
auth.{userId}.local | local registration data
auth.{userId}.{provider} | registration via social network data

These paths are available in the templates and we will use them to define the status of authorization. 

### Example application

Let’s plug bootstrap. Execute this command in the console to install bootstrap v3.0:

```bash
npm i d-bootstrap -S
```

Add this line `to src/app/index.js`:

```bash
app.use(require('d-bootstrap'));
```

There are going to be 2 pages in the application:
 
1 `/` (there will be authorization data and the authorization status &mdash; it will tell us if the user logged in)
2 `/login` (there will be registration and authorization form)

Before these pages are rendered I will fetch authorization data for the auths collection. 
Here is derby application `.js` file (`src/app/index.js`):

```js
var derby = require('derby');
var app = module.exports = derby.createApp('auth', __filename);

global.app = app;

app.use(require('d-bootstrap'));


app.loadViews (__dirname+'/../../views');
app.loadStyles(__dirname+'/../../styles');

app.get('*', function(page, model, params, next){

  var user = 'auths.' + model.get('_session.userId');
  model.subscribe(user, function(){
    model.ref('_page.user', user);
    next();
  });

});

app.get('/', function (page, model){
  page.render('home');
});

app.get('/login', function (page, model){
  page.render('login');
});
```

Let’s create a layout with the menu in `index.html` file. We put the page with the information about user status into `home.html` and `login.html`.

<div class='code-caption'>index.html</div>

```js
<import: src="./home">
<import: src="./login">


<Title:>
  Derby-auth example

<Body:>
  <div class="navbar navbar-inverse" role="navigation">
    <div class="container">
      <div class="navbar-header">
        <a class="navbar-brand" href="/">Derby-auth Example</a>
      </div>
      <div class="collapse navbar-collapse">
        <ul class="nav navbar-nav">
          <view name="nav-link" href="/">Информация</view>
          <view name="nav-link" href="/login">Логин</view>
        </ul>
      </div>
    </div>
  </div>


  {{if _session.flash.error}}
    <div class="container">
      {{each _session.flash.error as #error}}
        <div class="alert alert-warning">{{#error}}</div>
      {{/}}
    </div>
  {{/}}


  <view name="{{$render.ns}}"></view>

<nav-link: element="nav-link">
  <li class="{{if $render.url === @href}}active{{/}}">
    <a href="{{@href}}">{{@content}}</a>
  </li>
```  

This menu and most of html in this file have little to do with the authorization (except alerts). 
Let’s look at this line:

```html
<view name="{{$render.ns}}"></view>
```

Derby is going to replace this line with the contents of `home.html` or `login.html`. It depends on what we specified in the controller: 

`page.render('home')` or `page.render('login')`

`home.html` (the page with user status information)
Page with the additional information &mdash; `home.html`:

```js
<index:>
  <div class="container">

    <h1>Information:</h1>
    {{if _session.loggedIn}}
      <p>The user logged in</p>


      <h2>Local strategy:</h2>
      {{if _page.user.local}}
        <p>User name:: <b>{{_page.user.local.username}}</b></p>
      {{else}}
        <p>The field is emptyт</p>
      {{/}}

      <h2>GitHub:</h2>
      {{if _page.user.github}}
        <p>User name: <b>{{_page.user.github.username}}</b></p>
      {{else}}
        <p>The field is empty</p>
      {{/}}

      <a class="btn btn-danger" href="/logout">Выйти</a>
    {{else}}
      <p>The user did NOT log in </p>
      Page <a href="/login">Registration/Authorization</a>
    {{/}}
  </div>
```

Note that “Log out” button links to the `/logout` url which is handled by `derby-auth`. 
Let’s take the form with the styles directly from the boostrap website (there is a great example of ‘sign in’) -- http://getbootstrap.com/examples/signin/
Styles are copied from the bootstrap example:
 
<div class='code-caption'>styles/index.styl</div> 

```scss
.form-signin {
  max-width: 330px;
  padding: 15px;
  margin: 0 auto;
}
.form-signin .form-signin-heading,
.form-signin .checkbox {
  margin-bottom: 10px;
}
.form-signin .checkbox {
  font-weight: normal;
}
.form-signin .form-control {
  position: relative;
  height: auto;
  -webkit-box-sizing: border-box;
  -moz-box-sizing: border-box;
  box-sizing: border-box;
  padding: 10px;
  font-size: 16px;
}
.form-signin .form-control:focus {
  z-index: 2;
}
.form-signin input[type="email"] {
  margin-bottom: -1px;
  border-bottom-right-radius: 0;
  border-bottom-left-radius: 0;
}
.form-signin input[type="password"] {
  margin-bottom: 10px;
  border-top-left-radius: 0;
  border-top-right-radius: 0;
}
```

Here is `login.html` with authorization and registration:

<div class='code-caption'>login.html</div>

```js
<index:>

  <view name="signin"></view>
  <hr/>
  <view name="signup"></view>

<signin:>
  <div class="container">
    <form class="form-signin" role="form" action='/login' 
          method='post'>
      <h3 class="form-signin-heading">Sign in</h3>
      
      <input name="username" type="text" class="form-control" 
             placeholder="name" required="" autofocus="">
      <input name="password" type="password" class="form-control"
             placeholder="password" required="">

      <button class="btn btn-lg btn-primary btn-block" type="submit">
        Login
      </button>
      <br/>
      
      <a class="btn btn-lg btn-danger btn-block" href="/auth/github">
        via GitHub
      </a>
      
    </form>
  </div>

<signup:>
  <div class="container">
    <form class="form-signin" role="form" action='/register' 
          method='post'>
      <h3 class="form-signin-heading">Sign up</h3>
      
      <input name="username" type="text" class="form-control" 
             placeholder="name" required="" autofocus="">
      <input name="email" type="email" class="form-control" 
             placeholder="e-mail" required="">
      <input name="password" type="password" class="form-control"
             placeholder="password" required="">

      <button class="btn btn-lg btn-primary btn-block" type="submit">
        Sign up
      </button>
      
    </form>
  </div>
```

This is how the application looks like: 

![Derby login application](/img/tutorials/derby3/3.png)

I’ve edited bootstrap form a bit. Note there is a post form in the authorization form for `/login` which is handled by derby-auth. Github button submits posts requests to `/auth/github` which is also handled by derby-auth. As for the registration form, there are post requests to `/register`. 
Let’s run the application:

```bash
npm start
```

Check the user status &mdash; “the user did NOT log in”. Register and now we can see that the status is “The user logged in”. We can log out &mdash; `/logout`. While we are logged in we can connect github account to current user profile &mdash; we need to click “Sign in with GitHub”. We will be redirected to github where we will sign in and then we will get back to the website.
 
`derby-auth` allows to sign in on the website via social networks without registering via local strategy. 
