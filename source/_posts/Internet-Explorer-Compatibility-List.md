title: Frameworks and Internet Explorer Compatibility
date: 2015-09-16 23:06:50
tags:
---
# Internet Explorer Compatibility

- Bootstrap ( latest v3.3.5 )
  
  Internet Explorer 8 and 9 are also supported, however, please be aware that some CSS3 properties and HTML5 elements are not fully supported by these browsers. In addition, **Internet Explorer 8 requires the use of Respond.js to enable media query support.**
  
  | Feature         | Internet Explorer 8 | Internet Explorer 9          | 
  | --------------- | ------------------- | ---------------------------- | 
  | `border-radius` | Not supported       | Supported                    | 
  | `box-shadow`    | Not supported       | Supported                    | 
  | `transform`     | Not supported       | Supported, with `-ms` prefix | 
  | `transition`    | Not supported       |                              | 
  | `placeholder`   | Not supported       |                              | 
  
  [reference](http://getbootstrap.com/getting-started/#support)
  
- jQuery ( latest v1.11.3 or v2.1.4 )
  
  |            | Internet Explorer | Chrome                   | Firefox                  | Safari | Opera                           | iOS  | Android   | 
  | ---------- | ----------------- | ------------------------ | ------------------------ | ------ | ------------------------------- | ---- | --------- | 
  | jQuery 1.x | 6+                | (Current - 1) or Current | (Current - 1) or Current | 5.1+   | 12.1x, (Current - 1) or Current | 6.1+ | 2.3, 4.0+ | 
  | jQuery 2.x | 9+                |                          |                          |        |                                 |      |           | 
  
  [reference](https://jquery.com/browser-support/)
<!--more-->  

- AngularJS ( latest v1.4.5 & v1.2.28 )
  
  **Note:** AngularJS 1.3 has dropped support for IE8. Read more about it on [our blog](http://blog.angularjs.org/2013/12/angularjs-13-new-release-approaches.html). AngularJS 1.2 will continue to support IE8, but the core team does not plan to spend time addressing issues specific to IE8 or earlier.
  
  [reference](https://docs.angularjs.org/guide/ie)
  
  Angular 1.3 only supports jQuery 2.1 or above. jQuery 1.7 and newer might work correctly with Angular but we don't guarantee that.
  
  [reference](https://docs.angularjs.org/misc/faq)
  
  since jQuery is known for breaking changes in minor versions, we tend to stick to a minor version of jQuery unless we are releasing a new minor version of Angular:
  
  - 1.2.x supports jQuery 1.10.x
  - 1.3.0 will switch to supporting jQuery 2.x
  
  [reference](https://github.com/angular/angular.js/issues/3585)
  
- Angular UI Bootstrap ( latest v0.13.4 )
  
  This repository contains a set of **native AngularJS directives** based on Bootstrap's markup and CSS. As a result no dependency on jQuery or Bootstrap's JavaScript is required. The**only required dependencies** are:
  
  - [AngularJS](http://angularjs.org/) (requires AngularJS 1.3.x, tested with 1.4.5). 0.12.0 is the last version of this library that supports AngularJS 1.2.x.
  - [Bootstrap CSS](http://getbootstrap.com/) (tested with version 3.1.1). This version of the library (0.13.4) works only with Bootstrap CSS in version 3.x. 0.8.0 is the last version of this library that supports Bootstrap CSS in version 2.3.x.
  
  [reference](https://angular-ui.github.io/bootstrap/)
  
  ​
