---
title: Lazy Loading Ajax in ASP.NET MVC
date: 2010-07-22 16:30 UTC
tags: .NET, ASP.NET, ASP.NET MVC, C#, jQuery
---

I recently changed jobs and now work for an e-commerce company called [vente-exclusive.com](https://www.vente-exclusive.com/). We typically open new sales around 9am in the morning and immediately get a MASSIVE amount of traffic from that moment.

Of course the architecture behind our ASP.NET MVC frontend is made to handle that and we manage to keep a pretty good response time using a mix of caching techniques, content delivery networks and clever programming.

I was tasked with implementing a new "users who bought this also bought..." kind of feature you may see on most e-commerce sites. The processing time and data to load on the page are not negligible and that feature being displayed at the very bottom of the page, we might end up using a lot of additional resources for nothing if the user does not scroll down.

It was decided that we would display these frequently combined items only if the area where it's supposed to load comes into the user's viewport - that is if it's displayed on the user's screen whether he scrolled down or his screen is big enough to handle the whole page. Amazon uses the technique on its [homepage](https://www.amazon.co.uk/) (scroll down to the bottom).

I found a pretty neat jQuery plugin to handle just that, it can be found at:
[http://remysharp.com/2009/01/26/element-in-view-event-plugin](http://remysharp.com/2009/01/26/element-in-view-event-plugin)

The `frequently-ordered` block has a `display:none` property. While remaining invisible it can be attached to an event called `inview` that will be triggered only once on the page (hence the ".one").

```js
$(document).ready(function () {
  $('#frequently-ordered')
  .one('inview', function (event, visible) {
      if (visible) {
          loadFrequentlyOrderedProducts(1);
      }
  });
});

```

The next step is standard MVC Ajax. The `loadFrequentlyOrderedProducts` function calls the Controller Action and returns the data from the partial:

```js

function loadFrequentlyOrderedProducts(page) {
  $('#frequently-ordered').show();
  var url = '/Product/LoadFrequentlyOrderedProducts';

  $.get(url, { productId: 324, saleId: 234 },
    function (data) { 
      $('#frequently-ordered').html(data); 
    }
  );
};

```


The Action would be fairly straightforward:

```csharp

public ActionResult LoadFrequentlyOrderedProducts(int? productId, int?saleId)
{
    FrequentlyOrderedProductsModel model = 
        new FrequentlyOrderedProductsModel();
    
    //code omitted for illustration purposes
    return PartialView("FrequentlyOrderedProducts", model);
}
```

That's an interesting technique for high traffic sites often dealing with a trade-off between high availability and rich dynamic features.