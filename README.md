# Predictive Optimizing Code Loading

Study of application acceleration by reducing the size of code to be loaded.

1.1 The minimization is based on statistics of what functions are really invoked at runtime,
and loading only them when the application is started next time. (The rest of the functions
remain available and loaded at first invocation.)

1.2 Initially load only the code used _immediately_ at the application startup
(renders initial UI, load initial data). Other code could be loaded afterwards in background,
or upon some events (user navigates to a certain part of the app).

We study this approach in application to javascript in web pages.

# Results

## What can we save?

By opening real world web applications (Google Docs, github, etc) via
proxy, which instruments their javascript code to record function invocations,
we determined that in an average web application only around 20-45 % of the
javascript functions loaded into the browser are actually called at runtime.

<a href="https://www.youtube.com/watch?v=IoFleV1ybxE" target="_blank"><img src="pocl-1-youtube.png"></a>

Tested it also with my web application http://testsheet.biz. This
application will be used later to experiment with actual code removal.
The average JS functions usage ratio was 37%. The main JS file of the
application - testsheet.min.a.js - has usage ratio of 50%. Other files - 
google libraries loaded dynamically have lower usage ration.

## Actually removing the unused code

We have created a prototype which removes the unused code based on the
function call statistics.

<a href="https://www.youtube.com/watch?v=ciKSqUfdC8k" target="_blank"><img src="pocl-2-youtube.png"></a>

It's a javascript-to-javascript compiler, which replaces each unused function
by a stub. The stub, when invoked, first records the fact that it was invoked,
then loads the original function body and executes it. The function calls
statistics is accumulated and uploaded to the POCL server to adjust the code - include only functons seen actually called and remove the rest. Essentially, it implements statistical dead code elimination (1.1).

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

This setup allows to employ POCL even for otherwise static web sites.
(TODO: make it possible to include hash codes into file names,
for more reliable work with HTTP cashes, and deployment of new versions
of myapp.min.js file, new versions of POCL)

It was tested on http://testsheet.biz. The original testsheet.min.js file
is generated by Google Closure Compiler. After invoking every app
feature in UI, the "active" testsheet.min.a.js file weights 65.89%
of testsheet.min.js.

Taking into account that in testsheet.min.js, of all JS functions
only 50% are used, and that in many other applications only
around 35% of functions are used, we can estimate
that for an average application its code size can be reduced to maybe
a half of the original size.

The overhead in file size comparing to the number of invoked functions
(68% / 50%) is caused by the fact we don't just remove the unused functions,
but replace them with a stub expression, able to load the original function and call it.
Also, if we remove closures, when the original closure body is loaded by the stub,
we need to provide it with access to variables captured by the closure from the surrounding scope.
It requires some code to be injected, therefore adds to the overhead. Probably the injected
code can be made more compact, but some overhead will remain anyway, unless POCL is supported by the javascript
engine natively.

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

This approach can save JS programmers from using and developing libraries
which are "only X kilobytes gzipped" - we could use libraries of large size
but the app only loads what is necessary (and when necessary, if we implement
more intelligent statistics handling).

# Further notes

This approach can be applied not only to Javascript but to other languages
and run-time systems. I originally invented it in 2013 when dreaming
about in-browser Common Lisp implementation. Only later I understood it's not necessary to have control on the language implementation to investigate this idea, and decided to try on Javascript. Most of Common Lisp implementation have relatevely large application load time, and I wanted to reduce this time, because in case of web pages, even 2 seconds startup time is too long. I was thinking how to reduce the load time. For example Java loads classes not immediately at startup, but only when the class is first accessed at run-time. But when executed in a web page, making a separate request for every function would not be effective. Java would download the whole .jar file. But I was thinking to use some kind of cache and download only the functions actually used. Another analogy, is startup of native applications by contemporary operating systems. The system frist maps the executable file into virtual memory. And only when code is accessed by CPU, the code page is loaded from disk. The pages which are never accessed are never loaded. If we start the same program again, some pages could still remain cached in memory, so OS does not need to load this code from disk again. The difference from POCL here is granularity: functions vs code pages. A code page could contain together with active code some unused (dead) code too. Also in OS the cache is ephemeral, in-memory. It's not persisted and not shared between distributed systems.

The difference from the approach currently used in JS: instead of the programmer specifying explicitly what code to laod and when, programmer only specifies what code constitutes his "codebase", and runtime system decides itself what code to load and when.

Not only the application code, but also the standard library of the language could be minimized that way.

POCL could be deployed as a tool individually for every web application, as a cloud web acceleration service, or as part of browser implementation. In the last case we could have better minimization if the browser JS engine is adjusted to support POCL natively, thus eliminating the need for some support code injected into the JS sources.

Differentiate between what to download from the internet, and what to load into any particular web page.
For popular libraries used by many web pages (e.g. jquery), large part of the library may be downloaded from Internet to the local cache, and only part of that code is loaded into each particular web page. (This assumes we
don't require all the application code to be combined into a single file).

When deciding what to load, consider not just what web application is it, but what browser is used; maybe differentiate users into classes (example: occasional user who only invokes minor part of fuctionality versus pro users who use more features).

How important is the win we have? If we reduce the javascript size to 50% (1.1), or even implement the more intelligent handling (1.2) by loading only around 20% of JS initially, and preloading in background the scripts probable to be used soon (for example, when initial google sarch page is loaded, we could load in background the scripts necessary for the search results page). Will it improve significantly the performance of particular web applications, user system in general, and Internet as a whole?

Security: what if somebody maliciously submits false usage statistics? In the worst case whole the javascript code will be loaded into browser, as without POCL - not a big problem. And this problem can be solved too. For example, services like CloudFlare protect web apps from malicious activities. Also, if we differentiate users into classes, the malicious statistics will not affect all the users, only those looking similar to the attacker (maybe only other hooligans like him).

Privacy: so, we upload code usage statistics to online storage. Does it violate user's privacy? - No. First of all, web servers see each URL accesses by the user; if they were going to spy, the URLs is more than enough, seeing what JS functions are invoked doesn't change much. And most importantly, the POCL statistics is anonymous. 

It would be nice to find support for several month of work to continue investigating the POCL concept.
