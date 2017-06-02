---
layout: post
title: Building minification-safe Angular.js applications
date: 1970-01-16 21:09:03.000000000 -07:00
---
The first time I pulled up my text editor and starting working with Angular.js minifcation wasn't on my mind. Playing with Angular's cool features took precidence, and dependency injection was a very new concept to me. Of course, as my application grew and I began to think about compiling and compressing my javascript, I was presented with a fun plethora of javascript errors and missing providers. What happened? It all comes down to dependency injection and the way Angular handles internal dependencies.

You see, when you create a controller, directive, modules, service, etc. (basically anything where you can inject dependencies) Angular runs the parameters of your function through its registered components and looks for matches.

<small>Controller creation in Angular.js</small>

	var myController = function($scope, $http) {
		//We included $scope and the $http service into our controller.
	}
    
    
In this example, Angular automatically found $scope and $http and included them as parameters into your controller. Using custom services is very similar:

	//Somewhere else:
	myModule = angular.module('myModule'), [ //Module dependencies ];

	//My service creation
	myModule
  	.service('myService', [function() {
    	return {
      	saySomething : function() { console.log("Something"); }
    	}
  	}])

	var myController = function($scope, $http, myService) {
  	//We included $scope, $http, and myService into the controller.

  	myService.saySomething() //"Something"
	}
    
Did you catch the special sauce in that service declaration? If not, don't worry, we'll get there ;).

This is how most of us learn to use Angular, and it works great until we try to run it through uglify or minify to compress the code. If you know how minifcation works, it should be pretty obvious what happens. The first example above may compile to look a little like this (I've expanded the code back to readable JS to make it readable):

<small>Minification output of our controller above</small>

	var myController = function(a, b) {
  	//This will throw an error at runtime becasue 'a' is not a 	valid dependency.
	}

Turns out this breaks everything. The solution for controllers is different than that of modules. For controllers, there is a simple way to make your code minification-safe:

<small>Minification-safe controller declaration</small>

	var myController = function(anyVariableName, $http) {
    /*
     * We've included $scope and the $http service
     * into our controller. Since we set up 
     * the $inject attribute on our controller,
     * it doesn't matter what variable names
     * we use.
     */
	}

	myController.$inject = ['$scope', '$http']
    
Even when you minify this and 'anyVariableName' changes to 'a' this code will work exactly the way it was intended to.
Remember when I mentioned the special sauce in the service declaration above? As it turns out, most new Angular users will define their modules, services, directives, and factories just like the documentation tells them:

	angular.module('greetMod', []).
 
  	factory('alert', function($window) {
    	return function(text) {
      	$window.alert(text);
    	}
  	}).
 
  	value('salutation', 'Hello').
 
  	factory('greet', function(alert, salutation) {
    	return function(name) {
      	alert(salutation + ' ' + name + '!');
    	}
  	});

But wait... won't we have the same problems we had with our first controller? How will the factory 'alert' know that $window is $window when its minified to 'a'? How will 'greet' know to access 'alert'? Well, they won't, and it will break. This is where the docs can send you in the wrong direction if you don't read them carefully. The correcy way to initialize this factory and retain the ability to minify the javascript is a small but powerful change:

	angular.module('greetMod', []).
 
  	factory('alert', ['$window', function($window) {
    	return function(text) {
      	$window.alert(text);
    	}
  	}]).
 
  	value('salutation', 'Hello').
 
  	factory('greet', ['alert', 'salutation', 	function(alert, salutation) {
    	return function(name) {
      	alert(salutation + ' ' + name + '!');
    	}
  	}]);
    
Try and spot the difference. We've initialized our factory function() inside an array preceded by each dependency as string literals in quotes. Its a simple change that will save you time and effort.

Below are examples for declaring any type of dependency-injected part of Angular.

	//Controllers
	var myController = function(anyVariableName, $http) {
  
	}

	myController.$inject = ['$scope', '$http']

	//Modules
	var myModule = angular.module('myModule', [ 	
    	//Dependencies go here as an array of strings.
        ])

	//Services
	myModule.service('serviceName', ['$http', '$location', function($http, $location) {
  

	}])

	//Directives
	myModule.directive('myDirective', ['$http', 	'$location', function($http, $location) {
  

	}])

	//Factories
	myModule.factory('factoryName', ['$http', 	'$location', function($http, $location) {
  

	}])

	//Of course those last 3 are very similar. Its as if they did that on purpose :)
    
There you have it. Use the format above and you'll never have to worry about minifying your wonderful Angular.js code again.

Thanks for reading. Comment or share if you found this helpful.
