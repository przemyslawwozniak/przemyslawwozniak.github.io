---
title:  "Uploading files the HTML5 way with AngularJS"
date:   2017-03-11 22:00:00 +0100
categories: programming
excerpt: A handy tip for those of you who look into uploading files paneless with the use of latest HTML5's File API.
---

Today I'd like to invite you for a quick practical session with HTML5's File API, which is really handy when it comes to file manipulation. If you've read some of my other posts, you already know, that I don't write typical tutorials here, but I rather concentrate on a specific problems. In this case I was looking for the simplest, most elegant solution for uploading CSV file for further backend processing. As I've decided to add AngularJS to my technology stack for P1 (just an operating name for my side project), the solution also has to be applicable to AngularJS. Without further introduction, let's get into it!

The first question that I've asked myself was: how to provide the user with files browser for picking up a file(s) to upload? You don't have to believe me, just try it yourself - it's as simple as this:

{% highlight html %}
<input type="file" />
{% endhighlight %}

This will create a button for file upload control, with the caption like "Browse" (Firefox) or "Choose File" (Safari). On the click of it, the file explorer window will pop up. If you'd like the user to only upload specific file types, it's a good idea to add additional `accept` attribute for the input, which will filter files on the browsers that support this attribute, resulting in less files to choose from. There's a [plenty of media types](http://www.iana.org/assignments/media-types/media-types.xhtml) to choose from. In case of CSV files, you can use the following:

{% highlight html %}
<input type="file" accept=".csv, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel" />
{% endhighlight %}

This way, only `.csv`, `.xls` and `.xlsx` files will be available for selection. Of course, there is still need for server-side validation, as the `accept` attribute works by examining the file extension. Once again: client-side validation is nice, but server-side is a necessity.

