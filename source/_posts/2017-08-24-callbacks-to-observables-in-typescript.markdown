---
published: true
layout: post
title: "Callbacks to Observables in TypeScript"
date: 2017-08-24 22:24:08 +0000
comments: true
categories: Programming, TypeScript, Functional, Observable 
---

> ####TLDR; 
> Functional programming techniques makes it possible to reduce the amount of boiler plate code required when working with JavaScript libraries that require traditional callback functions. This blog will show how to drastically reduce the code and how to turn the callback handlers into observables.


### Back story

I was working on hooking the [AWS Cognito JS SDK][cognito-sdk] into my Angular 4 project. I was using [SytemJS][systemjs] and [RollupJS][rollup] and after struggling for some time to get the bundling to work properly, I gave up and decided to just use the plain JavaScript API.

This meant that not only was I missing the nice TypeScript typings for the SDK, but also faced with the clunky callback patterns used in some JavaScript API's.

### JavaScript Callbacks

There are various common JavaScript patterns for handling async methods via callbacks. These callback mechanisms allow callers to provide a hook so that they can be called back in the future with the result or error. 

One common approach is that the method requires a single callback function in addition to its normal parameters. The called function will then invoke the callback function with an error as the first parameter, or a result as the second parameter. It is up to the callback function to determine whether it received an error or not.

The following contrived example shows how the `sendMsg` function will invoke the callback and pass an error message or success result.
``` javascript
function sendMsg(message, callback) {
    if (message === 'dontsend') {
        callback('whoops not sending!');
    }
    else {
        callback(null, 'Success!');
    }
}

sendMsg(message, function(error, result) {
    if (error) {
        console.error("An error occurred" + error);
    }
    else {
        console.log("Message was sent successfully." + result);
    }
})
```

As mentioned before there are other common ways to do callbacks such as requiring separate success and error handling functions, or using objects with `onSuccess` and `onFailure` methods. However, the solution is similar regardless of the callback pattern followed so the rest of this article will use the single callback function approach above as the example.

### TypeScript, Observables and callbacks

Calling such a JavaScript function from TypeScript looks something like this:

``` ts
public sendMessage(message: string, cb: (err:any, res:any) => void) {
    
    sendJS.sendMsg(message, cb);
}
```

This is all fine and works ok but I don't want the clients of my code to have to pass a callback handler. What I really want is to return an [Observable][observable] to my clients. I can achieve this as follows:

``` ts
public sendMessage(message: string): Observable<string> {
    
    let sendResult = new Subject<string>();   
    sendJS.sendMsg(message, 
            (err:any, res:any) => {
                if (!!err) {
                    sendResult.error(err);
                }
                else {
                    sendResult.next("Message was sent successfully." + res);
                }    
            }
    );
    
    return sendResult.asObservable();
}
```

However the bit on `line 6` where we check for the error is a bit tedious. I will have to duplicate this all over the place and I don't necessarily know what to do with the error. In most cases I will just send it along for the client to deal with.

To get rid of the error handling duplication I write a function to take the subject, success and *optional* error handler functions and return a callback function

``` ts
interface CallbackFunction {
    (error: any, result: any): void;
}

 function subToHandler<T>(subject: Subject<T>, successFn: (m:any) => T, errorFn?: (a:any) => any): CallbackFunction {
    return (error:any, result:any) => {
        if (!!error) {
            if (!!errorFn) {
                subject.error(errorFn(error));
            }
            else {
                subject.error(error);
            }
        }
        else {
            subject.next(successFn(result));
        }
    };
 }
```
The interface `CallbackFunction` cleans the method signature up a bit and makes things a bit easier to read.

The returned callback function checks the error and, if an error handler was provided, applies it before calling `error` on the subject `(line 5)`. If no error is provided to the callback function then it will call `next` on the subject to pass the valid result on to the observers.

This means that I can now rewrite the send message code as follows:
``` ts
public sendMessage(message: string): Observable<string> {
    
    let sendResult = new Subject<string>();   
    sendJS.sendMsg(message, 
        subToHandler(sendResult,
            res => {
                return "Message was sent successfully." + res
            }
        )
    );
    
    return sendResult.asObservable();
}
```

This is much better since I don't have to call `next` and `error` on the subject and I don't have to check for the error anymore. There is also much less code because I no longer have to explicitly handle the error. It can be fully delegated to the observers of the subject.

However, this still not perfect. I will be duplicating the `Subject` and `Observable` creation lines everywhere. It also looks a bit weird to be creating a Subject and passing it on to another function. 

I can get rid of the statements on `lines 3 and 12` above by writing another function which will return the Observable, relieving me from creating the subject explicitly.

Consider the following snippet:
``` ts
function exec<T>(fn: (cb:CallbackFunction)=>void, successFn: (a:any)=>T, errorFn?: (e:any)=>any): Observable<T> {
    
    let result = new Subject<T>();
    let obs = result.asObservable();

    setTimeout(() => fn(subToHandler(subject, successFn, errorFn)), 0);
    return obs;
}
```

Here I add another function `exec` which will create a subject, turn the subject into a handler using `subToHandler` defined earlier, apply the function provided as first parameter and then return an observable.

Note that I create the observable on `line 4` before applying the function `fn`. This is important since subject will only notify its subscribers of changes after the time they subscribed. `Line 6` wraps the call to the function `fn` in a timeout to handle the case where `fn` doesn't actually call the callback function asynchronously.

You'll also notice that the first parameter to exec, `fn`, is a function which takes a `CallbackFunction` as its sole parameter. The JavaScript library functions require parameters in addition to the callback funtion, so I cannot pass the JavaScript function in to `exec`. 

Consider the following rewrite of the `sendMessage` function:

``` ts
public sendMessage(message: string): Observable<string> {        
    return exec(
        (cb: CallbackFunction) => {
            sendJS.sendMsg(message, cb);
        },
        res => {
            return "Message was sent successfully." + res;
        }
    );
}
```

Here I effectively partially applied the original `sendJS.sendMsg` function by closing over the value of the first argument, `message`, and returning a new function which takes *only* a callback function as parameter. This allows the `exec` function to create the appropriate callback function, using `subToHandler` and pass it to the partially applied `sendMsg`.


### In closing

By using some basic functional programming techniques I could reduce the amount of boiler plate code required when working with JavaScript libraries with traditional callback functions. I managed to reduce the code drastically and turned the callback handler into an Observable which clients of my service can subscribe to.


[cognito-sdk]: https://github.com/aws/amazon-cognito-identity-js
[systemjs]: https://github.com/systemjs/systemjs
[rollup]: https://rollupjs.org/
[observable]: http://reactivex.io/documentation/observable.html