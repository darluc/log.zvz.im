title: Nested AngularJS Form
date: 2015-08-24 21:25:56
tags:
---
## A problem about nested form in AngularJS
Today when I happened to be work with a page having a form nested in another one. I found the nested oneâs ng-controller directive is not working. So I tried to replace the FORM tag with DIV tag, and the ng-controller directvie work again.

The conclusion is that nested FORM is not compiled. I did not find any supported post about this. If you know the reason, please let me know.

[This is an example](http://jsbin.com/yitiku/edit?html,js,output)

**Related book:**  <a href="http://www.amazon.cn/gp/product/B00G3XSBG8/ref=as_li_qf_sp_asin_tl?ie=UTF8&camp=536&creative=3200&creativeASIN=B00G3XSBG8&linkCode=as2&tag=imzvz-23">用AngularJS开发下一代Web应用</a><img src="http://ir-cn.amazon-adsystem.com/e/ir?t=imzvz-23&l=as2&o=28&a=B00G3XSBG8" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />
