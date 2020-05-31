---
title:  "AngularJS: passing a value between pages"
date:   2017-02-26 10:00:00 +0100
categories: frontend angular
excerpt: Learn how to pass values between AngularJS controllers and webpages, and why 2xx status code doesn't always evaluate to success in AngularJS's $http service.
---

I keep on programming the frontend for P1 (which I introduced to you in my [previous post]({% post_url 2017-02-19-angularjs-rest %})), so this text is also dedicated to AngularJS. Today, I'd like to share with you issues that I've encountered trying to pass a value from one distinct webpage to another - and, of course, how I've finally solved them.

We'll start with some Java code for a Spring's `@RestController`. Keep in mind that it's greatly simplified by removing the business logic. It's perfectly OK, as it still gives you an idea of the implemented flow.

{% highlight java %}
@RequestMapping(value = "/generate", method = RequestMethod.POST)
public ResponseEntity<?> generate(@RequestBody GeneratorsListEntryDTO generator) {
        try {
            String generatedHtml = singletonGenerator.generateHtml(generator);

            return new ResponseEntity<>(Collections.singletonMap(SessionConstants.ORIGINAL_HTML, generatedHtml),
                    HttpStatus.OK);
        }
        catch(ClassNotFoundException | InstantiationException ex) {
            String errorMsg = "Fatal error occured during HTML generation: " + ex.getMessage();
            System.out.println(errorMsg);

            return new ResponseEntity<>(errorMsg, HttpStatus.BAD_REQUEST);
        }
    }
{% endhighlight %}

As you can see, the POST method introduced above can either return a simple map containing one key-value pair with generated HTML code (and HTTP 200 "OK" status), or, in case of failure, it'll produce a response, which body contains error message (just a string, not a JSON) and HTTP 400 "Bad Request" status code.

Now, assuming that we're on a choose_generator.html page, where our AngularJS app is provided with `ng-app` directive, what we'd like to do, is to call the `generate` method with a payload based on form contents, receive the returned HTML code, and, on success, redirect to a new page, editor.html. But it's not the whole story: we want another controler, that operates on this new page, to have access to the received HTML code. Long story short, it's a POST with redirect and passing a value.

When it comes to passing a value between pages, there are a plentiful of possibilities here. My goal is to store the generated HTML code on the client-side, as the user will probably often reload it and there's no need to serve the static content from the server and thus increasing the load of transfered data.

I've read some posts on this topic on stackoverflow and my first attempt was to implement a service like this:

{% highlight javascript %}
app.factory("GeneratedHtmlService", function() {
    var generatedHtmlService = {
        original_html : "An error occured upon sending data from server :("
    };

    generatedHtmlService.setOriginalHtml = function(html) {
        generatedHtmlService.original_html = html;
    };

    generatedHtmlService.getOriginalHtml = function() {
        return generatedHtmlService.original_html;
    };

    return generatedHtmlService;
});
{% endhighlight %}

`Factory` component is used to provide reusable, application-wide business logic. In the code snippet presented above, you can see that a JavaScript object `generatedSignatureService` with a property `signature_html` is created and returned. There are also two functions defined: getter and setter for the aformentioned property. It's that simple.

Let's have a look at the controller which will operate on the choose_generator.html page:

{% highlight javascript linenos %}
app.controller('generatorChooser', function($scope, $http, GeneratedHtmlService, $window) {
    $scope.doPost = function() {
        $http.post("/generate", collectPostData())
        .then(function onSuccess(response) {
            GeneratedHtmlService.setOriginalHtml(response.data.original_html);
            $window.location.href = '/editor.html';
        }, function onFailure(response) {
            console.log("Something went wrong - response status is " + response.statusText);
        });
    };
});
{% endhighlight %}

