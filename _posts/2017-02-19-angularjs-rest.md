---
title:  "Client-side rendering with AngularJS - consuming RESTful webservice"
date:   2017-02-19 21:50:00 +0100
categories: programming
excerpt: Case study of consuming a RESTful webservice with AngularJS to render a complex, dynamic User Interface.
---

One can say many things about corporations, but not that they are agile. That's why the most innovative products are invented by start-ups. Then they are acquired, so they can use a bottomless pit of corporate money to introduce their product to the market and reach consumers.

This conservatism is also reflected in the (programming) technologies used by corporations. A Java developer who works on behalf of a massive corporation, like me, is considered lucky if he/she has an opportunity to code in the latest Java version. In this regard, I'm a lucky programmer, but I have to still work with JSP for rendering webpages... Not even Thymeleaf!

But hey! If you really like programming, then I'm sure you can find few spare hours a week to play with code on your own. So do I. Currently, I'm working on a webapp, which - I hope so! - will solve one of the problems that people who have to be recognizable in Internet communications have. Let's simply call it "project one" or, in shortcut, P1. In the following posts, I'll share with you some of the issues I've encountered by the time I work on P1 and found interesting or that could be helpful to you - exactly what I've promised you when launching this blog. So, please remember, that my posts aren't standard tutorials. I wish to do so, but unfortunately I don't have that much free time: full-time job, side-projects, spending time with family and friends, and posting (in foreign language) fill the week quite tight. To conclude, think of this blog more like issue-solution repo.

Quite an introduction, but this is a first technical post in a series, so I thought that it's the best place to inform you what to expect. Why I've written about JSP and side-project? For the starters, P1's technology stack was Java 8 and Spring 4. Now, that the architecture and core functionality is coded (don't worry, I'll go back to this stage's issues in the future posts), it's time to decide, what technology will be used to render webpages. I've never used AngularJS before, but heared a lot of nice things about it. Considering the service is RESTful, it's the perfect opportunity to learn something new and of increasing popularity. That being said, I invite you to read my recipe for consuming RESTful webservice with AngularJS - from the (mostly) backend Java developer's perspective. Let's do it!

I've started with a Spring's @RestController with the following method:

{% highlight java %}
@RequestMapping(value = "/generators", method = RequestMethod.GET)
public List<GeneratorsListEntryDTO> getGeneratorsList() {
    return generatorsList;
}
{% endhighlight %}

As you can see, it's a basic getter, that simply returns the list of a `GeneratorsListEntryDTO` objects. `GeneratorsListEntryDTO` class is a POJO that consists of two private fields, including a list of POJOs of type `ConstructorParamDTO`. When run in [Postman](https://www.getpostman.com/), the `getGeneratorsList()` method will respond with the following body:

{% highlight javascript %}
[
  {
    "generatorName": "Generator no 1",
    "constructorParams": [
      {
        "type": "Boolean",
        "value": "true",
        "displayText": "Description no 1"
      },
      {
        "type": "Integer",
        "value": "2",
        "displayText": "Description no 2"
      },
      {
        "type": "Integer",
        "value": "2",
        "displayText": "Description no 3"
      }
    ]
  },
  {
    "generatorName": "Generator no 2",
    "constructorParams": [
      {
        "type": "Integer",
        "value": "10",
        "displayText": "Description no 1"
      },
      {
        "type": "Integer",
        "value": "10",
        "displayText": "Description no 2"
      }
    ]
  }
]
{% endhighlight %}

My goal is to deliver UI that will allow user to choose a generator (represented by `GeneratorsListEntryDTO` object) from a drop-down list and, upon a selection, display each of the generator's constructor's parameters in a format: `displayText` : input. Input should be specific to a type of constructor parameter, eg. `Boolean` should be represented with an HTML selector with options: true and false, `Integer` should be a numeric input (introduced in HTML5) which accepts only positive numbers etc. What's more, the default value should be provided from the `value` property of `ConstructorParamDTO` object.

