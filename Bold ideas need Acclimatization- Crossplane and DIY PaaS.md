# Bold ideas need Acclimatization: Crossplane and DIY PaaS

Sometimes, we come across an idea so bold and radical that it appears insane initially. However, after multiple rounds of research and testing, it gradually becomes clear that the idea actually make sense - just not intuitively. Moreover, in retrospect, the idea may even appear obvious.

The last thing you want to do in this scenario is to then go about telling everyone how obvious this great invention is - because its obviousness is conditional on having a relatively deep understanding of what's going on. Instead, we may have better luck by acclimatizing an idea - to bring little nuggests of it in small but consistent installments, each tiny little step that make sense on its own.

Crossplane is one such example.

## The need for customized PaaS

Heroku is still the gold standard for a magical Developer Experience that "just work" - the promise of a PaaS is that it largely removes the need to consider DevOps from developer. Just follow the twelve factor app patterns, `git push`, and the platform will figure out the rest.

However, it turns out that in practise many companies do have special requirements such that an off-the-shelf solution can only fit 80%, and the remaining 20% is absolutely critical. Sadly, implementing your own PaaS is such a daunting tasks that even well-resourced company can struggle, asides from taking away lots of time.

## Kubernetes is a Platform to build a Platform

Since PaaS have fallen out of fashion, Container-as-a-Service (CaaS) have somewhat replaced it and continued the line. As container technology become acknowledged, established, and then entrenched in the market, the next thing back then is what is called the "Container Orchestration War" - basically different kinds of competing cluster manager sus it out. The details may differ, but they all promise to use a cluster of node to host, and then orchestrate a fleet of containers to provide higher level functionality.

We all know what happened. Kubernetes from Google "won" and become a virtual monopoly. Kubernetes crystalizes operational knowledge/elements of what Google consider to be needed in a production deployment. In the aftermath, Kubernetes (k8s) become so widely adopted by cloud vendor that it emerges as if it's a vendor-neutral platform for container. Are PaaS back in the form of a disguise?

Well, yes and no. It is true that a common use case of k8s from the developer's perspective is to just use it as if it is a PaaS. But k8s is too low level to be an actual PaaS - you are exposed to elements that would fall squarely within the realm of operation, and writing a `yaml` config to deploy your app involves too much details that makes it tedious compared an actual PaaS.

Despite these short-comings, developer have no better alternative, so they used it as a clutch. And... it worked. Kinda. If you are willing to invest time to learn it, and somehow able to cross the language barrier to a more infra side, that is.

Should k8s be blamed for this? No, because that's not its intended usage. A [twit](https://twitter.com/kelseyhightower/status/935252923721793536) captured this wisely:

> Kubernetes is a platform for building platforms. It's a better place to start; not the endgame.
> 
> Kelsey Hightower


Just like how in technology stack we always have lower layers, upon which upper layers libraries/frameworks etc are implemented, k8s would be perfect if we think of it as an abstraction layer that provide easy access to common patterns in infra/operation. It would then be up to us to coordinate these elements into features of an actual Platform.

## Generic System Components

Suppose we want to implement PaaS on top of k8s. Aside from doing some further abstraction, let's say that an App Components automatically unpacks to `Deployment`, `Service`, and `Ingress` objects, there's also the need for some generic system components that are independent of any particular apps from the end user, and moreover aren't included by default in k8s. Some examples include (but are not limited to):