In the first line, `generatorChooser` controller is defined and services are injected, including `GeneratedHtmlService` passed as a parameter. The `doPost` function (which is called on button click with `ng-click`) posts to the `/generate` endpoint with a JSON object, created by `collectPostData` function. Then, `onSuccess` uses our custom service to store received data (the value of `original_html` property, to be precise, as `data` represents the whole JavaScript object). Then, the `window` service is used to redirect to the editor.html page. If the response status is other than 2xx, then the `onFailure` function is fired to print out the error message to a developer's console.

Before I proceed, I'd like to point out, that not only the status code is responsible for choosing which function of the two will be called. It surprised me a lot, but these `@RestController` methods returns evaluates to false:

{% highlight java %}
//respond with HTTP 200 code
return new ResponseEntity<>(HttpStatus.OK);
{% endhighlight %}

{% highlight java %}
//respond with HTTP 200 code and place generated string in body
return new ResponseEntity<>(generatedHtml, HttpStatus.OK);
{% endhighlight %}

Why's that? `$http` success is fired upon receiving a **valid** JSON! So, in order to enter the success function in `$http`, you have to rewrite the above like that:

{% highlight java %}
//adds empty JSON
return new ResponseEntity<>("{}", HttpStatus.OK);
{% endhighlight %}

{% highlight java %}
//serializes map to JSON with one property
return new ResponseEntity<>(Collections.singletonMap(SessionConstants.ORIGINAL_HTML, generatedHtml), HttpStatus.OK);  
{% endhighlight %}

Editor.html has a standard HTML structure with `ng-app` directive in a `body` section, a `div` with `ng-controller`, and another `div`, for which we'd like to set the generated HTML:

{% highlight html %}
<body ng-app="myGenerator">
<div class="container">
    <div id="htmlSignature" ng-controller="htmlManipulator">
        <div ng-bind-html="generatedHtml">

        </div>
    </div>
</div>
<!--
scripts reference here
-->
</body>
{% endhighlight %}

`ng-bind-html`, as the name suggests, allows for binding HTML code to an HTML tag. Tha last thing to do is to define an `htmlManipulator` controller:

{% highlight javascript %}
app.controller("htmlManipulator", function($scope, GeneratedHtmlService, $sce) {
    $scope.generatedHtml = $sce.trustAsHtml(GeneratedHtmlService.getOriginalHtml());
});
{% endhighlight %}

The only thing this controler does is creating a scope variable `generatedHtml`, which is initialized to the value of `original_html` of `generatedHtmlService` object (which, at this moment, should store the value of generated HTML code, received from POST operation). We also use `$sce` service's shortcut method `trustAsHtml` here to resolve the `attempting to use an unsafe value in a safe context` error, which in other case would be fired.

So, we are ready, yes? Certainly we should... But we're not. Generated HTML isn't displayed. Instead, we end up with a text "An error occured upon sending data from server :(". Why's that?

Reloading a page with `$window.location.href` causes the application to be reinitialized again, with all the controllers, services etc. That's why the provided solution works perfectly well within a single page webapp, but not throughout multiple pages. When an `htmlManipulator` controller tries to load the value of `original_html` property with `getOriginalHtml` function, a new instance of `GeneratedHtmlService` is returned, and thus the property's value evaluates to default "An error occured upon sending data from server :(".

To accomplish our task, we have to use WebStorage features of HTML5: `localStorage` or `sessionStorage`. The first one allows for storing information as long as the user does not delete it. `sessionStorage`, on the other hand, stores the information as long as the session lives (mostly that means closing the tab). I've decided to use `sessionStorage`, so I've rewritten the `GeneratedHtmlService` code as following:

{% highlight javascript %}
app.factory("GeneratedHtmlService", function($window) {
    var setOriginalHtml = function(html) {
    $window.sessionStorage.setItem('original_html', html)
    };

    var getOriginalHtml = function() {
        return $window.sessionStorage.getItem('original_html');
    };

    return {
        setOriginalHtml: setOriginalHtml,
        getOriginalHtml: getOriginalHtml
    };
});
{% endhighlight %}

From now on, our solution should work as intended.
