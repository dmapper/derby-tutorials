# Tutorial. Multiplayer game on Derby.js

In this tutorial we're going to create a simple online multiplayer game on Derby.js

## The game

The professor creates a question: “Guess the number of jelly beans in the jar.” Everyone has to enter. Once the professor closes the game, the average is displayed to everyone. No more people can enter the game. People can enter the game without a login.

## Generating project

Create a project using `generator-derby`:
 
```bash 
$ sudo npm install -g yo
$ sudo npm install -g generator-derby
$ mkdir my-app
$ cd my-app
$ yo derby --coffee
$ npm start
``` 

## Creating game

We want to create a game which has a name and a question in it. Create two inputs and a ‘Create new game!’ button, wrapped in a form which has a ‘post’ method.

<div class='code-caption'>home.jade</div>

```haml
form(method='post', action='/create_game')
  p Type the name of the game:
  input(type='text', name='name')
  p Type the question:
  input(type='text', name='question')
  br
  br
  button Create new game!
```  

![Derby game create new game page](/img/tutorials/derby5/1.png)

Write a controller that handles `/create_game` action.

<div class='code-caption'>index.coffee</div>

```coffee
app.post '/create_game', (page, model, params) ->
  model.add 'games', {
    name: params.body.name
    question: params.body.question
    players: {}
    userIds: []
  }, ->
    page.redirect '/'
```    

Here we create a game: add a new `games` document with a `name`, a `question`, an empty object with `players` and an array with `userIds` (we need them to easily loop through all players in the game). model.add also generates an id for each game. As for form variables (`name`, `question`), we get them  from `params.body` in the controller. We redirect to `/` page, because we want to stay on the same page after creating the game.
Let’s display the games on the home page. Each game will be a link. 

## List of games

To be able to loop through the games in `games` collection (in order to display the list of games) we need to create a local db out of `games`, convert it to an array and make a reference in private `_page` collection.

<div class='code-caption'>index.coffee</div>

```coffee
app.get '/', (page, model) ->
  model.subscribe 'games', ->
    model.at('games').filter().ref '_page.games'
    page.render 'home'
```

We are subscribing for `games` collection to make it accessible on `/` page. `model.at` creates a local `games` db, `filter()` converts `games` to an array and `ref()` creates a reference from this array. Now we can loop through the array of games in views: 

<div class='code-caption'>home.jade</div>

```haml
p Games:
each #root._page.games as #game
  a(href='/game/{{#game.id}}') {{#game.name}}
```  

We are displaying each game as a link which will redirect us to the game page. id and name are the fields in game object. 
Let’s add a game.

![Derby multiplayer game tutorial. New game creation](/img/tutorials/derby5/2.png)
  
## Deleting games

To delete all the games add a ‘Delete all games!’ input.

<div class='code-caption'>home.jade</div>

```haml
input(on-click='deleteGames()', type='submit', 
      value='Delete all games!') 
```

When clicking it, `deleteGames` function is triggered. 

<div class='code-caption'>index.coffee</div>

```coffee
app.proto.deleteGames = ->
  games = @model.get 'games'
  for key of games
    @model.del "games.#{key}"
```

We need to loop through games deleting each game from `games` collection. It is necessary to subscribe for `games` and set it into a variable. `for of` loop goes through each game and `model.del` deletes each game. Notice that in the last line we used double quotes. String in double quotes supports interpolation. In this case, we interpolate key, that is game id.

## Adding player’s name 

Let’s add player’s name.

<div class='code-caption'>home.jade</div>

```haml
p Player's name:
form(on-submit='setUserName()')
  input(type='text', value='{{#root._page.userName}}'')
  br
  br
  button(type=’submit’) Save!
```  

We are binding the value of the input. Submitting the form triggers `setUserName` function. 

![Tutorial on Derby multiplayer game. Name of player](/img/tutorials/derby5/3.png)

<div class='code-caption'>index.coffee</div>

```coffee
app.proto.setUserName = ->
  userId = this.model.get '_session.userId'
  userName = this.model.get '_page.userName'
  if userName?
    userName = userName.trim()
  if userName
    this.model.set 'users.' + userId + '.userName', userName
    this.model.del '_page.userName'
  else
    alert 'Name cannot be empty!'
```    

We get `userName` from `_page.userName`, trim and set it to `users` collection. If an empty input is submitted we show "Name cannot be empty!" alert message
Also, let’s greet the player. To do this, we subscribe for current user (not for players, because name is stored in `users` collection). We also need to get userId, which is stored in private collection `_session.userId` and set it in a variable to be able to subscribe for current user.  

<div class='code-caption'>index.coffee</div>

