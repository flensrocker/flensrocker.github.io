---
layout: post
title:  "Angular: Signalize async service calls"
date:   2024-09-16 20:00:00 +0200
categories: angular signals
---

{% highlight typescript %}
type ServiceCallFn<TRequest, TResponse> =
  (request: TRequest) => Observable<TResponse>;
{% endhighlight %}
