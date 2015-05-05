# Typesafe ConductR %PLAY_VERSION%

## Introduction

There is nothing particularly special about having your applications and services live within ConductR. In fact, while we have a particular love for Typesafe RP based applications and services, you can run almost anything within ConductR. Furthermore you can run applications and services without any source level change whatsoever.

One change that we do encourage is the move toward your application or service being a reactive one. If you've not read the [Reactive Manifesto](http://www.reactivemanifesto.org/) then please do so now. The manifesto is the DNA of ConductR and so if you want to understand ConductR then it will enlighten you. Your application should be designed with resilience in mind in particular. Because ConductR encourages an application or service to exist across many nodes, and that they can access services of other resources that may come and go, resilience is particularly important.

ConductR assists with locating other services run within the same ConductR cluster, and it does so in a way that imposes no real changes to your resilient applications or services. We provide a library named [`conductr-bundle-lib`](https://github.com/typesafehub/conductr-bundle-lib#typesafe-conductr-bundle-library) that you can use to leverage the environment that ConductR provides. Note that even if you use this library, your application or service will continue to run just fine when used outside of ConductR. 

We think it is important for you to avoid vendor lock-in as much as is reasonable. Furthermore all of the interfaces that we provide we do so in an open and transparent manner so that you can understand what is going on at all times.

In order to quickly get start visit the [quickstart](Dev-Quickstart.html) document.

Happy ConductR-ing!

With regards,
The ConductR Team