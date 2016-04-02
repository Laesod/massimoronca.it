---
title: Meteor and Kendo UI integration guide
date: "2015-06-05"
description: "Wanna know what people are talking about in your city?<br>Build a reactive application with Meteor and Kendo UI and find out!"
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/120684737224/integrate-meteor-with-kendo-ui
image: /assets/images/meteor-kendo.png
tags:
- Telerik
- Meteor
- Kendo UI
- Telerik Kendo UI
- Meteor and Kendo UI
- Meteor and Kendo UI integration
- Meteor tutorial
---

At the end of last year I've been involved in a testdrive project aimed to explore the possibility of replacing good old desktop applications with modern web applications, a demanding task, but nonetheless a fun one.   

The client gave me complete freedom to chose whatever technology I wanted, except for some key elements, one of them was [Kendo UI][1], a widget library they had been testing for a while with an impressive [list of features and components][2].   

On my side I've decided to put [Meteor][3] on test, I confess, I'm not the biggest Javascript fan around, but I've rarely felt so confident with a new platform like I've been with Meteor, so I went for it.   
The biggest advantage Meteor gave us was speed, we've been able to put together an incredible (IMHO) amount of work in a very short period of time.   

Long story short: the project was a success, everything was good, we also had the opportunity to experiment on the infrastructure side, and I've learned a lot in those few months, about things I could never have accessed otherwise.  

There was only one small glitch that constantly bugged me: the integration between Meteor and Kendo, especially in regards to reactivity.  

Meteor might not be the *new kid on the block* anymore, but it's not the [well established framework][4] either: there was no official Telerik package for Meteor, and I had to roll my own, learning by doing.  

I don't know wheter Telerik listened to my prayers or not, but on february of this year they released a bunch of [packages on Atmoshpere](https://atmospherejs.com/telerik), making things easier for everyone, including me, so I decided to write this simple guide.   

The Telerik packages differ from each other only in the default theme used, the set of bundled components is the same, you can find the complete list on [their github project page][5], although they included only the open source version of the components, this guide applies to the paid version as well.   

We are going to build a simple applications that shows a list of tweets for a particular location (in this example the city of Milan).   
Let's start by creating a new empty application and configure the packages 

``` bash

$ meteor create telerik-demo
$ cd telerik-demo

# disable autopublish, we don't want to publish all the tweets, but only the
# 30 most recent
# disable insecure too, but it's not required
$ meteor remove autopublish insecure

# add official momentjs
$ meteor add momentjs:moment
# add twitter library
$ meteor add mrt:twit  
# add telerik components 
# choose your favourite theme
# if you install the bootstrap theme, bootstrap CSS framework is included too
$ meteor add telerik:kendo-ui-core-material-theme

```


Open `telerik-dom.js`, the core application code, and add the declaration of our collection of tweets, making it available to both client and server.

```javascript
Tweets = new Mongo.Collection('tweets');
```

then, look for the line `if (Meteor.isServer) {`, this is the entry point of the server side app.  
What we need to do is setup the Twitter library, subscribe on a stream filtering by location and language, and then add every tweet we receive to a collection in our MongoDB instance.   
You might be tempted to do something like this

```javascript
Meteor.startup(function () {
  var t = new TwitMaker({
    consumer_key:     '...'
    , consumer_secret:    '...'
    , access_token:     '...'
    , access_token_secret:  '...'
  })

  // I used this tool to get the bounding box coordinates
  // http://boundingbox.klokantech.com/
  var milan = ['8.9936308861','45.3026233328','9.5197601318','45.6359571671']
  // subscribe to tweets from Milan in Italian
  var stream = t.stream('statuses/filter', {locations: milan, language: 'it'})

  // listen and wait
  stream.on('tweet', function (tweet) {
    Tweets.insert(tweet);
  );
});
```

Aaaand you would be wrong…   
If you do this, Meteor will exit with the error `Meteor code must always run within a Fiber`.  
The problem is that the code that inserts the new tweets is executed asynchronously, outside the Meteor Fiber, so you need to provide Meteor the right environment in which to run. The esiest way is to wrap the callback inside a `Meteor.bindEnvironment` call, like this

```javascript
  stream.on('tweet', Meteor.bindEnvironment(function (tweet) {
    Tweets.insert(tweet);
  }, function () {
    console.log('Failed to bind environment');
  }));
```

Don't forget to publish your tweets, to be correct, the 30 most recent tweets.

```javascript
Meteor.publish('latest_tweets', function() {
  return Tweets.find({}, {sort: {timestamp_ms: -1}, limit: 30});
})
```

Now that we have a working Twitter scraper, let's work on the frontend:
open `telerik-demo.html` and replace its content with this

```html

<head>
  <title>twitter</title>
</head>

<body>
  {{ > tweets }}
</body>


<template name="tweets">
  <div id="tweets"></div>
  <div id="pager"></div>

  <script type="text/x-kendo-tmpl" id="tweet">
    <div class="tweet" data-twitter-id="${id}">
      <ul>
        <li>
          <div class="user">
            <a href="http://twitter.com/${user.screen_name}" aria-label="${user.name} (screen name: ${user.screen_name})" data-scribe="element:user_link">
              <img alt="" src="${user.profile_image_url}" data-scribe="element:avatar">
              <span>
                <span data-scribe="element:name">${user.name}</span>
              </span>
              <span data-scribe="element:screen_name">@${user.screen_name}</span>
            </a>
          </div>
          <p class="tweet">${text}</p>
          <p class="timePosted">Posted <a title="#= moment(created_at).format('dddd, MMMM Do YYYY, h:mm:ss a') #">#= moment(created_at).fromNow() #</a></p>
          <p class="interact">${place.full_name}</p>
        </li>
      </ul>
    </div>
  </script>


</template>
```

