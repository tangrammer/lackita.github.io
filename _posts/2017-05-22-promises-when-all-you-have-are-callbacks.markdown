---
layout: post
title: "Promises when all you have are callbacks"
date: 2017-05-22 12:00:00
categories: [gcloud, javascript, js, promises]
comments: true
---

Google cloud functions terminate shortly after they finish, so any
callbacks and unresolved promises at worst never happen and at best
occur intermittently. Their workaround is, if a promise is the return
value, they wait to terminate until it's resolved.

This works great when the API you're working with actually uses
promises, but a lot of third party libraries haven't caught up with
this new feature. Fortunately, Javascript is pretty good at closures,
so we can use one to wrap a callback in a Promise:

``` javascript
function awesome_cloud_function(request, response) {
	return new Promise((resolve, reject) => {
		ThirdPartyAPI.function_with_callback(() => { resolve() });
	});
}
```

This creates a promise that only resolves once the callback
finishes. Obviously you should make sure anything that needs to be
done in the callback happens before resolving.
