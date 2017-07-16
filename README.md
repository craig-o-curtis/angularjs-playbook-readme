Notes from AngularJS Playbook course

#### Note
Code is as-is from the tutorial. One could always refactor into the latest and greatest, but there is too much good material out there by excellent authors, like Scott Allen, the author of this Pluralsight course. 

These notes are findings I find interesting or enlightening at this moment of my coding career. I hope others may find them useful as well.


## Contents

[1.1 The Revealing Module Pattern and an Auth Token Service](#11-the-revealing-module-pattern-and-an-auth-token-service)

[1.2 Angular HTTP Interceptors](#12-angular-http-interceptors)

[1.3 Protecting from XSS and using Sanitization and $sce](#13-protecting-from-xss-and-using-sanitization)


## 1.1 The Revealing Module Pattern and an Auth Token Service


This is a nice way to encapsulate logic in a service. 
Just return an object with the methods as keys.
No need to expose internal business logic.

Example: localStorage.service.js
```javascript
(function(module) {

    var localStorage = function($window) {

        var store = $window.localStorage;

        var add = function (key, value) {
            value = angular.toJson(value);
            store.setItem(key, value);
        };

        var get = function(key) {
            var value = store.getItem(key);
            if (value) {
                value = angular.fromJson(value);
            }
            return value;
        };

        var remove = function(key) {
            store.removeItem(key);
        };
        // here just return the methods
        return {
            add: add,
            get: get,
            remove: remove
        };
    };
    // and put class-like object above into factory def
    module.factory("localStorage", localStorage);

}(angular.module("common")));
```

And a form encoder service

example: formEncoder.service.js
```javascript
(function(module) {

    var formEncode = function() {
        return function(data) {
            var pairs = [];
            for (var name in data) {
                pairs.push(encodeURIComponent(name) + '=' + encodeURIComponent(data[name]));
            }
            return pairs.join('&').replace(/%20/g, '+');
        };
    };

    module.factory("formEncode", formEncode);

}(angular.module("common")));
```

and this service can be used in ANOTHER service

Example: oauth.service.js
```javascript 
(function (module) {

    var oauth = function ($http, formEncode, currentUser) {

        var login = function (username, password) {

            var config = {
                headers: {
                    "Content-Type": "application/x-www-form-urlencoded"
                }
            }
            // call that above formEncode serfice
            var data = formEncode({
                username: username,
                password: password,
                grant_type: "password"
            });
            // object returns a promise
            return $http.post("/login", data, config)
                        .then(function (response) {
                            currentUser.setProfile(username, response.data.access_token);
                            return username;
                        });

        };
        // and returns above class-like modularized object
        return {
            login: login
        };
    };

    module.factory("oauth", oauth);

}(angular.module("common")));
```

Moving along, if the above code gets a successful response with a token, we can store that.
Example: currentUser.js
```javascript
(function(module) {
   
    var currentUser = function(localStorage){

        var USERKEY = "utoken";

        var setProfile = function (username, token) {
            profile.username = username;
            profile.token = token;
            localStorage.add(USERKEY, profile);
        };

        var initialize = function () {
            var user = {
                username: "",
                token: "",
                get loggedIn() { // notice get keyword
                    return this.token;
                }
            };

            var localUser = localStorage.get(USERKEY);
            if (localUser) {
                user.username = localUser.username;
                user.token = localUser.token;
            }
            return user;
        };

        var profile = initialize();
        
        return {
            setProfile: setProfile,
            profile: profile
        };
    };

    module.factory("currentUser", currentUser);

}(angular.module("common")));
```

[Back to Contents](#contents)


## 1.2 Angular HTTP Interceptors

Interceptors invoked for:
* request
* requestError
* response
* responseError

General Use:

In this example, an interceptor is used to add a token to an HTTP header.

Example: addToken.js
```javascript
(function (module) {

    var addToken = function (currentUser, $q) {

        var request = function (config) {
            if (currentUser.profile.loggedIn) {
                config.headers.Authorization = "Bearer " + currentUser.profile.token;
            }
            return $q.when(config); // $q.when() always resolves, wraps in a promise
        };

        return {
            request: request
        }
    };

    module.factory("addToken", addToken);
    // use this factory inside the config, push to interceptors array
    module.config(function ($httpProvider) {
        $httpProvider.interceptors.push("addToken");
    });

}(angular.module("common")));
```


[Back to Contents](#contents)

## 1.3 Protecting from XSS and using Sanitization and $sce

HTML tags are not trusted by default to protect against XSS Cross Site Script attacks.
Note - not 100% secure - apps still can be exploited in other ways.

### Without ng-sanitize
Using Interpolation {{ }}:
```js
    vm.test = "French <em>Test</em";
```
```html
    <p>{{ vm.test }}</p>
```
```
    Output: 
    French <em>Test</em>
```

Using ng-bind:
```html
    <p ng-bind="vm.test"></p>
```
```
    Output is same: 
    French <em>Test</em>
```

Using ng-bind-html: 
```html
    <p ng-bind="vm.test">{{ test }}</p>
```
```
    Output is blank: 
```
* Console error shows trying to use ng-bind-html in an unsafe context.

### Implicit approach With ng-sanitize:
* Good to use with trusted users.
* Not for all users, since susceptible to XSS.

* Include angular-sanitize script:
```html
<script src="[path to]/angular-sanitize.js"></script>
```
```js
    angular.module('app',['ngSanitize'])
```

Result:
```js
    vm.test = "French <em>Test</em";
```
```html
    <p>{{ vm.test }}</p>
```


Output: 
French _Test_  

* Sanitize will also remove any event handlers or script tags


### Explicit Approach with Trusting HTML - $sce
* Can use a function getter for this
```html
    <p ng-bind-html="vm.getTrustedParagraph()"></p>
```
```js
    vm.getTrustedParagraph = function() {
        return $sce.trustAsHtml(vm.test);
    }
```



[Back to Contents](#contents)