OK, so the user can now pick up a file(s) of his/her liking. What's now? We have to somehow access this file(s). Here comes the File API: selected files are represented with `FileList` - as the name implies, it's a list of files selected by the user, each being a `File` object. In this case, we're only interested in the first one. That's really all we need to know when it goes to the File API itself, but you can find out more on [this Mozilla Developer Network devoted to File API](https://developer.mozilla.org/en-US/docs/Using_files_from_web_applications) - a lot of examples included!

Now, we're ready to implement the client-side process with AngularJS. The most straightforward way would be to simply give an ID to the input and read it contents directly. It is simple, but it's also considered a bad practice. After all, it's not a jQuery. Instead, we'll define our own AngularJS directive - a custom HTML element that allows to pick up the CSV file and, on change, saves it to the object from the parent's scope:

{% highlight javascript linenos %}
app.directive("csvFileLoad", function() {
   	return {
     	  restrict: 'E',
     		replace: true,
     		template: '<input type="file" accept=".csv, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel" />',
     		link: function(scope, elem, attrs) {
     			elem.on('change', function() {
     				scope.$apply(function() {
     					scope.loadedFile.value = elem[0].files[0];
     				});
     			});
     		}
   	}
})
{% endhighlight %}

The `restrict` property from the 3rd line allows to use this directive only as an HTML element, that is in the form of `<csv-file-load></csv-file-load>`. The `replace` is set to `true`, therefore the HTML element will be replaced with the given `template`, that was already presented. The `link` property defines the functionality of the directive. In the above example, it sets the `loadedFile` object's `value` property to the first file selected by the user (line no 9). It is done on element change, as stated in the 7th line of code. It is worth noticing, that the `loadedFile` object that's modified in this function lives in the directive's parent's scope - that is, the controller, that contains it. That's because we haven't specified `scope` property here, which evaluates to default: directive gets its parent's scope. You have to be aware of the consequences that it implicates, ex. your controller has to have `loadedFile` object defined in its scope.

Our file is ready to be send, so now it's time to define a backend controller that will provide a proper endpoint:

{% highlight java linenos %}
@Controller
public class ModelController {
    @PostMapping("/createWithCsv")
    public ResponseEntity<?> createWithCsv(@RequestParam("file") MultipartFile request) {
        try {
            ModelSingleton.createInstance(new BufferedReader(
                                          new InputStreamReader(
                                          new ByteArrayInputStream(request.getBytes()))));

            return ResponseEntity.ok("{}");
        }
        catch(IOException e) {
            String errorMsg = "Fatal error occured during Model creation: " + e.getMessage();
            System.out.println(errorMsg);

            return new ResponseEntity<>(errorMsg, HttpStatus.BAD_REQUEST);
        }
    }
}
{% endhighlight %}

Spring's `@Controller` is used here, as the request is not in the form of beloved REST. Instead, it's a `MultipartFile`. In the above example it's used to instantiate a `BufferedReader` from the file's bytes stream. Uploading files requires registering a `MultipartConfigElement` class (which is the same as `<multipart-config>` in XML configuration). If you're using Spring Boot (like me), than you're already setup, as it's done for you. If you're not... Start doing so! That's one of the most impressive projects of Spring Framework. It is also a good idea to define maximum size of uploaded files. That's just one of the properties of `MultipartConfigElement`. To do so, simply add this line to `application.properties`:

{% highlight java %}
spring.http.multipart.max-file-size=32KB
{% endhighlight %}

I've also used the shortcut `@PostMapping` annotation that's pretty handy. There isn't much to describe here - the intresting part is how we can serve it with AngularJS. Or maybe it's just a Java developer's perspective... You can always ask me additional questions in the comments section :)

We'll start with defining a new AngularJS controller that wraps our custom directive and an additional button used to send selected file to the backend:

{% highlight html %}
<div ng-controller="dataLoader">
  <csv-file-load></csv-file-load>
  <br/>
  <button ng-click="send()">Send data</button>
</div>
{% endhighlight %}

And here comes the controller:

{% highlight javascript linenos %}
app.controller("dataLoader", function($scope, $http, $window) {
    $scope.loadedFile = {value : ''};

    $scope.send = function() {
        var formData = new FormData();
        formData.append('file', $scope.loadedFile.value);

        $http({
            url: "/createWithCsv",
            method: "POST",
            data: formData,
            headers: {"Content-Type": undefined}
        })
        .then(function onSuccess(response) {
            $window.location.href = '/choose_generator.html';
        }, function onFailure(response) {
            console.log("Something went wrong - response status is " + response.statusText);
        });
    }
});
{% endhighlight %}

In the 2nd line we define a `loadedFile` object within a controller's scope. The `send` function, that's called on button click, creates a new `FormData` and appends a key `file` with the value of selected file. Then, it's just an ordinary HTTP call, but with the `Content-Type` set to `undefined` - the browser will set this accordingly on its own (in my case it is set to `text/csv`).

That's all. For those of you who prefer to stick to the REST, you can create a JSON object corresponding to the Java's DTO, and set one of its properties with the contents of the file by using AngularJS's `FileReader`:

{% highlight javascript linenos %}
app.controller("dataLoader", function($scope, $http, $window) {
    $scope.loadedFile = {value : ''};

    $scope.send = function() {
        var file = $scope.loadedFile.value;
            reader = new FileReader();

        reader.readAsText(file, 'UTF-8');

        reader.onloadend = function(e) {
          var data = JSON.stringify({data:e.target.result});

          $http.post("/createWithCsv", data)
          .then(function onSuccess(response) {
              $window.location.href = '/choose_generator.html';
          }, function onFailure(response) {
              console.log("Something went wrong - response status is " + response.statusText);
          });
        }
      }
});
{% endhighlight %}

We use `FileReader`'s `readAsText` function to read the content of the file (another function that may be interesting to you is `readAsArrayBuffer`). `onloadend` function is fired when the job is done. Inside this function a JSON object is created with only one property, namely `data`, which value equals to the content of the file. Then, the data is POSTed to specified URL.