```coffee
app.get '/', (page, model) ->
   userId = model.get ‘_session.userId’
   model.subscribe 'games', ‘users.’ + userId,  ->
     model.ref ‘_page.user’, ‘users.’ + userId
     model.at('games').filter().ref '_page.games'
     page.render 'home'
```

There is a  reference (`model.ref`) on the current user and now we can check in views if the user has name. If he does, the greeting appears on the page.
 
```haml 
p Player's name:
form(on-submit='setUserName()')
  input(type='text', value='{{#root._page.userName}}'')
  br
  br
  button(type=’submit’) Save!
  if #root._page.user.userName
    p Hello, {{#root.users[#root._session.userId].userName}}
```

![Tutorial Derby.js multiplayer online game](/img/tutorials/derby5/4.png)

## Game page

Add `game.jade` file to `/views/app` and import it to `index.jade`

<div class='code-caption'>game.jade</div>

```haml
index: 
```

<div class='code-caption'>index.jade</div>

```haml
import:(src='./home')
import:(src='./game')

Title:
  | {{_page.title}}

Body:
  view(name='header')
  view(name='{{$render.ns}}')
  view(name='footer', year='2014')

header:
  h1 Header

footer:
  p Footer {{@year}}
```

Create a controller that handles game page.

<div class='code-caption'>index.coffee</div>

```coffee
app.get '/game/:gameId', (page, model, params) ->
  page.render ‘game’
```  

## Display the question

We want to display the question in the game. To do that, subscribe for the game (and make a reference for easy access), set the game id and make a reference, so that we can access the game id in views.

```coffee
app.get '/game/:gameId', (page, model, params) ->
  model.subscribe ‘games.’ + params.gameId, ->
    model.set ‘_page.gameId’, params.gameId 
    page.render ‘game’
```

Here comes the views.

<div class='code-caption'>game.jade</div>

```haml
index:  
|+  p {{#root.games[#root._page.gameId].question}}
```
  
Add input for the answer

<div class='code-caption'>game.jade</div>

```haml
index:  
  p {{#root.games[#root._page.gameId].question}}
|+  form(on-submit='setAnswer()')
|+    input(type='text', value='{{#root._page.answer}}')
|+    br
|+    button Save
```    

Let’s subscribe for the users in the game (because we want to display players in the game) and make references to  players, current user, current game and current player. To subscribe for users in the game, we need to make a query request. 

```coffee
app.get '/game/:gameId', (page, model, params) ->
  model.subscribe ‘games.’ + params.gameId, ->
    model.set ‘_page.gameId’, params.gameId 
|+    usersInGame = model.query 'users', 'games.' + params.gameId + 
|+                  '.userIds'
|+    userId = model.get '_session.userId'
|+    model.subscribe usersInGame, 'users.' + userId, ->
|+      model.set '_page.gameId', params.gameId
|+      model.ref '_page.user', 'users.' + userId
|+      model.ref '_page.game', 'games.' + params.gameId
|+      model.ref '_page.players', 'games.' + params.gameId + '.players'
|+      model.ref '_page.player', 'games.' + params.gameId +
|+                '.players.' + userId
      page.render 'game'
```

## Add player to the game

<div class='code-caption'>index.coffee</div>

```coffee
app.get '/game/:gameId', (page, model, params) ->
  model.subscribe ‘games.’ + params.gameId, ->
    model.set '_page.gameId', params.gameId 
    usersInGame = model.query 'users', 'games.' + params.gameId + 
                  '.userIds'
    userId = model.get '_session.userId'
    model.subscribe usersInGame, 'users.' + userId, ->
      model.set '_page.gameId', params.gameId
      model.ref '_page.user', 'users.' + userId
      model.ref '_page.game', 'games.' + params.gameId
      model.ref '_page.players', 'games.' + params.gameId + '.players'
      model.ref '_page.player', 'games.' + params.gameId + 
                '.players.' + userId
|+      unless model.get '_page.player'
|+        model.add '_page.players', { id: userId }
|+        model.push '_page.game.userIds', userId	
      page.render 'game'
```      

When opening the game page, the player is added to the game, unless he is already there. model.add adds a new document with random id. It is possible to specify an id. We want players’ ids to be the same as users’ ids, that is why we set id as userId. When the player is added, his id is saved into game’s userIds array. 

## Save the answer 

<div class='code-caption'>index.coffee</div>

```coffee
app.proto.setAnswer = ->
  answer = this.model.get '_page.answer'
  unless answer?
    answer = ""
  answer = parseInt( answer.trim() )
  if isNaN( answer )
    alert 'The answer must be a number!'
  else
    this.model.set '_page.player.answer', answer
    this.model.del '_page.answer'
```

