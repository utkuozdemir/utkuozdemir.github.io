---
title: "Why you should care about Kubernetes as a backend developer?"
categories:
  - blog
tags:
  - technical
  - kubernetes
  - devops
  - backend
  - spring
---

## Introduction

I started my career as a Java backend developer. Until the last few years my focus was mostly on
developing backend applications. About 2 years ago I shifted my focus to cloud and 
infrastructure topics, particularly to Kubernetes.

This shift on my focus actually opened my eyes: I have noticed that many problems that I was trying
to solve in my application code were actually solved in a cleaner and lighter way with the tools 
and primitives that are offered by the underlying platform.

## A case example: Spring Cloud Config and Kubernetes

As a case example, I want to take a look at the configuration management pattern 
I have seen in a project. The project consists of Spring Boot based microservices 
running on Kubernetes. It was using
[Spring Cloud Config Server](https://cloud.spring.io/spring-cloud-config/reference/html/){:target="_blank"} 
to centralize the configuration and 
[Spring Cloud Bus](https://spring.io/projects/spring-cloud-bus){:target="_blank"} 
with a message queue (e.g. RabbitMQ) to notify the services to refresh its application scope 
whenever there is a change on the configuration.

This setup got me thinking: Wasn't this an overkill 
for a pretty much solved problem by the underlying platform? Or even more generally: 
Is most of the functionality of Spring Cloud necessary at all when running on Kubernetes?
I am talking not only about the configuration management, 
but also the service discovery, load balancing and so on.

**Note:** I use Spring Boot and Kubernetes as an example in this post. However, similar concepts 
can be applied to projects written in other languages and frameworks, running on other platforms.
{: .notice--info}

## Reloading on configuration change: the Kubernetes way

Let's focus on this specific case and see how we would tackle it in a simpler way.

On Kubernetes, there are multiple approaches to "reload" the application when its configuration 
changes. Let's have a look into some of them.

In all of our approaches, we will use the `ConfigMap` and `Secret` resource types of Kubernetes.

**Note:** None of these approaches require a new build or running the tests. 
We do not build a new container image for the new configuration. A simple CI/GitOps pipeline
is enough to apply the configuration changes in short time.
{: .notice--info}

### a. Treating ConfigMaps as immutable resources

There is still an open [feature request](https://github.com/kubernetes/kubernetes/issues/22368){:target="_blank"}
on Kubernetes to implement to trigger a rolling restart to a workload (deployment, statefulset etc.) 
when the ConfigMap it uses is updated.

Yes, the ticket is still open, however, 
there is a currently [recommended solution](https://stackoverflow.com/a/40624029/1005102){:target="_blank"}: 
Treating ConfigMaps as immutable resources: 
For every configuration change, creating a new ConfigMap, 
and updating the reference on the workload manifest to point the new one.

When we do this, Kubernetes controller for that specific type of workload will detect the change
and trigger the series of events that will result in the pods of that workload to be gracefully 
terminated and restarted with the new configuration while adhering to the 
[update strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy){:target="_blank"} 
defined for that workload.

### b. Doing "`helm upgrade`"

If you are managing your deployments via [Helm](https://helm.sh/){:target="_blank"}, 
you can simply keep the checksum of the ConfigMaps as annotations on the pod template section 
of your workload. A change in the ConfigMap (that is sourced by a change in the Helm values) 
will cause a pod annotation change on the workload, and this will trigger a new set of pods to be 
scheduled, which will pick up the correct configuration.

Of course, the same story with the rolling update applies here to this approach as well: 
The upgrade will adhere to the update strategy of the workload.

Helm itself recommends this approach in their official [Chart Development Tips and Tricks](
https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments){:target="_blank"}.
Also, you can see that it is a [common pattern in many charts](
https://github.com/helm/charts/search?q=checksum%2Fconfig&unscoped_q=checksum%2Fconfig){:target="_blank"}.

### c. Using [Reloader](https://github.com/stakater/Reloader){:target="_blank"}

The open-source project Reloader implements the exact feature we need: 
To issue a rolling restart on our application when its configuration (ConfigMap or Secret) changes.

We need to deploy it to our cluster only once, and annotate our workloads to be reloaded 
when their configuration changes via `reloader.stakater.com/auto: "true"`, and that's it.

Needless to say, this solution also restarts the application with its own upgrade strategy. 

## What is the benefit?

You might be asking: Aren't these solutions _just some other solutions_ to the same problem?
How are they better?

In my opinion, the benefits are the following:

#### Configuration stored on Kubernetes
Compared to the Spring Cloud Config Server, which has support for various backends, 
these solutions use the  **native configuration solution offered by the platform**. 
The actual store it is stored is [etcd](https://github.com/etcd-io/etcd){:target="_blank"}, 
that is a distributed and a reliable database.  

**Note:** The actual source of your configurations should be your VCS. 
The comparison I do here mostly applies to the Config Server backends other than the Git backend. 
{: .notice--warning}

#### Minimal or no overhead

In the solutions `a` and `b`, we do not run any additional process 
to achieve our goal. We do not need to run a config server, 
which is a Java-based application (not very lightweight).
We do not need to configure the Spring Cloud Bus or any of its dependencies (e.g. RabbitMQ).
Only on option C, we run an additional service (Reloader), which is very lightweight process 
written in Go.  
  More importantly: fewer moving parts means less possibility of things going wrong.

#### Not relying on application context refresh

Spring framework supports refreshing and even restarting the entire application context without 
terminating the process. 

However, I think these features are prone to errors: 
Spring applications tend to get pretty complex over time,
so there are possibilities of memory leaks, unclosed resources, stale and/or broken internal 
state and so on. Some Googling can point you to issues people are facing with these features.

Even if we assume that these features are working perfectly, 
they can be **only as good as** a fresh process 
(which translates to a fresh pod with the new configuration in Kubernetes world).

#### The application only worries about the business logic

I think this is **the most important benefit**: By using the primitives offered by Kubernetes, 
namely the ConfigMap/Secrets, update strategies and pod templates, we achieved the same 
result without polluting our application codebase: we do not need a cloud config client, 
a cloud bus, or a support for context refresh. We do not need to be careful while writing our code 
to make sure the context refresh will work.

## Do not avoid deployments or restarts

If you have some working experience at enterprises, you might have also worked on an ages old legacy 
system. It is often very tricky to make new releases for these systems, and sometimes even more 
difficult to get these new releases deployed. And the worse part is, you need to deliver _somehow_.

Even if you didn't experience something like this personally, you might have heard the _horror stories_.

Both from my experience and from the stories I have heard, in this type of environments, 
a dirty workaround emerges over time: Moving the business logic from the actual application code to
the configuration and/or data. This can come in different forms: In one of the cases I have heard,
the developers started to implement the new features in stored procedures and manually inserted 
them to the database, which were then picked up by the application and executed.

Another form of this I have seen was storing scripts that are written in an interpreted language 
(Javascript, [MVEL](http://mvel.documentnode.com/){:target="_blank"} etc.) 
to avoid code change and deployment.

These workarounds actually point to a big problem: 
**Their build/release/deployment process is broken.**
Actually, the above workarounds are only making the situation worse over time. 
The cure these projects need is a fix on their deployment process. 

And imagine we managed to fix it: We would finally unlock our whole toolset. We would no longer be 
afraid of deployments, we would have business logic at a single place, at the place it belongs to: 
the actual application code.

Those days are mostly over thanks to many factors: the rise of the DevOps culture (+GitOps), 
microservices architecture, CI/CD, containerization and its orchestration and so on. 
However, old habits die hard: 
Still, many programmers try to do everything in their long-running process.

Nowadays, if we find ourselves ever "avoiding" deployments (or worse, restarts), 
we should change our aim to actually improve the deployment process and the pipelines, 
up to the point where we can confidently make new releases and get them deployed.

## Know your tools well

As you can see, you get all these benefits with one condition: better knowledge on your tools. 
More you know the tools at our disposal, more you can come up with simpler solutions to 
the problems you are trying to solve.

I used the configuration management as an example in this post, but in fact, 
it applies to many other concerns as well.
For example, when you are running on Kubernetes:
- Do not "push the logs to an external system". Simply log to stdout. 
  They will be collected by some other process and indexed for you.
- Do not try to do your own load-balancing. Kubernetes `Service`s already implement that for you.
- Do not worry about the service discovery. Call the services by their name. 
  [kube-dns](
  https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/){:target="_blank"} 
  will take care of that.
- Do not run any scheduled tasks inside your process. Instead, develop short-lived applications, 
  and set up [CronJobs](
  https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/){:target="_blank"}.
- Do not terminate SSL. Leave it to your 
  [ingress controller](
  https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/){:target="_blank"}.
- For distributed tracing, rate limiting, circuit breaking, check out service mesh solutions such as
  [Istio](https://istio.io/){:target="_blank"} or [Linkerd](https://linkerd.io/){:target="_blank"}.

## Conclusion

One of the key points for being a better developer is to know your tools well.

For a backend developer, this means (but not limited to) looking at what the infrastructure offers.
Kubernetes is a pretty widespread solution, and learning what it offers can change your approach to
solving your problems in a simpler, better way.
