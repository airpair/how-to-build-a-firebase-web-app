This is a quick tutorial on how to build a basic web application using  Firebase data API.

I'm going to build RSS subscription application (oh yes, it's not a TODO app!) that allows a user to manage website subscriptions (frontend), tracks updates via RSS and sends updates to the users email (backend).

There are three parties that will be involved:

1. Frontend application written in [CoffeeScript](http://coffeescript.org/) with the help of [jQuery](http://jquery.com/).
2. [Firebase](https://www.firebase.com) database setup and hosting for the web application code.
3. Backend [Node.js](http://nodejs.org/) application hosted on [Heroku](http://www.heroku.com/). This is basically a cron job script that tracks RSS feeds and sends email updates.


## Prerequisites

You'll need the following things to build this out for yourself:

1. Please register [Firebase](https://www.firebase.com/signup/) account (free) and create a new application via account console.
2. [Heroku](https://signup.heroku.com/www-header) is used to deploy backend notification code, register if you don't have one (CC required), the hosting of the app is free of charge.
3. [CodeKit](https://incident57.com/codekit/) ($30, trial is available which is enough) is used to compile CoffeeScript code for both frontend and backend, it also provides localhost web server that is used with frontend app development. There are lots of console options to this, but for beginner this tools is helpful.
4. You need to understand CoffeeScript (and be comfortable with Javascript).

Having all that ready let's jump to the code.


## Frontend

Please check out the deployed app to see what we are about to build:  [rss-rocks.firebaseapp.com](http://rss-rocks.firebaseapp.com/).

Sources can be found on github: ​[https://github.com/slate-studio/rss.rocks-firebase](https://github.com/slate-studio/rss.rocks-firebase).

First let's create an HTML foundation for the app — create a folder and pull [html5-boilerplate](https://github.com/h5bp/html5-boilerplate) repo from github:

```bash
$ mkdir firebase-app
$ cd firebase-app
$ git clone https://github.com/h5bp/html5-boilerplate.git ./
```
Remove everything except ```index.html```, ```/css```, ```/js/main.js``` and ```/js/vendor``` — these are all we are interested in at this point.

Now let's add this folder to the CodeKit app. This will help to compile CoffeeScript sources and test on localhost. Here is how the project structure looks like in the app:

![CodeKit Project Init](//slate13-media.s3.amazonaws.com/uploads/character/images/53d0f4ff6336366f90000000/regular_firebase-app-codekit.png)

The green Server button provides you with links to access application on your local machine (```localhost```). Create ```/coffee``` folder and applications coffee source files like on a screenshot:

```bash
$ mkdir coffee
$ cd coffee
$ touch _main.coffee app.coffee authentication.coffee dashboard.coffee subscription.coffee
```

```/coffee/_main.coffee``` compiles to ```/js/main.js``` which is included in ```index.html```. Here is the compilation configuration for CodeKit:

![CodeKit Compilation Configuration 1](//slate13-media.s3.amazonaws.com/uploads/character/images/53d0f6b96336366fbd000000/regular_firebase-app-codekit-2.png)

![CodeKit Compilation Configuration 2](//slate13-media.s3.amazonaws.com/uploads/character/images/53d0f6bb6336366f90010000/regular_firebase-app-codekit-3.png)

At this point all maintenance is done and we're ready to begin writing code.

At first we need to include the Firebase javascript libraries that we're going to use. The first one is the Firebase javascript client to access data via API and the second one is authentication library, that helps to add users to the app in a very simple way.

There is also a ```div``` element included with ```app``` id, this is to be used in app further. Here is how ```index.html``` looks like after these libraries are in place:

```markup,linenums=true
<!doctype html>
<html class="no-js" lang="">
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1">

        <title>RSS.rocks!</title>
        <meta name="description" content="Delivers RSS feed updates to an email">

        <link rel="stylesheet" href="css/normalize.css">
        <link rel="stylesheet" href="css/main.css">
        
        <script src="js/vendor/modernizr-2.8.0.min.js"></script>
        <script type='text/javascript' src='https://cdn.firebase.com/js/client/1.0.17/firebase.js'></script>
        <script type='text/javascript' src='https://cdn.firebase.com/js/simple-login/1.6.1/firebase-simple-login.js'></script>
    </head>
    <body>
        <div id='app'></div>
        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js"></script>
        <script>window.jQuery || document.write('<script src="js/vendor/jquery-1.11.1.min.js"><\/script>')</script>
        <script src="js/main.js"></script>
    </body>
</html>
```


### _main.coffee

Let's start with ```/coffee/_main.coffee``` file. This integrates all application pieces (via CodeKit as mentioned before), sets up the Firebase app name and runs the application code. Nothing fancy here:

```coffeescript
$ ->
  firebaseAppName = 'rss-rocks'
  window.app = new App(firebaseAppName)
```

**rss-rocks** is the Firebase application name in my case, for you it should be something that you've chosen or some random string provided by Firebase as default app.

Worth it to say that we created an ```app``` object in a ```window``` global namespace, to make it possible to access its methods from any part of the code.


### app.coffee

This implements the ```App``` class that initializes root Firebase reference, renders all app views: Dashboard and Authentication, — sets the ```#app``` element which views are attached to and provides the ```show``` method that helps to switch between them.

```coffeescript,linenums=true
class App
  constructor: (@firebaseAppName) ->
    @firebase = new Firebase("https://#{@firebaseAppName}.firebaseio.com")
    @ui()
    @render()
  
  ui: ->
    @$el =$ '#app'
  
  render: ->
    @dashView = new Dashboard @$el, @firebase
    @authView = new Authentication @$el, @firebase, (@user) =>
      @show(@dashView)
  
  show: (view) ->
    @currentView?.hide()
    @currentView = view
    @currentView.show()
```

The Authentication view implements auth logic for users and gets a callback as the last parameter that's called when the user is logged in — it shows the users dashboard.


### authentication.coffee

Firebase handles most of the authentication for you. In our app, login via email is used. This feature has to be enabled in app settings, go to your firebase account page: [https://www.firebase.com/account](https://www.firebase.com/account) and select app. There in Simple Login tab select "Email & Password" section and check "Enabled" checkbox.

![Firebase Authentication Enable](//slate13-media.s3.amazonaws.com/uploads/character/images/53d119976336366f90020000/regular_firebase-app-simplelogin.png)

Authentication class implements UI for sign in/up/out functionality and integrates Firebase user auth mechanics.

```coffeescript,linenums=true
class Authentication
  constructor: (@$rootEl, @firebase, @onAuthCb) ->
    @usersRef = @firebase.child('users')
    @_render()
    @_ui()
    @_bind()
    @_authenticate()
  
  _render: ->
    @$rootEl.append """
      <section id='auth' style='display:none;'>
        <p>Welcome to <strong>RSS.rocks</strong>!</p>
        <form id='auth_form'>
          <input id='auth_email' value='' placeholder='email' type='email' required>
          <input id='auth_password' value='' placeholder='password' type='password' required>
          <input id='auth_submit' value='login' type='submit'>
          <span id='auth_loading' style='display:none;'>please wait...</span>
        </form>
        <p id='auth_error' class='error'></p>
        <p id='auth_signup_info'>&mdash; if you have an account, please <a id='auth_login_btn' href='#'>login</a></p>
        <p id='auth_login_info'>&mdash; if you don't have an account, please <a id='auth_signup_btn' href='#'>signup</a><br>
          &mdash; if you forgot your password, <a id='auth_reset_password' href='#'>reset</a> it via an email</p>
      </section>
    """
  
  _ui: ->
    @$el         =$ '#auth'
    @$form       =$ '#auth_form'
    @$email      =$ '#auth_email'
    @$password   =$ '#auth_password'
    @$submitBtn  =$ '#auth_submit'
    @$loading    =$ '#auth_loading'
    @$error      =$ '#auth_error'
    @$loginBtn   =$ '#auth_login_btn'
    @$signupBtn  =$ '#auth_signup_btn'
    @$resetBtn   =$ '#auth_reset_password'
    @$loginInfo  =$ '#auth_login_info'
    @$signupInfo =$ '#auth_signup_info'
  
  _bind: ->
    @$loginBtn.on  'click',  (e) => @showLogin()          ; false
    @$signupBtn.on 'click',  (e) => @showSignup()         ; false
    @$resetBtn.on  'click',  (e) => @resetPassword()      ; false
    @$form.on      'submit', (e) => @formSubmitCallback() ; false
  
  _authenticate: ->
    @auth = new FirebaseSimpleLogin @firebase, (error, user) =>
      @$loading.hide()
      if user
        @onAuthCb?(user)
      else
        app.show(this)
      if error
        @error(error.message.replace('FirebaseSimpleLogin: ', ''))
  
  signup: ->
    email    = @$email.val()
    password = @$password.val()
    @$loading.show()
    @auth.createUser email, password, (error, user) =>
      @$loading.hide()
      if error
        @error(error.message.replace('FirebaseSimpleLogin: ', ''))
      else
        @usersRef.child(user.uid).child('email').set(user.email)
        @onAuthCb?(user)
  
  login: ->
    @$loading.show()
    @auth.login 'password', {
      email: @$email.val(),
      password: @$password.val(),
      rememberMe: true
    }
  
  show: ->
    @showLogin()
    @$el.show()
    @$email.focus()
  
  hide: ->
    @$el.hide()
  
  showLogin: ->
    @formSubmitCallback = @login
    @$submitBtn.val('login')
    @$loginInfo.show()
    @$signupInfo.hide()
    @$error.html('')
  
  showSignup: ->
    @formSubmitCallback = @signup
    @$submitBtn.val('signup')
    @$signupInfo.show()
    @$loginInfo.hide()
    @$error.html('')
  
  logout: ->
    @auth.logout() ; delete app.user
    @$email.val('')
    @$password.val('')
    @$error.html('')
    app.show(this)
  
  error: (msg) ->
    @$error.html(msg)
  
  resetPassword: ->
```

```_render```, ```_ui``` and ```_bind``` methods creates UI elements maps them to object attributes and bind events to controls.

```_authenticate``` method creates the handler for firebase auth result and check if user is signed in. If user is signed in it calls callback passed in application, which shows Dashboard view otherwise Authentication view is shown.

```login``` method is triggered when user fills in auth fields and click login. It only sends auth parameters to Firebase server, no callbacks here. The one defined in ```_authenticate``` function is used to process Firebase simple login reply.

```signup``` method creates a new user. This one has a callback that should process errors like *"email is already taken"*. If a user is created they are automatically authorized and the users email is saved to the database so we know where to send updates. Here is the line that does the trick, I'll explain it in the Dashboard section:

```coffeescript
@usersRef.child(user.uid).child('email').set(user.email)
```

This is pretty much everything noteworthy regarding authentication. As you can see there are only three functions required to implement user sign in/up feature. There are also ```logout``` and ```resetPassword``` (to be implemented) methods but they are pretty trivial and self explanatory.

Now we have the user authenticated, let's move on to the Dashboard.


### dashboard.coffee

I would consider this the most interesting part. The Dashboard shows a list of users subscriptions and provides a way to add a new one or remove an existing one.

But before the piece of code let's see how data is structured in database and why.

In Firebase database you're allowed to store objects in JSON format (hashes key-value) and arrays of objects. There is one limitation though: keys should not contain special character, so say email or url can't be a key — this is different from what we have in Javascript where any string could be a key.

This limitation is there because Firebase makes it possible to browse stored objects with url.

E.g. ```/users/alex/profile/name/first``` — gives you value from the bottom of the object:

```json
{
  'users': {
    'alex': {
      'profile': {
        'name': {
          first: 'alex',
          last:  'kravets'
        }
      }
    }
  }
}
```

Example is simple but you get the point.

In our case we need to store two things:
1. users email where to send RSS updates;
2. list of subscriptions (title and rss feed url).

Here is the structure I've ended up with:

![Firebase Data Scheme Demo](//slate13-media.s3.amazonaws.com/uploads/character/images/53d123b06336366fbd010000/regular_firebase-app-data.png)

```simplelogin:3``` is a unique user id which is provided for each authorized user by Firebase simple login. So for every signed in user we know which subscriptions to show.

```cache``` object is not important at this point. It's used on backend while sending emails, we will talk about it later.

```coffeescript,linenubers=true
class Dashboard
  constructor: (@$rootEl, @firebase) ->
    @subscriptionCollection = []
    @_render()
    @_ui()
    @_bind()
  
  _render: ->
    @$rootEl.append """
      <section id='dash' style='display:none;'>
        <p><span id='dash_email'></span> &mdash; <a id='dash_logout_btn' href='#'>logout</a></p>
        <div id='subscriptions'>
          <p>
            <form id='subscriptions_new_form'>
              <input id='subscriptions_new_name' placeholder='name' />
              <input id='subscriptions_new_url' placeholder='rss feed link' type='url' />
              <input id='subscriptions_new_submit' value='Add' type='submit' />
            </form>
          </p>
          <ul id='subscriptions_list' class='subscriptions-list'></ul>
        </div>
      </section>
    """
  
  _ui: ->
    @$el        =$ '#dash'
    @$email     =$ '#dash_email'
    @$logoutBtn =$ '#dash_logout_btn'
    @$newForm   =$ '#subscriptions_new_form'
    @$newUrl    =$ '#subscriptions_new_url'
    @$newName   =$ '#subscriptions_new_name'
    @$list      =$ '#subscriptions_list'
  
  show: ->
    @uid       = app.user.uid
    @userEmail = app.user.email
    @$email.html(@userEmail)
    @subscriptionsRef = @firebase.child('users').child(@uid).child('subscriptions')
    @subscriptionsRef.on 'child_added', (snapshot) =>
      @subscriptionCollection.push(new Subscription(@$list, snapshot))
    @$el.show()
  
  hide: ->
    $.each @subscriptionCollection, (i, el) -> el.destroy()
    @subscriptionsRef.off()
    delete @uid
    delete @userEmail
    delete @subscriptionsRef
    @$el.hide()
  
  _bind: ->
    @$logoutBtn.on 'click',  (e) -> app.authView.logout() ; false
    @$newForm.on   'submit', (e) => @addNewSubscription() ; false
  
  addNewSubscription: ->
    url  = @$newUrl.val()  ; @$newUrl.val('')
    name = @$newName.val() ; @$newName.val('')
    @subscriptionsRef.push({url: url, name: name})
    @$newName.focus()
```

The most noteworthy thing from the code above happens in the ```show``` method:

```coffeescript
@subscriptionsRef = @firebase.child('users').child(@uid).child('subscriptions')
@subscriptionsRef.on 'child_added', (snapshot) =>
  @subscriptionCollection.push(new Subscription(@$list, snapshot))
```

First we get the Firebase data reference ```@subscriptionRef``` to user subscriptions collection. This reference object is used to access data and bind events. We bind 'child_added' event. Whenever new item put into collection our handler is called. It creates a new view for every new item.

```child_added``` event triggered also when app loads data stored in collection while init. This allows to process these two cases with just one handler.

We add new subscription items to collection using ```@subscriptionRef``` reference object with a ```push``` method:

```coffeescript
@subscriptionsRef.push({url: url, name: name})
```

After new item successfully added to collection ```child_added``` event triggered and new UI element is built for it with the help of Subscription class, special Firebase object ```@snapshot``` is passed through.

When user logs out ```hide``` method removes all users data, references and ```child_added``` event bind.


### subscription.coffee

Subscription class is used to add UI element representing subscription item and provides a way to remove subscription. ```@snapshot``` object, which is passed as parameter to ```child_added``` callback, has two methods ```val``` and ```ref```.

```val``` returns javascript object in our case it is a hash with name and url fields.

```ref``` returns firebase data reference which is used when we need to delete subscription.

```coffeescript,linenumbers=true
class Subscription
  constructor: (@$rootEl, @snapshot) ->
    @_render()
    @_ui()
    @_bind()
  
  _render: ->
    @ref  = @snapshot.ref()
    @data = @snapshot.val()
    @$el = $ """
      <li class='subscriptions-list-item'>
        <a href='#' class='subscriptions-remove-button'>&times;</a> #{@data.name} &mdash; <a href='#{@data.url}' target='_blank'>#{@data.url}</a>
      </li>
    """
    @$rootEl.append @$el
  
  _ui: ->
    @$deleteBtn = @$el.find('.subscriptions-remove-button')
  
  _bind: ->
    @$deleteBtn.on 'click',      (e) => @remove() ; false
    @$deleteBtn.on 'mouseenter', (e) -> $(e.currentTarget).parent().addClass 'striked-out'
    @$deleteBtn.on 'mouseleave', (e) -> $(e.currentTarget).parent().removeClass 'striked-out'
  
  _unbind: ->
    @$deleteBtn.off()
  
  remove: ->
    if confirm('Remove subscription?')
      @ref.remove()
      @destroy()
  
  destroy: ->
    @_unbind()
    @$el.remove()
```

There are two methods that semantically similar but serve different purpose:

```remove``` is used when user wants to remove a subscription from the database.

```destroy``` is called when user logs out of application and all his data should be wiped up from the browser DOM.


### main.css

Some CSS styles to be inserted to ```/css/main.css``` that makes the thing look better.

```css
body { margin: 0 15px; }
.error { color: red; }
.subscriptions-remove-button { text-decoration: none; visibility: hidden; }
.subscriptions-list-item:hover .subscriptions-remove-button { visibility: visible; }
.subscriptions-list { padding-left: 0; list-style-type: none; }
.subscriptions-list-item.striked-out { text-decoration: line-through; }
```


## Deploy

Deploying of the app to firebase hosting is a piece of cake (assuming you're in applications folder):

```sh
$ npm install -g firebase-tools
$ firebase init
$ firebase deploy
```

App should be available via URL: ```your-app-name.firebaseapp.com``` — and look like this:

![RSS.Rocks! App Screenshot 1](//slate13-media.s3.amazonaws.com/uploads/character/images/53d220a663363678f3000000/regular_firebase-app-login.png)

![RSS.Rocks! App Screenshot 2](//slate13-media.s3.amazonaws.com/uploads/character/images/53d220a863363678c4000000/regular_firebase-app-subscriptions.png)


## Backend

On the backend we need to make a cron job that checks rss feeds for new items and send updates to subscribed users via email. This is implemented with node.js and deployed on Heroku.

Sources on github: [rss.rocks-nodejs](https://github.com/slate-studio/rss.rocks-nodejs).

After creating an app on Heroku, there are two plugins to be added to the project:
  1. Scheduler: [https://addons.heroku.com/scheduler](https://addons.heroku.com/scheduler)
  2. Postmark: [https://addons.heroku.com/postmark](https://addons.heroku.com/postmark)

The first one is to run  a cron job and the second one sends emails. You'll need to do some domain configuration to send emails from a custom domain email address.

Please checkout the [repository](https://github.com/slate-studio/rss.rocks-nodejs) for implementation details.

[Slate Studio](http://www.slatestudio.com) @ 2015