Before I'll proceed to the AngularJS implementation of the features I've described above, I'd like to clarify that to "enable" AngularJS in your webapp, you could directly include the JavaScript library from the Internet (with `<script>` HTML tag) or use a tool which handles web resources for you. The second option is prefered for the production-grade applications, so if you're thinking about going live with your webapp, consider it. I use wro4j myself. To get a hint how to deal with it on a basic level, read "Front End Assets" section from [this Spring guide](https://spring.io/guides/tutorials/spring-security-and-angular-js/).

We'll start by adding new HTML page to our static resources - one, which will display the described UI. Take a look at the `<body>` section of it:

{% highlight html linenos %}
<body ng-app="signaturesGenerator">
<div class="container">
    <div ng-controller="generatorChooser">
        <h3>Which generator do you wish to use?</h3>
        <div>
            <select ng-model="selectedGenerator" ng-options="g.generatorName for g in generators"></select>
        </div>
        <div>
            <p>Provide additional parameters for {{selectedGenerator.generatorName}} or use the default ones: </p>
            <p ng-repeat="param in selectedGenerator.constructorParams">
                {{param.displayText}} : <constructor-param param="param"></constructor-param>
            </p>
        </div>
    </div>
</div>
<script src="js/angular-bootstrap.js" type="text/javascript"></script>
<script src="js/signaturesGenerator.js"></script>
</body>
{% endhighlight %}

If you asked me, one of the greatest features of AngularJS is how flawlessly it integrates into ordinary HTML code. Obviously, it's possible thanks to many directives. First one is `ng-app` which defines an AngularJS application. In this case, the directive is placed inside a `<body>` tag, which means, that it is the "owner" of the app, mainly in terms of scope. In the 3rd line of code you can see another directive: `ng-controller` defines the controller, an object, which manipulates model's data, but can also perform many other operations, e.g. HTTP operations.

In the 6th line you can see how to easily create a drop-down list of options based on JavaScript object. `ng-model` binds the value of HTML controls (in this example it's a `<select>`) to application data. Thanks to that, you can manipulate such variable in an AngularJS app (i.e. in controller). `ng-options` creates a drop-down list based on object - JSON representation of `GeneratorsListEntryDTO` provided by our @RestController. We want the options to be equal to `generatorName`, that's why we use `ng-options="g.generatorName for g in generators"`. Once again, notice how nicely it blends into the HTML code.

Selector is done, so it's high time for displaying the constructor's parameters. In 9th line you can see an AngularJS expression written between a pair of double curly braces. Expressions can be used to bind AngularJS data to HTML - in this case, we extract the value of `generatorName` field out of the generator selected by the user, represented by `selectedGenerator`. This way we will display a new HTML paragraph `<p>` with an info to the user.

As you can see in the JSON snippet, a list of constructor's params are sent. We want to iterate over each of them and for each one render `displayText` associated with it and the specific input. For iterating we use another AngularJS directive, namely `ng-repeat`. Line number 10 tells that we want to iterate over `constructorParams` list of selected generator and refer to the current one with the variable named `param` - but of course it could be anything else, it's just for your use. In 11th line, the expression is used to get the aformentioned `displayText` property. There is also a weird HTML element... In fact, it's a custom AngularJS directive. A Java developer can think of it as a method, that will provide some kind of output based on a given input. I'll soon explain it to you in details.

But before it, let me show the heart of AngularJS application. It's postfix is quite verbose - don't tell me you thought it wouldn't involve some JavaScript. Here's the code snippet:

{% highlight javascript linenos %}
var app = angular.module('signaturesGenerator', []);

app.controller('generatorChooser', function($scope, $http) {
    $http.get('/generators/')
    .then(function onSuccess(response) {
        $scope.generators = response.data;
    }, function onFailure(response) {
        console.log("Something went wrong - response status is " + response.statusText);
    });
});

app.directive("constructorParam", function() {
    return {
        scope : {
            param : '='
        },
        templateUrl : '/html_templates/constructor_param_template.html'
    };
});
{% endhighlight %}

In the first line, an AngularJS app is being created - we refer to the name that we've given to it in the HTML code, presented previously. Then, we define a controller `generatorChooser` - once again, it was referenced in the HTML. The controller's single responsibility is to make an HTTP GET request to our @RestController endpoint. I've used a [shortcut method](https://docs.angularjs.org/api/ng/service/$http#get) here, which takes an URL (`/generators` in this case) and specifies what should be done in case of success (first function after `then`) - that's to save the response data (our JSON) to this application's scope `generators` variable - and what should be done in case of failure (logging to console). This should be pretty obvious.

Line number 12 is a custom directive definition. I have to admit, that this is the part, which forced me to [ask for a help](http://stackoverflow.com/questions/42328023/angularjs-dynamic-custom-directive-with-multiple-templates/42328338) of developers who are more experienced in AngularJS. My custom directive defines only two things. One is it's scope: in this case it allows for two-way object binding. This allows to "inject" an object to the custom directive to work with. It's a necessity as we want to have a different HTML output for a different types of input parameters. That's what's dealt with inside a template - here we only specify template's location.

We're ready to have a look at the final code snippet of a template:

{% highlight html linenos %}
<div ng-switch="param.type">
    <div ng-switch-when="Integer">
        <input type="number" name={{param.type}} min="0" value={{param.value}}>
    </div>

    <div ng-switch-when="Boolean">
        <select ng-if="param.value" name={{param.type}}>
            <option selected="selected" value="true">Yes</option>
            <option value="false">No</option>
        </select>

        <select ng-if="!param.value" name={{param.type}}>
            <option value="true">Yes</option>
            <option selected="selected" value="false">No</option>
        </select>
    </div>

    <div ng-switch-when="String">
        <input type="text" name={{param.type}} value={{param.value}}>
    </div>

    <div ng-switch-when="Double">
        <input type="number" name={{param.type}} step="0.01" min="0" value={{param.value}}>
    </div>
</div>
{% endhighlight %}

I use `ng-switch` and `ng-switch-when` directives, which behaves in the same way as good old Java's switch-case. The choice of a specific HTML code is done by evaluating parameter's `type` property. If it's equal to `Integer` for example, I crete an HTML5's input of type number, and set it's attributes `name` and `value` to the desired values of parameter object. On the other hand, if it's a `Boolean`, the template will generate HTML code for selector: if it's default value is true (what is checked with `ng-if` directive), then "Yes" is provided as default. Simple, nice and clean, isn't it? :)

That's it, now you're set up. I hope that this read was helpful to you. If so, post a comment below!