The code inside the `<script type="text/x-kendo-tmpl">` is the template code that the Kendo ListView is going to use to render every single item in the list.   
An explanation of the custom syntax used in Kendo templates is beyond the purpose of this guide, but you can refer to the [Telerik official documentation](http://docs.telerik.com/kendo-ui/framework/templates/overview), it's really easy.

Now back to `telerik-demo.js`, inside `if (Meteor.isClient) { ... }` we subscribe to the publications created above

```javascript
  Meteor.subscribe('latest_tweets', function() {
    console.log("subscribed to latest tweets");
  })
```

Now taht the channel is open, the server will start to send the data to the client and keep it in sync, automatically.   
The last step is to bootstrap the Kendo component and display our tweets on the page

```javascript
// when the tweets template has done rendering
Template.tweets.rendered = function() {

  // create the datasource for the listview showing 3 items per page
  var dataSource = new kendo.data.DataSource({
    pageSize: 3
  });

  // initialize the listview with the datasource and the template code
  $('#tweets').kendoListView({
    dataSource: dataSource,
    template: kendo.template($("#tweet").html())
  });

  // initialize the pager component
  $('#pager').kendoPager({
    dataSource: dataSource
  });

  // this function is run automatically when dependencies change,
  // in our case when the collection is updated 
  // (items are changed, added or removed)
  // Meteor figured out that if we subscribed to a publication, we wanna
  // be also informed when it changes
  this.autorun(function() {
    // we use fetch becase it returns a Json list of all the items 
    // in the collection, which is one of the formats supported by
    // Kendo datasource 
    // this also triggers the rendering of the list view and the pager
    dataSource.data(Tweets.find({}, {sort: {timestamp_ms: -1}}).fetch()); 
  });
};
```

And that could be all. If now you launch `meteor` inside the project folder and navigate to `http://localhost:3000` you should see a list of tweets automatically updating.   
It's not the best looking app ever, but it works.   
{{% figure src="http://i.imgur.com/lXxC3Gg.png" title="" %}}

Let's do something to improve the presentation layer.   
I'm not a great designer, so I grabbed this [Twitter widget customization][6] from Codepen and copied the relevant bits inside `telerik-demo.css`.   
Next thing I wanted to try is to add a fade-in effect to every new tweets, to make things smoother.   
I've created a class that runs an animation inside the CSS and I wanted to attach this class to every new tweet `div`.   

```css
@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.fade-in {
  animation-duration: 1s;
  animation-name: fade-in;
}

/* I removed the prefixed versions of keyframes and animation-* for clarity */
```

My first naive approach relied on [Mutation Observers][7].  

```javascript
  document.addEventListener("DOMNodeInserted", function(event) {
    if($(event.target).hasClass("tweet")) {
      // a new tweet has been added to the DOM, start fading
      event.target.classList.add('fade-in');
      console.warn("Another node has been inserted! ", event, event.target);
    }
  }, false);
```

Only after a few unsuccesfull attempts, where every tweet was being faded, I realized that when you change the `data` property of the `datasource` component, the current view (so only the current page) is re-rendered completely, making every tweet a *new* tweet to the eyes of the poor browser.   
The solution is simple, I made a diff between the actual data and the new data coming from the server and only added the fade to those tweets that really were new, I used the Twitter `id`   attribute as the key.   

update the template with the new Twitter id attribute

```html
<!-- telerik-demo.html -->
<script type="text/x-kendo-tmpl" id="tweet">
    <div class="tweet" data-twitter-id="${id}">

```

update the autorun function to compute a diff and apply the fade-in effect 
only to the new tweets

```javascript
this.autorun(function() {
  var current_values = dataSource.data();
  var updated_values = Tweets.find({}, {sort: {timestamp_ms: -1}}).fetch();
  var current_ids = _.map(current_values, function(x) { return x.id; });
  var updated_ids = _.map(updated_values, function(x) { return x.id; });
  var diff = _.difference(updated_ids, current_ids);

  // update dataSource and trigger listview render
  dataSource.data(updated_values);
  // fade in new nodes
  _.each(diff, function(x) {
    $('div[data-twitter-id="'  + x + '"').addClass('fade-in');
  })
});
```

not strictly required, but advised, remove the fade class when animation ends

```javascript
var animationListener = function(event){
  if (event.animationName == "fade-in") {
    console.log("remove class fade-in from", event.target);
    event.target.classList.remove('fade-in');
  }
}
document.addEventListener("animationend", animationListener, false); 
```

much better now
{{% figure src="http://i.imgur.com/gr9F232.png" title="" %}}

You can see the application in action [here](http://telerik.meteor.com) and find the source code on [Mikamai's Github](https://github.com/mikamai/telerik-meteor-integration-demo)


[1]: http://www.telerik.com/kendo-ui "Kendo UI"
[2]: http://www.telerik.com/kendo-ui#more-widgets "Kend UI widgets"
[3]: http://meteor.com "Meteor"
[4]: http://en.wikipedia.org/wiki/Spring_Framework "Spring MVC"
[5]: https://github.com/telerik/kendo-ui-core "Kendo UI Core"
[6]: http://codepen.io/nerijusgood/pen/Ggqygo "Twitter custom widget CSS"
[7]: http://updates.html5rocks.com/2012/02/Detect-DOM-changes-with-Mutation-Observers "Mutation Observers"