- CI/CD system
- Container image registry (to store the result of a `docker image build`
- Logging and Monitoring stack

For comparison, example of features that *are* provided within k8s itself:

- Rollout strategy
- Health check
- Ability to rollback

Anyway. We make two observations about these components before going to the next steps:

- Although the k8s ecosystem is *huge* and have additional in-cluster softwares that can satisfies many, if not most, of these demands, there is no intrinsic law of the universe that force us to do it this way. It is perfectly possible to use softwares providing these functions that are just standalone software and itself have nothing to do with k8s.
  - Example: Harbor for container registry, Argo/flux for gitops. Example of softwares outside the k8s ecosystem: Jenkin classical (but see JenkinX), Prometheus (but there's an k8s operator - see next point)
  - Aside: for non-k8s software, it is still possible to bring it into the k8s ecosystem by containerising the software itself.
  - Aside: Advantage of using an in-cluster version is that tight integration is possible...
- These components correspond to a system-level concern and should be separated from application level workload. e.g. We may have many apps in a cluster, but they all share the same Prometheus instance for monitoring.

## Ingredient 1: Control Plane vs Data Plane

See:

- https://www.cloudflare.com/learning/network-layer/what-is-the-control-plane/
- https://konghq.com/learning-center/cloud-connectivity/control-plane-vs-data-plane

This is a basic concept in cloud software. In short the control plane is a centralized place where you exert administrative control over the end-user/tenant's workload, and the data plane is where the workload actually happens.

Continuing the discussion, the system level components above would mostly belong to the Control Plane. And for the sake of discussion, let's assume that the Control Plane is physically a k8s cluster. Now comes the interesting part - what is the Data Plane in our use case?

### Self-service cluster

It would be a (separate) full k8s cluster (!). This can be a bit tricky so let's think about it. In a multi-tenancy situation, where your users may be on different team (or even organization) and don't know/trust each other, it make sense that users should get their own dedicated, exclusive cluster. No one should be allowed to peek into other ppl's cluster.

So what would happen is that upon a user's creation, we should say create/provision a new k8s cluster for him/her (preferrably using cloud API). Depending on how much control/meddling we want, we may also say manages the k8s namespaces in that cluster on his/her behave. Each namespace should correspond to the user's different projects.

The remaining problem would be - does the same hold if all users belong to the same team/org? Wouldn't it be silly to have exactly 2 k8s clusters (1 control plane and 1 data plane), with one of them controlling the other? Why not just merge them? Although tempting, this can become an anti-pattern for two reasons. The first is that their semantics/meaning are different while their apparently similarity is only a coincident and should not be relied on. The second is that your situation may change in the future into a multi-tenancy scenario, if you didn't plan that ahead you'd be locked into a deadend.

## Ingredient 2: Custom Resource Definition and Infrastructure as Code

We made a small leap of faith in the last section and arbitrarily dictated that the Control Plane be a k8s cluster. Why? Indeed it is theoretically possible to do an alternative technological choice, but then you wouldn't be using CrossPlane. As there is no absolute reason for it to be, I can only offer potential advantages:

- As the Control Plane hosts a number of production grade components, they themselves are subject to the same question of production-readiness/ops side of DevOps. k8s is the most straightforward answer - just host it inside k8s, and you get most of those taken care of almost for free.
  - (Note that CrossPlane itself does not come with this and it is indeed possible to externalize all of these components elsewhere)
- k8s have a wide and deep mindshare, and the patterns it has established is well known by many. By riding on it, platform engineer can avoid having to learn something entirely new from scratch.
- k8s also provide implementation reuse of those patterns - we don't need to implement them ourselves, we just use k8s.

The last two points may be difficult to wrap your head around (it is for me), and I don't have any magic tricks other than to think through it more. The LLVM article may also help: https://danielmangum.com/posts/crossplane-infrastructure-llvm/

What I surmise is this: It is well known that k8s's basic model is the [synchronization/control loop pattern](https://kubernetes.io/docs/concepts/architecture/controller/), and it allows a declarative approach to deploying your resources. But the pattern itself is fully general and isn't really restricted to deploying containers only.

Indeed, that pattern can be compared to other tools in the IaC (Infrastructure as Code) space, such as terraform. In both you'd write the configuration/setup/composition/structure of your infra you want in a declarative fashion, then submit them (`terraform apply` vs `kubectl apply`). There are both similarities and difference in how the request is realized into actual infra: both undergoes a process of comparing desired vs current state; Terraform uses state explicitly (https://www.terraform.io/language/resources/behavior), while k8s is more implicit and have the controller + controller manager running behind the scene, reacting to the changes in desired state.

Continuing this comparison, k8s is also extensible like Terraform can be extended through Plugin/Provider. k8s's `CustomResourceDefinition` (CRD) let you define your own custom resource alongside normal k8s resources like `Pod`. They are ultimately just API object (read: *data*) that can be CRUD'ed. The second part is to write your own custom controller (eg https://medium.com/speechmatics/how-to-write-kubernetes-custom-controllers-in-go-8014c4a04235 ). It's basically a Go program that listen to events coming from the api server and perform any actions needed to make the physical world match up the desired state.

### So, what is CrossPlane?

You can go check the official definition yourself, though I find expressing it in my own words may help by skipping the jargons. **We basically just hijack the extension mechanism in k8s to make it works like IaC tools like Terraform, for the purpose of providing higher level abstraction of infra to developers in the form of a Domain Specific Language (DSL).**

Maybe an example is in order. Let's say we find it too tedious to require developers to write yaml for 3 resources just to deploy a backend app. Why not collapse it into one resource, but with all those "surrounding config" being required as well?

So we may define XRD (Crossplane's analog of CRD), so the end user will just create XR (Composite Resource), say named `BackendApp`.

In a naive case, we'd then write a custom controller, read the config values of the XR, then create all 3 underlying resources (`Deployment`, `Service`, and `Ingress`) with part of those values filled in from the XR's value.

But CrossPlane find that even this is too tedious, and so they provided a kind of "Low code" approach to it. Instead of controller, you define an additional resource called `Composition` - which is like a template for the actual resources you want to create as the underlying of the XR. In a `Composition` you'd specify the base yaml for each of the constituent resources, then patch them selectively using yaml path from the XR's value. (Ref https://crossplane.github.io/docs/v1.9/reference/composition.html)

### Why multi-cloud? And where does the name crossplane come from

In the most basic use case, that'd be it. But CrossPlane have something more in mind. It is designed to be multi-cloud from the core - Just like how Terraform has Providers for various cloud vendors (some are developed by the official team, but many are community contributed), so too does CrossPlane supports using its core mechanisms (XR) to control resources in other cloud through Provider. For example, an AWS Provider let you manage AWS resources in your AWS account (which uses AWS cloud API behind the scene).

Now the question is, why? I suspect it is to make it more useful for a wider class of use case. For example you may decide to outsource things that are not your Core Competency to third party cloud vendors, say RDBMS and Logging (because ELK/EFK can't satisfy you), but deploy containers in your own k8s clusters, and wire them up together (inject connection URL of the DB to your container's env. variable).

Finally, what's the name CrossPlane? It is actually Cross-cloud Control Plane. If you'd read this far, hopefully that phrase completely make sense now ;).

## Mode of consuming CrossPlane

- One can implement his own web UI and web server for his PaaS, then call the Control Plane k8s API directly to CRUD the XR.
  - One can leverage the `kubectl` command line client integration provided by CrossPlane to implement his own command line client that shells out to that.
- One can implement a Terraform Provider that map to these XR directly.
- One can also use a "Managed k8s cluster" pattern - give access to your cluster to a third party vendor hosting the Control Plane and let it provision multi-cloud resources, which are then wired up/injected to your apps.

## In Retrospect - recursion hurts

At the end of this journey, it appears to me that why it might be confusing is because it have kind of a recursive pattern - like using kubernetes not directly, but to control another kubernetes (at least in one possible use case). I did not get that you are totally allowed to have more than one kubernetes cluster in the first place until later on.

## Bonus: Open Application Model (OAM)

Reference: https://www.murillodigital.com/tech_talk/crossplane/

We mentioned above that k8s is a platform to build a platform. A side effect of developer using k8s directly as a clutch is that the distinction between different roles - like specifying app deployment vs building a platform vs managing your cloud account - is somewhat blurred. OAM aims to clarify by defining them more precisely.

In a related way, it is regretable that although the Twelve factor app model has been pretty good, it get over-shadowed somewhat by k8s. OAM's resource (Components, Application Scopes, Traits and Application Configurations) aim to recapture that nice model in a cloud/tech-agnostic manner.

Perhaps the main value of OAM is in giving a clarified set of vocabs to faciliate communication, collaborations, as well as just clear thinking.

## Reference

https://www.linkedin.com/pulse/crossplane-writing-custom-provider-from-scratch-shailendra-sirohi/

https://blog.upbound.io/diy-k8s-paas/

https://www.infracloud.io/blogs/custom-control-plane-crossplane/

https://thenewstack.io/crossplane-a-kubernetes-control-plane-to-roll-your-own-paas/

> An application model is needed to standardize an application’s configuration settings so they can be deployed in multiple environments. Last year, Microsoft and Alibaba teamed together to create one, the Open Application Model. An application model simply details all the different requirements that the application needs to run, including password pointers, configuration settings, hardware requirements, cluster settings if it’s a Kubernetes app, and so on.
> 
> While Kubernetes operators also offer a way to capture this information for automation, it doesn’t offer a clean separation between administrator and developer responsibilities, leaving the developer to suss out the network settings and the like, Prasek said.

https://www.infoq.com/articles/crossplane-paas-kubernetes/

https://www.techtarget.com/searchitoperations/tutorial/Step-by-step-guide-to-working-with-Crossplane-and-Kubernetes

https://danielmangum.com/posts/crossplane-infrastructure-llvm/

https://blog.upbound.io/the-crossplane-resource-graph/

https://www.murillodigital.com/tech_talk/crossplane/

The Open Application Model and Crossplane - Part 1

An agnostic way to build application centric platforms and infrastructure in a team centric way, across clouds

https://www.infoq.com/articles/kubernetes-successful-adoption-foundation/

https://medium.com/building-a-kubernetes-platform-for-fun-and-profit/building-a-kubernetes-platform-for-fun-and-profit-part-one-fc7f10fd1a5e

https://itnext.io/why-everyone-builds-internal-kubernetes-platforms-284c2cf76226
