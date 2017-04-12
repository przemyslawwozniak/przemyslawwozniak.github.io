---
title:  "A few tips for jQuery first-timers"
date:   2017-04-11 23:30:00 +0100
categories: programming
excerpt: I share with you what I've learned about jQuery during my HTML editor development.
---

Last time I published a post one month ago. That's because I've been busy developing an HTML editor for P1. 90% of this time was spent with well-known (and somewhat obsolete, I guess) jQuery framework, while the other 10% was AngularJS. That's kind of funny that (mostly) backend developer writes about frontend. But that's how one man army rocks - you're on your own with everything. For those of you who are looking forward to backend, I've got a good news - there's high possibility that the next post will be devoted to beloved ORM solution, Hibernate. Or Spring Data. Or pure database... I still haven't decided which one I'll use.

Let's leave the future alone and talk about jQuery. For those of you who haven't heard the name before, jQuery is a battle-proven JavaScript framework, that makes it more convenient to manipulate DOM elements, which allows for applying dynamic changes to webpage. Of course there's more, e.g. AJAX calls are one of the most extensively used features of jQuery. A good place to start is [jQuery Fundamentals](http://jqfundamentals.com/). The lecture should provide you with basic understanding of what can be achieved with jQuery. As with other frameworks, the rest is in documentation. I personally prefer the unofficial one, [jQAPI](http://jqapi.com/).

Having said that, I could save this text file, commit and push it to my GitHub repo (Jekyll and GitHub Pages rocks, I promise I'll post about them in a future), open a polish craft beer (beer craftsmanship is experiencing its renaissance, which is super cool)... What I have to say is that I won't teach you jQuery - there's far more better resources for that. Instead, I'll share with you things I've found not so obvious. Let's get into it.

### IDs are meant to be unique
Thank you, Captain Obvious! Don't be so harsh on me. You could feel tempted to search an element with certain ID nested within another element, with another ID. Let's take a look at the following HTML code:

{% highlight html %}
<div id="textEditTool">
  <div id="fontSizeTool"></div>
  <div id="fontColorTool"></div>
</div>
{% endhighlight %}

Looks fine, IDs are unique - what can possibly go wrong here? Well, certainly this jQuery statement:

{% highlight javascript %}
$('#textEditTool #fontColorTool');
{% endhighlight %}

That's breaking the rule of IDs uniqueness, even if it's not visible at the first sight. Such a selector suggests that same ID can be used multiple times (e.g. distributed across many divs). So, in case of parent-child relationships, your best bet is to identify a parent with ID and a child with a class:

{% highlight html %}
<div id="textEditTool">
  <div class="font-size-css"></div>
  <div class="font-color-css"></div>
</div>
{% endhighlight %}

This selector will return the unambiguous result:

{% highlight javascript %}
$('#textEditTool .font-color-css');
{% endhighlight %}

### Custom HTML attributes
Imagine that you'd like to somehow mark an HTML element e.g. for further modifications with jQuery. With HTML5(.2) it's now possible thanks to the so called [custom data attributes](http://w3c.github.io/html/single-page.html#embedding-custom-non-visible-data-with-the-data-attributes). These are used just like the "normal" attributes, the only difference is that they are prefixed with `data-`. For example, if you'd like to mark links that should prevent default behaviour on click, you could write:

{% highlight html %}
<a href="https://przemyslawwozniak.github.io/" data-prevent-default="yes" />
{% endhighlight %}

Then, you'd simply check if the `data-prevent-default` attribute is present on the element with jQuery (or it's value, what depends on your specific usage).

### Bind functions to events in JavaScript code
In real-world apps you won't place JS code inside `<script>` tag in your HTML code. You'd create a new JS script file and refer to it via `<script src="path_to_js_script></script>`. To keep "visual" code (HTML) separated from the app's logic (JS) and avoid complications with DOM readiness, it's best to bind functions to events with jQuery. I'll show you what I mean on the example of a button's click. Instead of writing this in your HTML code:

{% highlight html %}
<div id="textEditTool">
  <input class="close-text-edit-tool" type="button" value="Close" click="closeEditTool"/>
</div>
{% endhighlight %}

Write in your JS code:

{% highlight javascript %}
$('#textEditTool .close-text-edit-tool').click(closeEditTool);
{% endhighlight %}

You can also use more generic jQuery's [on method](http://api.jquery.com/on/).

### Harness the power of JavaScript's Map
Maps are one of the most commonly used data structures in web apps (it may be correlated with the popularity of the Model-View-Controller design pattern). [ECMAScript 2015](http://es6-features.org/#MapDataStructure) (or ES6 in short) standard introduced Map implementation. Usage of such objects is rather intuitive:

{% highlight javascript %}
var editHistory = new Map();

function saveHistory(elem) {
  if(editHistory.get(elem.attr('id')) === undefined) {  //if no key is present in the map
    let historyStack = [];  //let is scoped to the nearest enclosing block
    historyStack.push(elem);
    editHistory.set(elem.attr('id'), historyStack); //adds new entry to the map
  }
  else {
    let historyStack = editHistory.get(elem.attr('id'));  //gets the value associated with the given key
    if(historyStack.length > (MAX_HIST_SIZE - 1)) {
      historyStack.shift();   //removes 1st element of array
    }
    historyStack.push(elem);
}
{% endhighlight %}

Code example shows only the basic usage of a map (it'll fill up around 90% of overall time spent with maps, though) - much more detailed information can be found on [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map).

I'll give you one more recipe involving maps - how you can easily iterate over their content with `forEach`:

{% highlight javascript %}
function replaceOldMap(map) {
    Object.keys(map).forEach(function(key) {
        replaceIcons(key, map[key]);  //key, value
    });
}
{% endhighlight %}

### Execute code only when page is ready
To avoid unnecessary "surprises", always put your JavaScript code inside this block of code:

{% highlight javascript %}
$( document ).ready(function() {
  //your code goes here
}
{% endhighlight %}

This rule is especially important if you bind a function to the HTML element, as the target element could not be available (rendered) at the moment, and therefore couldn't be find to do function binding.

### Looking for a decent color-picking solution?
One of the reasons why software evolves so rapidly is that we, programmers, don't reinvent the wheel over and over again, but rather build on top of each other's solutions. I had to provide a color picker for my editor, and... I found it. [Spectrum](https://bgrins.github.io/spectrum/) is open-source, well-documented, involves adding only two files to your project (JS + CSS) and is customizable to the great extent.

I feel kind of yearning for more of these tips, but unfortunately I haven't written them down during the development process, which was a huge mistake, as now I had to rely only on my (not so good) memory. Nevertheless, I hope that you'll find something useful here. Meanwhile, till the next post!
