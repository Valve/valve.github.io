---
layout: post
title: "jQuery inputmask plugin + AngularJS"
date: 2013-08-01 13:09
comments: true
categories: [jquery, angularjs, javascript]
---

The plugin I was going to use was [jquery inputmask](https://github.com/RobinHerbots/jquery.inputmask) by Robin Herbots.

I was working on a small [angularjs application](http://mobeedeals.ru) and needed a text input masking to force users to enter their telephone numbers in a certain format. "So, what's the problem?" - I thought.

<!--more-->

I added the neccessary code:

{% codeblock lang:html %}

<form ng-submit="saveApplication()">
  <input type="text" ng-model="application.phone" id="phone"/>
  ...
</form>
{% endcodeblock %}

{% codeblock lang:javascript %}
// ....
// phone mask for Russia 
$('#phone').inputmask('+7(999)999-99-99');

// My controller
...
$scope.saveApplication = function(){
  phone = application.phone;
  ... saving phone here
}
{% endcodeblock %}

... only to find out that `$scope.application` was `undefined` and no phone value never made it to controller.

I decided to use any angular-specific masking library and [found one](https://github.com/angular-ui/ui-utils#mask), but it was still buggy and couldn't handle my simple `+7(999)999-99-99` mask.

Exasperated, I tried several input masking plugins for jQuery, but none of them worked with angular.

Then I stumbled upon [this](http://stackoverflow.com/questions/14994391) and [this](http://stackoverflow.com/questions/16935095) and understood:

#### One must wrap a jQuery plugin into a directive in order for the plugin to work.
Easy-peasy, let's do it.

Having prepared myself with [the documentation on the subject](http://docs.angularjs.org/guide/directive), I wrote version #1:


{% codeblock lang:javascript %}
... assuming I have a module ngApp

ngApp.directive('inputMask', function(){
  return {
    restrict: 'A',
    link: function(scope, el, attrs){
      $(el).inputmask(scope.$eval(attrs.inputMask));
    }
  };
});
{% endcodeblock %}

And the html:

{% codeblock lang:html %}
<input input-mask="{mask: '+7(999)999-99-99'}" ng-model="application.phone"/>
{% endcodeblock %}

Refreshing the browser, I saw the working mask, but the `$scope.application` was still `undefined` after filling in the phone, i.e mask was correctly initialized but the value never propagated to the controller. So I added some jQuery into the directive:
{% codeblock lang:javascript %}
... assuming I have a module ngApp

ngApp.directive('inputMask', function(){
  return {
    restrict: 'A',
    link: function(scope, el, attrs){
      $(el).inputmask(scope.$eval(attrs.inputMask));
      $(el).on('change', function(e){
        scope.application == scope.application || {}
        scope.application.phone = $(e.target).val();
      });
    }
  };
});
{% endcodeblock %}

How do I get rid of hardcoded bindings? We can access the value of `ng-model` attribute and set 
the appropriate scope value:

{% codeblock lang:javascript %}
... assuming I have a module ngApp

ngApp.directive('inputMask', function(){
  return {
    restrict: 'A',
    link: function(scope, el, attrs){
      $(el).inputmask(scope.$eval(attrs.inputMask));
      $(el).on('change', function(){
        scope.$eval(attrs.ngModel + "='" + el.val() + "'");
        // or scope[attrs.ngModel] = el.val() if your expression doesn't contain dot.
      });
    }
  };
});
{% endcodeblock %}

I achieved what I wanted, but I'm still not quite satisfied because of the nudging feeling that this code could still be made more idiomatic and robust.




