---
layout: post
title: 'Using JVM Clojure for Google Cloud Functions'
excerpt_separator: <!--more-->
published: false
---

I recently ran into a requirement where a Google Cloud Function (GCF) seemed a good fit.
I wanted to use Clojure for this, at least as a first cut, but the examples I found
were centered around ClojureScript/NodeJS, which wound up being a non-starter
for various boring reasons. Googling about turned up essentially nothing about
how to use JVM Clojure for this purpose. The GCF story for Java is about as
easy as it can get, just implement a known interface, uberjar,
and deploy with the `gcloud` tool. 

My first attempt in Clojure used `gen-class`, but deployment resulted in this
exception:

```
"Exception in thread "main" java.lang.ExceptionInInitializerError
	at clojure.lang.Namespace.<init>(Namespace.java:34)
	at clojure.lang.Namespace.findOrCreate(Namespace.java:176)
	at clojure.lang.Var.internPrivate(Var.java:156)
	at ck.proxysql_notifier.<clinit>(Unknown Source)
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.base/java.lang.reflect.Constructor.newInstance(Constructor.java:490)
	at com.google.cloud.functions.invoker.NewHttpFunctionExecutor.forClass(NewHttpFunctionExecutor.java:51)
	at com.google.cloud.functions.invoker.runner.Invoker.startServer(Invoker.java:243)
	at com.google.cloud.functions.invoker.runner.Invoker.main(Invoker.java:129)
Caused by: java.io.FileNotFoundException: Could not locate clojure/core__init.class, clojure/core.clj or clojure/core.cljc on classpath.
	at clojure.lang.RT.load(RT.java:462)
	at clojure.lang.RT.load(RT.java:424)
	at clojure.lang.RT.<clinit>(RT.java:338)
	... 11 more" 
```

First step was to verify that `clojure/core__int.class` made it in the jar. It did.
Next guess was some kind of class loader issue. I've actually run into this scenario
several
times, where some service requires a Java class implementation, said service uses its
own class loader, and when the class is loaded you run afoul of the way Clojure uses
class loaders to support its dynamic features. It can be confusing, because it's clear
that the Clojure classes are available in the uberjar, but you get an error that
they're not found. Given that I've hit this problem
more than once, it's surprising how little is written about it.
[This post](https://groups.google.com/g/clojure/c/Aa04E9aJRog/m/f0CXZCN1z0AJ) reminded me
of how I had solved this problem in the past, using `setContextClassLoader`. And 
[this Slack thread](https://clojurians-log.clojureverse.org/clojure/2020-02-03)
further reminded me that my past solutions used a simple "shim" class, because
the class loader needs to be set before the Clojure runtime is loaded. Since I
hit this problem infrequently enough to forget the solution between instances,
I thought I would document it here.

This solution is specific to implementing a GCF in Clojure, but the general pattern
holds in other cases, which is to implement whatever interfaces the service or
framework requires as a Java class, which in turn  calls into Clojure code 
implementing the actual logic. Here's the Clojure:

```clojure
(ns sparkofreason.cloud-function
  (:require [cheshire.core :as json]
  (:import (com.google.cloud.functions HttpRequest HttpResponse))
  
(defn service
  [^HttpRequest request ^HttpResponse response]
  (let [body (json/parse-stream (.getReader request))
        response-writer (.getWriter response)]
    (println "Received " body)
    (.write response-writer "ok")))
```

Pretty boring stuff, deserialize the request body JSON, print it out (which
will appear in the Stackdriver logs), and return "ok" for the response. The
Java wrapper looks like this:

```java
package com.sparkofreason.cloud_function;

import com.google.cloud.functions.HttpRequest;
import com.google.cloud.functions.HttpResponse;
import com.google.cloud.functions.HttpFunction;
import clojure.java.api.Clojure;
import clojure.lang.IFn;

public class MyCloudFn implements HttpFunction {

  static {
    Thread.currentThread().setContextClassLoader(MyCloudFn.class.getClassLoader());
    IFn require = Clojure.var("clojure.core", "require");
    require.invoke(Clojure.read("sparkofreason.cloud-function");
  }
  private static final service_impl = Clojure.var("sparkofreason.cloud-function", "service");
  
  @Override
  public void service(HttpRequest request, HttpResonse response)
    throws Exception {
    service_impl.invoke(request, response);
  }
}
```

For the interop, I'm just following [Alex Miller's example of calling Clojure from Java](https://github.com/puredanger/clojure-from-java).
The key part is the call to `setContextClassLoader`. We put this in the static
constructor *before* any Clojure stuff happens, to ensure the class loader is
set so that our service or framework will have visibility to the classes which
get loaded when you start poking at `clojure.java.api.Clojure`.

You'll then need to build and deploy this to your service/framework, which for GCF
looks like this:

1. Call `java -c` on the java file, output the class to some folder like "classes".
2. Build an uberjar using some Clojure tool (I used [depstar](https://github.com/seancorfield/depstar))
including the "classes" (or whatever) directory in the classpath (e.g. `:extra-paths ["classes"]` 
in a `:build` alias if using `deps.edn`).
3. In your CLI, go to the folder containing the uberjar.
4. Use the [`gcloud functions deploy` command](https://cloud.google.com/functions/docs/deploying/filesystem#deploy_using_the_gcloud_tool)
to deploy your cloud function.
5. Test with `curl` or whatever, `POST`ing some JSON for the body. You should see `ok` returned as the result, and whatever
you sent for the body printed in the Stackdriver logs.

The `Could not locate clojure/core__init.class` error can be mysterious in this type of interop
scenario, so hopefully if you run into that case, this post will save you some time. Note that I just through
this together quickly from memory, at some point I'll make a little example repo and update this
post. If you try the code before and find errors before I get that done, please DM @sparkofreason on
the clojurians slack.
