title: Nested AngularJS Form
date: 2015-08-24 21:25:56
tags:
---
## A problem about nested form in AngularJS
Today when I happened to be work with a page having a form nested in another one. I found the nested oneâs ng-controller directive is not working. So I tried to replace the FORM tag with DIV tag, and the ng-controller directvie work again.

The conclusion is that nested FORM is not compiled. I did not find any supported post about this. If you know the reason, please let me know.

[This is an example](http://jsbin.com/yitiku/edit?html,js,output)
