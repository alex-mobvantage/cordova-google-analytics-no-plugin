# How to setup Google Analytics for your Phonegap/Cordova app with no plugins

If you're like me and hate juggling a slew of plugin dependencies, you'll agree the simplest solution with the least amount of code is usually the best one.

I recently tried integrating Google Analytics for a Phonegap/Cordova app, expecting things to _just work_. And though I did have to spend a few hours debugging, surprisingly, with a few small tweaks, you'll be able to use Google's standard analytics Javascript snippet to start tracking your users.

# Step 1

Paste the standard Google analytics snippet into your app's `index.html` file. At the time of this writing, that looks something like this:

```html
<script>
(function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
(i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
})(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

ga('create', 'UA-XXXXX-Y', 'auto');
ga('send', 'pageview');
</script>
```

# Step 2

Here's where you'll need to make a few tweaks.

Replace the call to `create` with this:

```diff
-ga('create', 'UA-XXXXX-Y', 'auto');
+ga('create', 'UA-XXXXX-Y', {'storage':'none', 'clientId': localStorage.getItem('gaClientId')});
```

Here, `auto` tells analytics to use cookies to track the user, using the current URL to figure out the domain. Unfortunately since this is a Cordova app, we're inside a WebView which doesn't support cookies. Instead, we're saying to turn off tracking via cookies and use a clever localStorage workaround to grab the unique client ID. Note that if we didn't assign the `clientId`, we would generate a new active user each time the user opens the app. This is because normally the `clientId` is randomly generated everytime (or taken from the cookie session).

Now you'll want to add a line directly underneath like this:
```js
ga('set', 'checkProtocolTask', null);
```

Normally, Google Analytics looks at the current location's protocol (in our case `file://`) and blocks anything that's not `http://` or `https://`. The line above disables this check.

Next we need to add the following:
```js
ga(function(t){ localStorage.setItem('gaClientId', t.get('clientId')); });
```

This tells Google Analytics to schedule a command (specified by the function) into the queue to be executed when the library is fully loaded. The function receives `t`, the tracker instance, which contains our `clientId`. The first time the whole snippet loads, `localStorage.getTime('gaClientId')` (specified in our creation parameters) returns `undefined` so a random client ID is generated. When our scheduled function runs, we grab the client ID from the tracker instance and save it to localStorage. The next time the app launches, we already have a client ID saved, which is then read from localStorage and passed to the creation parameters.

Finally, we need to tweak our pageview a little bit:
```diff
-ga('send', 'pageview');
+ga('send', 'pageview' , {'location' : '<YOUR DOMAIN AND PAGE HERE>' });
```

Normally, the pageview event defaults to sending the current document's location. Since we're in a WebView, this address is pretty meaningless and presumably doesn't match up with your google analytics domain settings. The trick here is to manually override the location to make Google Analytics think the user is on your domain. Note that you can get creative here - the pageview event can be moved anywhere, and the location can be constructed in such a way that makes sense for your app.

# Step 3

If your app is put into the background, eventually Google Analytics concludes that the user has left your 'site'. Even if you bring the app in the foreground, no further pageview is generated. My workaround for this was to add the following to my `app.js` file:

```js
document.addEventListener('resume', function(){
  ga('send', 'pageview' , {'location' : '<YOUR DOMAIN AND PAGE>' });
}, false);
```

This is just the standard Cordova listener for when your app is put back in the foreground, where we just generate another pageview as before.

# TL;DR

Add this snippet to your `index.html` file:
```html
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');
  ga('create', 'UA-XXXXX-Y', {'storage':'none', 'clientId': localStorage.getItem('gaClientId')});
  ga('set', 'checkProtocolTask', null);
  ga(function(t){ localStorage.setItem('gaClientId', t.get('clientId')); });
  ga('send', 'pageview' , {'location' : '<YOUR DOMAIN AND PAGE>' });
</script>
```

Add this snippet to your `app.js` file:
```js
document.addEventListener('resume', function(){
  ga('send', 'pageview' , {'location' : '<YOUR DOMAIN AND PAGE>' });
}, false);
```

Happy tracking!