We are get the answer from `_page.answer` and set it to answer, checking whether it is not `undefined` or `null` and parsing it to integer. If the answer is `NaN`, we alert "The answer must be a number!". Otherwise the answer is set to `games` collection. 

Since we’ve made a reference, we don’t need to write the full path (`'games.' + params.gameId + '.players.' + userId`) &mdash; we are setting the answer to `_page.player.answer`.
We also want to display the answer on the game page. When the player changes his mind and enters a different guess, the answer on the page will be changed instantly. The answer should be displayed only if there is one, so we make an if-statement checking if there is an answer. 

<div class='code-caption'>game.jade</div>

```haml
  if #root._page.player.answer
    p Your answer is {{#root._page.player.answer}}
```    

## List of players in the game

Let’s display all players in the game. 

<div class='code-caption'>game.jade</div>

```haml
  p Players in the game
  each #root._page.game.userIds as #userId
    p {{#root.users[#userId].userName}}
```    

![Tutorial derby.js multiplayer game. List of players](/img/tutorials/derby5/5.png)

## Giving control to prof

Only the professor should be able to create a game, not the students. Add an input (type checkbox). If it is checked, `prof` field (this field is created in `_page.user` when the checkbox is checked) becomes `true`. Only when it is `true` the game can be created. 

<div class='code-caption'>home.jade</div>

```haml
index:
  label
    input(type='checkbox', checked='{{#root._page.user.prof}}')
    |  Professor
  if #root._page.user.prof
    form(method='post', action='/create_game')
      p Type the name of the game:
      input(type='text', name='name')
      p Type the question:
      input(type='text', name='question')
      br
      br
      button Create new game!
```

![Tutorial derby.js multiplayer game. Professor controls](/img/tutorials/derby5/6.png)

Besides, only the professor should be able to finish the game. 
We have already subscribed for the current user and made a reference (`_page.user`).
 
Write an `if`-statement which will check if the user is the professor. If he is, then the "Finish the game!" button will be displayed.

<div class='code-caption'>game.jade</div>

```haml
  if #root._page.user.prof
    input(on-click='finishGame()', type='submit', 
          value='Finish the game!')
```    

## Finish the game

Now let’s write the controller that handles finishing the game. The aim here is to collect the answers, strike an average and finish the game. To collect the answers, use the `for of` loop. The controller sets `_page.game.finish` to `true`. 

<div class='code-caption'>index.coffee</div>

```coffee
app.proto.finishGame = ->
  players = this.model.get '_page.players'
  average = 0
  for key, value of players
    average += value.answer
  numberOfPlayers = this.model.get('_page.game.userIds').length
  average = average / numberOfPlayers
  this.model.set '_page.game.average', average
  this.model.set '_page.game.finish', true
```  

## No going back

When the game is finished by professor, we want all the players inside the game to automatically go to the results page. To do that, we need to create a component from the view we render. For convenience, the name of the component should match the name of the template it’s associated with. 

We use component’s `create` method to execute code right after this component (in our case it’s the whole page) has been rendered. When the professor clicks "Finish the game!", `_page.game.finish` becomes `true`. So we want to listen for `_page.game.finish` variable changes. And if the game is finished, we want players to be redirected to the page with the results. 

<div class='code-caption'>index.coffee</div>

```coffee
app.component 'game', class Game
  create: ->
    gameId = this.model.root.get '_page.gameId'
    this.model.root.on 'change', '_page.game.finish', (value) ->
      app.history.replace '/game/' + gameId + '/results'
```      

New players shouldn’t be able to enter the game if it has been finished, so when going to the game page, we are checking whether the game is finished or not. 

## Results

<div class='code-caption'>index.coffee</div>

```coffee
app.get '/game/:gameId', (page, model, params) ->
|+  model.subscribe 'games.' + params.gameId, ->
|+    if model.get 'games.' + params.gameId + '.finish'
|+      page.redirect '/game/' + params.gameId + '/results'
|+      return
```      

Add `results.jade` file to `/views/app` and import it to `index.jade`: 

<div class='code-caption'>result.jade</div>

```haml
index:
```

<div class='code-caption'>index.jade</div>

```haml
import:(src='./results')
```

Here is the controller that handles results page.

<div class='code-caption'>index.coffee</div>

```coffee
app.get '/game/:gameId/results', (page, model, params) ->
  model.subscribe 'games.' + params.gameId, ->
    model.ref '_page.average', 'games.' + params.gameId + '.average'
    page.render 'results'
```

We subscribe for the game and make a reference to the average, which is stored in `games` collection. 
And, finally,  display the results on the results page. 

<div class='code-caption'>results.jade</div>

```haml
index:
  p The average is {{#root._page.average}}
```

## Final code

<div class='code-caption'>index.coffee</div>

