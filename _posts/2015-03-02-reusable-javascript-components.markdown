---
layout: post
title: "Composable Components: Comparing Angular and React"
---

Both Angular and React think you should be building client-side apps by encapsulating behavior within the DOM.

Angular does this by teaching the DOM new tricks—you add functionality to existing elements and create new ones entirely by defining directives. 
- Separation of behavior between what's declared in HTML and what's defined in the JS directive.

React eschews this separation in favor of defining your markup structure and new behavior *as* code.

Let's contrast these approaches with an example. Say you're a horrible person running a social network that rips off Twitter. Let's further say that anywhere that there's a list of tweets, you want to wrap it up in a way that puts a big 'ol promoted ad at the top. Because, again, you're an awful person.

Both our examples will create the following markup when run: a pair of tweets from the user with a prepended ad from our coporate sponsors.

{% highlight html %}
{% raw %}
<ul>
    <li>Promoted Tweet</li>
    <li>My Tweet One</li>
    <li>My Tweet Two</li>
</ul>
{% endraw %}
{% endhighlight %}

### Angular

In Angular, you can create directives that wrap arbitrary content using a mechanism called _transclusion_. _"What does this transclude option do, exactly? transclude makes the contents of a directive with this option have access to the scope outside of the directive rather than inside."_


Let's see our promoted-ad-first list in Angular ([JSFiddle](http://jsfiddle.net/92uz99f4/2/)) :

{% highlight html %}
{% raw %}
<div ng-app="transcludeExample">
  <div>
      <ul promoted-list>
          <li>My Tweet One</li>
          <li>My Tweet Two</li>
      </ul>
  </div>
</div>
{% endraw %}
{% endhighlight %}

{% highlight javascript %}
{% raw %}
angular.module('transcludeExample', [])
   .directive('promotedList', function(){
      return {
        restrict: 'A',
        transclude: true,
        template: '<ul>' +
                    '<li>Promoted Content</li>' +
                    '<li ng-transclude></li>' +
                  '</ul>'
      };
  })
{% endraw %}
{% endhighlight %}

### React

React doesn't require a special mechanism to pass dynamic content. Since our behavior and markup structure are all defined in Javascript, our dynamic content is just another argument to a function call.

In React, our example looks like this [JSFiddle](http://jsfiddle.net/2qydq9gs/2/):

{% highlight text %}
{% raw %}
var PromotedList = React.createClass({
    render: function() {
        return <ul>
                  <li>Promoted Tweet</li>
                  {this.props.tweets}
               </ul>
    }
});

var myTweets = [
  <li>My Tweet One</li>, 
  <li>My Tweet Two</li>
]
React.render(<PromotedList tweets={myTweets} />, document.body);

{% endraw %}
{% endhighlight %}

This is using React's JSX pre-compiler, 

_Should I use non-JSX functions for clarity?_

This feels like a fundamental design flaw in Angular- in order to create flexible APIs you have to keep trying to reinvent in HTML what programming languages can already do.

Design patterns - who said something like "formal design patterns make up for language deficiencies"?
