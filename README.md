# chainLoading #
Allows you to chain together deferreds or functions and control the order at which they call your callbacks.
The benefit being that all methods are called immediately and the callbacks are called in the order you want.
This can make loading a page that requires multiple API calls much faster since the API calls will happen in parallel.

Let's get straight into an example for rendering a user's profile page:
```JS
var chain = new ChainLoading();
//first thing we need is to fetch the template
chain.push(this.fetchTemplate('myPage.ejs')).done(_.bind(this.renderTemplate, this));
//we want to fetch the user's profile from the backend immediately, but delay calling
//renderUserProfile until the template was rendered
chain.push(this.loadUserProfile()).done(_.bind(this.renderUserProfile, this));
//fetch the comments immediately and render them on the same "level" as the user profile
//also we don't need to fail the WHOLE page if it fails to fetch comments, so ignoreFail
chain.addIgnoreFail(this.loadComments()).done(_.bind(this.renderComments, this));
chain.fail(_.bind(this.renderPageFailed, this));
```

The order is controlled by squential "levels". A "level" will not get resolved until the previous "level" is resolved.
If a deferred is rejected, none of the callbacks in levels after it in the chain will be called unless you used one of the
*ignore* methods.

**If you're upgrading from 0.2.x, please see the "upgrade notes" section below**

--------------------------

## Simple Usage ##
```JS
var chain = new ChainLoading();
//myCallback will get called first
chain.push($.get('http://example.com/1')).done(myCallback);
//myCallback2 will get called second
chain.push($.get('http://example.com/2')).done(myCallback2);
```

### push(deferred) ###
Adds the deferred to a new "level". This method returns a deferred.

### add(deferred) ###
Adds the deferred to the current "level". This method returns a deferred.
Order is **not** guaranteed within a "level". If you require order, then always use `push`.
This is useful when you have a group of deferreds that are independent of each other but all depend on an earlier "level".

### pushIgnoreFail(deferred) ###
Same as `push()` except that any failures will not cause the chain to stop.

### addIgnoreFail(deferred) ###
Same as `add()` except that any failures will not cause the chain to stop.

### after(func) ###
Adds a function to be called once the previous "levels" complete. Useful at the end of all your deferreds for knowing when
everything completed.

### fail(func) ###
Func is called whenever previous "levels" failed. Useful at the end of all your deferreds for knowing if the chain failed.
 
## Upgrade notes ##

If you're coming from 0.2.x release then you'll notice a few things changed:
- Usage of chain.bind is now deprecated
- `fail` will only be called when previous "levels" failed instead of every fail callback always being called.

## Advanced usage ##

### push(deferred, ..., deferredn) ###
push/add/etc all support multiple deferreds. The done callback will get called once ALL the deferreds have resolved and the
arguments will be all of the deferreds results combined.
```JS
chain.push(d1, d2, d3).done(function(one, two, three) {});
d1.resolve(1);
d2.resolve(2);
d3.resolve(3);
```

### storeArgs(arg1, ..., argn) ###
If you want to store the resulting arguments on the chain for later use by `applyArgs` then send this function to a deferred's done.

`storeArgs` does **NOT** support currying, so `dfd.done(chain.storeArgs(1))` will ONLY store `1` regardless of what is resolved.
Instead you can do `dfd.done(chain.storeArgs(1)).done(chain.storeArgs)`

*Keep in mind that if you're using `pushIgnoreFail` args will not be stored on fail unless you do something like
`fail(chain.storeArgs(null))`. Failure arguments can be unexpected so a decision was made to not add them by default.*
```JS
chain.push(myDfd).done(chain.storeArgs));
chain.after(chain).storeArgs("finished myDfd"));
chain.push(mySecondDfd).done(chain.applyArgs(myCallback, this));
//{myCallback} will get args from myDfd, then "finished myDfd", then args from mySecondDfd
```

### applyArgs(func, context, [arg1, ..., argn]) ###
Applies arguments stored using `storeArgs` to func. Can be used instead of a `bind` function. This does support currying so feel
free to use this on your last deferred.

### fork() ###
Returns a NEW chain that's dependent on the current level of this chain but will then run independently of this chain.
You can bring the 2 back together using push/add later if you choose.
