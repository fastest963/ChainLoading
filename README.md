#chainLoading#
Allows you to chain together deferreds or functions and control the order at which they call your callbacks.
The benefit being that all methods are called immediately and the callbacks are called in the order you want.
This can make loading a page that requires multiple API calls much faster since the API calls will happen in parallel.

The order is controlled by "levels". You can can have many deferreds or callbacks on each "level".
Each "level" is guaranteed to be called sequentially, but deferreds in the same "level" are called upon completion.
If a deferred is rejected, none of the callbacks in levels after it in the chain will be called unless you used one of the *ignore* methods.

--------------------------

## new ChainLoading() ##
Returns a new chainLoading object.

## Methods to add new deferreds to the chain ##

### push(deferred1, ..., deferredn) ###
Allows you to add deferreds to the next "level". All the deferreds passed as arguments will be added to the same level.

**For the deferred's done/fail/always methods, it's imperative that you use the chain's bind() method below for your callbacks!**
```JS
chain.push(myDfd.done(chain.bind(myCallback, this)));
```

### add(deferred1, ..., deferredn) ###
Allows you to add deferreds to the current "level". Remember the order is not guaranteed within a level.
This is useful when you have a group of deferreds that are independent of each other but all depend on an earlier "level".

**For the deferred's done/fail/always methods, it's imperative that you use the chain's bind() method below for your callbacks!**

### pushIgnoreFail(deferred1, ..., deferredn) ###
Same as `push()` except that any failures will be ignored and the chain will continue processing upcoming levels.

### addIgnoreFail(deferred1, ..., deferredn) ###
Same as `add()` except that any failures will be ignored and the chain will continue processing upcoming levels.


## Methods to send to deferreds ##

### bind(func, context, [arg1, ..., argn]) ###
Required method for adding callbacks to the deferreds passed to push/add. Even if you have nothing to bind to, you must use this method to add callbacks.

### storeArgs([arg1, ..., argn]) ###
If you want to store the resulting arguments on the chain for later use by `applyArgs` then send this function to a deferred's done.

`storeArgs` does **NOT** support currying (use `after` instead), so `dfd.done(chain.storeArgs(1))` will ONLY store `1` regardless of what is resolved.

*Keep in mind that if you're using `pushIgnoreFail` args will not be stored on fail unless you do something like `fail(chain.storeArgs(null))`.*
```JS
chain.push(myDfd.done(chain.storeArgs));
chain.after(chain.storeArgs("finished myDfd"));
chain.push(mySecondDfd.done(chain.applyArgs(myCallback, this));
//{myCallback} will get args from myDfd, then "finished myDfd", then args from mySecondDfd
```

### applyArgs(func, context, [arg1, ..., argn]) ###
Applies arguments stored using `storeArgs` to func. Can be used instead of `bind`.


## Methods to add callbacks to the chain ##

### after(func, [currentLevel = false]) ###
Adds a function to the next (default) or current level. Useful at the end of all your deferreds for knowing when everything completed.

### done(func, [currentLevel = false]) ###
Alias for `after()`

### fail(func) ###
Func is called whenever ANY of the non-ignored deferreds on ANY level fail. Useful for knowing if any part of the required chain failed.

### fork() ###
Returns a NEW chain that's dependent on the current level of this chain but will then run independently of this chain.
You can bring the 2 back together using push/add later if you choose.


--------------------------

## Example ##

```JS
var chain = new ChainLoading();
chain.fail(function() {
    //render failure page
});
chain.push(loadUserProfileInfo());
chain.push(loadUserMusic());
//render the template once we know we have the profile and music
chain.push(loadTemplate().done(chain.bind(renderTemplate, this)));
```
--------------------------

## Caveats ##

* Works with jQuery 1.11.0 and 2.1.0 but is not guaranteed to work with ALL future versions. This relies on the deferred calling done/fail with the context of the deferred itself.
* If you add the same deferred object to the chain multiple times the done/fail callbacks will be called when the FIRST occurence of the deferred resolves/rejects in the chain. See the tests.js file for an example.