```coffee
derby = require 'derby'

app = module.exports = derby.createApp 'app', __filename

global.app = app unless derby.util.isProduction

app.serverUse module, 'derby-jade'
app.serverUse module, 'derby-stylus'

app.loadViews __dirname + '/../../views/app'
app.loadStyles __dirname + '/../../styles/app'

app.get '/', (page, model) ->
  userId = model.get '_session.userId'
  model.subscribe 'games', 'users.' + userId, ->
    model.ref '_page.user', 'users.' + userId
    model.at('games').filter().ref '_page.games'
    page.render 'home'

app.post '/create_game', (page, model, params) ->
  model.add 'games', {
    name: params.body.name
    question: params.body.question
    players: {}
    # to have access to all players in the game
    userIds: []
  }, ->
    page.redirect '/'

app.get '/game/:gameId', (page, model, params) ->
  model.subscribe 'games.' + params.gameId, ->
    if model.get 'games.' + params.gameId + '.finish'
      page.redirect '/game/' + params.gameId + '/results'
      return
    model.set '_page.gameId', params.gameId
    usersInGame = model.query 'users', 'games.' + params.gameId + 
                  '.userIds'
    userId = model.get '_session.userId'
    model.subscribe usersInGame, 'users.' + userId, ->
      userId = model.get '_session.userId'
      model.set '_page.gameId', params.gameId
      model.ref '_page.user', 'users.' + userId
      model.ref '_page.game', 'games.' + params.gameId
      model.ref '_page.players', 'games.' + params.gameId + '.players'
      model.ref '_page.player', 'games.' + params.gameId +
                '.players.' + userId
      unless model.get '_page.player'
        model.add '_page.players', { id: userId }
        model.push '_page.game.userIds', userId

      page.render 'game'

app.get '/game/:gameId/results', (page, model, params) ->
  model.subscribe 'games.' + params.gameId, ->
    model.ref '_page.average', 'games.' + params.gameId + '.average'
    page.render 'results'

app.proto.deleteGames = ->
  games = @model.get 'games'
  for key of games
    @model.del "games.#{key}"

app.proto.setUserName = ->
  userId = this.model.get '_session.userId'
  userName = this.model.get '_page.userName'
  if userName?
    userName = userName.trim()
  if userName
    this.model.set 'users.' + userId + '.userName', userName
    this.model.del '_page.userName'
  else
    alert 'Name cannot be empty!'

app.proto.setAnswer = ->
  answer = this.model.get '_page.answer'
  unless answer?
    answer = ""
  console.log answer
  answer = parseInt( answer.trim() )
  if isNaN( answer )
    alert 'The answer must be a number!'
  else
    this.model.set '_page.player.answer', answer
    this.model.del '_page.answer'

app.proto.finishGame = ->
  players = this.model.get '_page.players'
  average = 0
  for key, value of players
    average += value.answer
  numberOfPlayers = this.model.get('_page.game.userIds').length
  average = average / numberOfPlayers
  this.model.set '_page.game.average', average
  this.model.set '_page.game.finish', true

app.component 'game', class Game
  create: ->
    gameId = this.model.root.get '_page.gameId'
    this.model.root.on 'change', '_page.game.finish', (value) ->
      app.history.replace '/game/' + gameId + '/results'
```

<div class='code-caption'>home.jade</div>

```haml
index:
  label
    input(type='checkbox', checked='{{#root._page.user.prof}}')
    |  Professor
    if #root._page.user.prof
      form(method='post', action='/create_game')
        p Type the name of the game:
        input(type='text', name='name')
        p Type the question:
        input(type='text', name='question')
        br
        br
        button Create new game!

  p Player's name:
  form(on-submit='setUserName()')
    input(type='text', value='{{#root._page.userName}}', name='name')
    br
    br
    input(type='submit', value='Save!')
    if #root._page.user.userName
      p Hello, {{#root.users[#root._session.userId].userName}}

  p Games:
  each #root._page.games as #game
    a(href='/game/{{#game.id}}') {{#game.name}}
    br
  br

  input(on-click='deleteGames()', type='submit', 
        value='Delete all games!')
```

<div class='code-caption'>game.jade</div>

```haml
index:
  p Game
  p {{#root.games[#root._page.gameId].question}}
  form(on-submit='setAnswer()')
    input(type='text', value='{{#root._page.answer}}')
    br
    button Save
  br
  if #root._page.player.answer
    p Your answer is {{#root._page.player.answer}}

  p Players in the game
  each #root._page.game.userIds as #userId
    p {{#root.users[#userId].userName}}

  if #root._page.user.prof
    input(on-click='finishGame()', type='submit', 
          value='Finish the game!')
```

<div class='code-caption'>results.jade</div>

```haml
index:
  p The average is {{#root._page.average}}
```  

