# Predictive Optimizing Code Loading

Study of application acceleration by reducing the size of code to be loaded.

1.1 The minimization is based on statistics of what functions are really invoked at runtime,
and loading only them when the application is started next time. The rest of the functions
are available and loaded at first invocation.

1.2 Also, initially load only the code used _immediately_ at the application startup
(renders initial UI, load initial data). Other code could be loaded afterwards in background,
or upon some events (user navigates to a certain part of the app).

We study this approach in application to javascript in web applications.

# Results

## What can we save?

By opening real world web applications (Google Docs, github, etc) via
proxy, which instruments their javascript code to record function invocations,
we determined that in average web application only around 25-30 % of the
javascript functions loaded into the browser are actually called at runtime.


## Actually removing the unused code

We have created a prototype which removes the unused code based on the
function call statistics.

It's a javascript-to-javascript compiler, which replaces each unused function
by a stub. The stub, when invoked, first records the fact that it was invoked,
then loads the original function body and executes it. The function calls
statistics is accumulated and uploaded to the POCL server to adjust the code.

The prototype assumes all the app javascript is combined into a single file.
Based on call statistics this codebase is separated into
two files: "active" and the "rest":

```
             call statistics
                    |
                    V
                 ┌------┐
myapp.min.js ->  | POCL | -> myapp.min.a.js
                 |      | -> myapp.min.r.js
                 └------┘

```

The web page includes the "active" script:
```html
<script src="myapp.min.a.js"></script>
```

Initially, when there is no call statistics, the "active" file includes all the functions.
As enough statistics is accumulated, the compilation is repeated
and new .a and .r files are generated. Now the .a file only contains
the functions seen invoked. Should any other function be called by the application,
its stub loads the .r file.

This setup allows to employ POCL along otherwise static web sites.
(TODO: make it possible to include hash codes into file names,
for more reliable work with HTTP cashes, and deployment of new versions
of myapp.min.js file, new versions of POCL)

It was tested on http://testsheet.biz. The original testsheet.min.js file
is generated by Google Closure Compiler. After invoking every app
feature in UI the "active" testsheet.min.a.js file weights 68%
of testsheet.min.js.

This is a very early version of the prototype and can be improved further.
For example, here is how a stub expression looks: `_$p.s(1010,_e$)`.
Clearly, the code elements injected by POCL can be made more compact.
Some overhead will remain anyway, but I estimate another 10% of code
can be removed, making the testsheet.min.a.js around 58% of the original
JS size.

Taking into account that at testsheet.biz, of all JS functions
around 50% are used, and that in many other applications only
around 20-30% of functions are used, we can estimate
that for an average application its code side can be reduced to maybe
40% of the original size.

# Advantages

- Comparing to dead code elimination based on static analysis,
  POCL results in better minimization even in the simplest implementation  (1.1),
  which loads ALL the code seen to be called at least once.
  Because there are code paths possible in theory, therefore not "dead"
  from the static analysis point of view, but which are never called,
  or rarely called by the application in reality.

- Any javascript code is eligible for this load optimization,
  without the need for any "export" annotations and other restrictions
  (unlike Google Closure Compiler)

This approach will save JS programmers from using and developing libraries
which are "only X kilobytes gzipped" - we could use libraries of large size
but the app only loads what is necessary (and when necessary, if we implement
more intelligent statistics handling).

# Further notes

This approach can be applied not only to Javascript but to other languages
and run-time systems. I originally invented it in 2013 when dreaming
about in-browser Common Lisp implementation.

## Security

## Privacy

## What to download from the internet, and what to load into any particular web page.
