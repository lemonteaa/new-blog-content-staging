# Hitchhiker's Guide to OpenStack

I have a confession to make. I wasted way too much of my life away trying to learn OpenStack, and faced up to repeated failures.

## Why this Guide?

OpenStack have both an on-ramping issue for beginners, and is notorious for being difficult to deploy successfully. We will focus mostly only on the first point.

What is the issue exactly?

Well, unlike Kubernetes, which arguably have similar complexities, OpenStack apparently lacks a good ecosystem of learning resources that is accessible to beginner, despite being a large and highly organized piece of software. The learning resources that are available pretty much assume that you are already well-versed in Operating System, Infra, Cloud, DevOps, etc. And if you get into any problems during installation or use, often the only search results are official docs and issue tracker (if you are lucky). Also, it is virtually gauranteed that you *will* run into some issue, quite probably exotic, if you are serious in learning OpenStack.

To make matter worse, the official documentation is a Kafkasuque sprawling mess, while the issue trackers (Hint: *not* github), together with the official source repo living on self-hosted servers, are also difficult to navigate. God help you if you actually want to join development effort - there is elaborate rituals involved, and you will need to use some custom heavy toolings during actual development.

I know this sounds negative, but this is not over yet. The hardest hitting part is that the second point also hurt the first point. To see why, look at Kubernetes again, they have accessible ways for beginner to get a working cluster to get their hands dirty on: `Minikube` and `k3s` are both good options that are close to "push-button", and there are also some specialized cloud options that are oriented towards development and testing instead of production. The same story just doesn't play out as nicely on Openstack (TODO).

Now I don't completely blame OpenStack. Despite their similarities, there are still some intrinsic difference that are due to their nature and not poor execution on OpenStack's part - for example Workload on Openstack tends to be heavier in terms of resource requirement than those on k8s, so you do need more resources (mainly RAM) to do realistic experiments on Openstack. However the end result is still the same - this presents a higher barrier of entry to beginner that hurt the adoption/mindshare, which then feedbacks and contribute to the "ecosystem not large enough" problem.

Now that I'm done ranting, let's begin. It would be arrogant of me to think that a simple, naive guide like this will magically solve all the problems, but what I do know is that most solutions, barring certain "genius", are ultimately composed of atomically small steps - it is the accumulation and persistence that matters.

### More comparison with Kubernetes

I think we can learn a lesson or two from Kubernetes's success. Here are the things I think k8s did right, and cleverly so:

- Although both k8s and Openstack are *deceptively* highly customizable (emphasis on "deceptively" and not just "customizable" per se), k8s did a better job of not letting this negatively impact the on-ramping, by hiding this complexity using a combination of sensible defaults and k8s Distributions. It is only when you reach the advanced level of learning where you examine the internal workings, that this truth is gradually rolled out to you.
- The "push-button" deploy options mean that k8s can use a "Top-Down" teaching strategy - let beginners get engaged with k8s and have working results with tight feedback quickly, or in other words "quick wins". This makes it easier to "hook-up" beginner into progressively using k8s more and more. Note that for this to work some contextual factors are necessary, such as there being a ubituous prescence of k8s as an actual production deployment option across a wide range of cloud service vendors, k8s having a monopoly on container orchestration, and there being a *rigid demand* of deploying applications using containers.
- Big Corporate interests are willing to publish public learning resources in a form that is reasonably vendor-neutral.


## Prerequisite

Since this is explicitly targeted to beginners, I only assume that (cue sarcasm):

- You have read the usual layperson intro to cloud computing (i.e. API-control, commodity view and burstable usage, economy of scale and the CapEx vs OpEx considerations, IaaS vs PaaS vs SaaS, and finally public/private/hybrid/multi-cloud)
- You preferrably have used at least one of the big public clouds somewhat and have a rough idea of the mode of access to cloud services, and the range of services offered.
- You have basic knowledge in using command lines.
- You have basic Networking knowledge - you know what is an IP network, understand IP addresses and subnets, and can manually configure one. With extra help you can configure routing and firewall. For the other parts you have at least heard of them - DNS, DHCP, proxy, TLS certificates. And Layer 2 if you plan to learn the internals deeply.
- You know what is a web server/backend.

I specifically do *not* assume deep knowledge of OS or infra.

## Background and Context

### What is OpenStack?

### A Short Note on Versioning

Openstack uses a 6 months release cycle. The major versions are codenamed using A-Z as the first letter for easier memorization. As of Jun 2022, the current release is Yoga.

As you search the web for articles/docs, please keep in mind that there are major changes to Openstack when the version is sufficiently different and so the info may no longer be applicable.

Also, what happens after Zed (we run out of the alphabets)? According to [this](https://governance.openstack.org/tc/reference/release-naming.html),

> After the release "Zed", each OpenStack release has an identification code: "year"."release count within the year"
> 
> Example:
> 
> OpenStack 2023.1 Axxxx
> 
> Where "2023" is the year of the release, "1" represents the first release of the year and "Axxxx" is the release name.

### OpenStack Components and Recommendation

OpenStack uses a modular design and consists of a large number of components, each implementing a particular kind of cloud service. Here I group them by how essentials they are. Within the first group they are arranged in installation dependency orders - those listed later may depends on earlier components. For components without interdependency I arrange to have the "easier" component come first, so that hopefully it presents a gradual learning curve.

- Absolute minimum
  - Keystone (Identity)
  - Glance (Image)
  - Cinder (Volume)
  - Neutron (Network)
  - Nova (Compute)
  - Horizon (Dashboard) (but see also Skyline for a modern revision)
- Essentials/Popular
  - Swift (Object Storage)
  - Heat (Resource Orchestration)
  - Magnum (Kubernetes)
  - Octavia (Load Balancer)
- Extended Universe
  - Ironic (Bare Metal)
  - Barbican (Secrets)
  - Trove (Database)
  - Murano (Application Catalog)
- Niche or Less well maintained

Note that there are also these "components" that are technically not a component due to their special nature:

- Ceilometer (Somewhat deprecated) - Collect usage metric, historically used for billing
- Cloudkitty - Modern component for billing
- Centralized logging - collect all Openstack logs and ship to an ELK/EFK stack
- Openstack unified client - for end-user access to the cloud

Finally, I strongly suggest beginner to, at least in the fist iteration, focus on getting the absolute minimum installation working, instead of being over-ambituous and try to install additional components.

### A Note on terms and jargons

- Ferret token:

## Concepts

### The most important Realizations

This is the single most important section in the whole article.

If I can time travel to the past, I would simply tell myself this three points, which I think can save me countless hours of futile efforts because I misunderstood the true nature of OpenStack:

1. OpenStack is a Framework, not a library.
2. OpenStack *manages* cloud resources, it is not responsible for physically provisioning them.
3. OpenStack is designed to optimize the "throughput" as you scale up.

Now let's elaborate.

**OpenStack is a Framework, not a library.** It gives you a bunch of components, and have what looks like a standard installation. But all components are highly customizable, moreover, there is actually no officially mandated *only* way to put these components together - all of these decision are ultimately up to you. Thus it is actually more accurate to conceptualize OpenStack not as one single software product, but as a framework giving you some ground-work and raw materials, which *you* them use to design, build, implement, and finally deploy your own cloud. Because of this, it is actually kind of misleading in daily conversation to say that "X is using the OpenStack private cloud on premise", as there is no such thing as "the" OpenStack cloud. Indeed an argument can be made that no two OpenStack clouds are the same. (While some may regard this as a weakness due to incompatibility etc, this is more of a value-choice)

**OpenStack manages cloud resources, it is not responsible for physically provisioning them.** So how does anything gets done? Well, OpenStacks relies on other existing, usually Open Source Technologies that have already implemented various aspects of virtualizations. It delegates the physical action onto these softwares, and monitors/oversees the progress. Combining with the first point (and also just coming down to good engineering sense of loose coupling), the design choice is obvious: use a plugin architecture. That is, for each class of cloud resources, have many plugins all providing roughly the same service, each plugin using a different underlying software. All plugin of the same class should conform to the same interface, so that OpenStack can just call the interface to get the job done. Administrator can choose/swap out the plugin as he see fits. (This can also partly be viewed through the lens of policy-mechanism separation)

**OpenStack is designed to optimize the "throughput" as you scale up.** Because of this it is willing to place a potentially large, but O(1) overhead if there is good return at large scale - the costs itself will be amortized away due to the sheer scale. The reason for such a choice is because the main target audience - those that wish to replace the public cloud with their own private cloud, need to advance an economic case to persuade the decision maker to make the switch/adopt it. And to be competitive one need to at least try to match the economny of the public clouds. As the public cloud pushes the costs down mainly by having large scale, OpenStack decided to go head-to-head and optimize for the large-scale case. Sadly, the interests of the beginner is a collateral damage - now you need to have some buffy machines to actually deploy even a test instance of OpenStack.



### Various Virtualization Technologies

- Compute: QEMU/KVM, libvirt
- Software-Defined Networking (SDN): Linux Bridge, OpenVSwitch (OVS)
- Storage: LVM, Ceph, and a whole lots more

### How OpenStack Works

(Keep in mind point 2 in the key realization above as you read this)

In short, watch this youtube video. The gists come down to this:

The API Server (connected to a Database) recieves request from end user. After making a decision and updating the DB if necessary, it sends a command to the relevant agents on a possibly different node using the message queue. The agents then execute the physical action to provision the cloud resources. Notifying the API server of the result/updating status of the cloud resources uses similar mechanisms.

### Reference Deployment Architecture

Although there is no officially mandated deployment architecture, and for practise one will most likely be using an *all-in-one* installation where everything is cramped onto a single node, it is still important to be aware of some details of a production style deployment, because the documentation will make implicit/tacit reference to it.

Below I sketch my own personally biased, selfishly designed architecture. The context is that of a generic tiny data center providing a self-service type cloud service. It is intended to mimic a production deployment without actually being one - what is important for beginners I think is to *get* the spirit without getting bogged down by some of the more tedious details.

<insert diagram here>

Physical Node types:

- Controller:
- Compute node:
- Network node:
- (Optional) Storage node:

We are responsible to properly configure the physical networking ourselves. Here's it:

- Management network: Connect all nodes under a single IP network. Controller sends command to change cloud resources to other nodes through this network.
- Provider network: Connection among the compute nodes and the network node. Carry user/tenant traffic. Packets are vxLAN.
- External network: Connect the network node to a gateway router that further connects to the public Internet.

Note that the API server on Controller should be proxied to the public internet through an extra web server node. This node also hosts the Horizon.

(Note: an alternative terminology is "control plane" for the management network and the "data plane" for all networks carrying user/tenant traffic.

### Extras for Data Center and Public IP

The architecture above is enough to get a rough idea, but it lacks realism. Here, let's try to iterate on that.

Reference:

Short summary: In a real data centers, the servers will be organized in racks, there will be a "top-of-rack" switch on each rack connecting all the servers. Then these switches are in turn interconnected through a second layer of switches, which are them connected to the gateway router.

TODO: iBGP.

## Installations

### Comparison of the methods

There are basically three classes of methods: Development/Ephemeral installs, Automated/Production installs, and Manual installs.

#### Development/Ephemeral installs

Recommended Use Case: Use on a throwaway VM or cloud based ephemeral instance for quick experimentation/tinkering.

Note that the minimum system requirement is 4GB RAM (plus some more for the VM workload you run on it) and at least 20GB of remaining disk space. (Some are for the OpenStack software, some are for storing image files and for provisioning the default ephemeral block storage while creating a VM)

*DevStack*

*Pros*

- Close to being a "push-button" deploy while remaining cross-OS.

*Cons*

- Won't work after a reboot, even restarting services won't work - you need to run the `unstack` scripts then start over.
- Configurations option are not well documented - you may need to examine the source code.

*Packstack*

*Pros*

- Even more "push-button" than DevStack as the configurations are more beginner-friendly.

*Cons*

- Only for Redhat derivative distros (CentOS, Fedoras).

*MicroStack*

*Pros*

- Perhaps the most "push-button" of the competitions as it is literally just a single command.

*Cons*

- The official System Requirement is higher than others in this class - 8GB of RAM and 100GB disk. This may make it off-limit to developer on a tight machine.



#### Automated/Production installs

*kolla-ansible*

*Pros*

- A cross-OS, officially supported method.

*Cons*

*TripleO*

This is related to the PackStack. Full name is "**O**penStack **O**n **O**penStack" - you install a simple OpenStack (the undercloud) and use this cloud to help install the production strength actual cloud (the overcloud).

*Charmed OpenStack*

This is Cannonical/Ubuntu's counterpart to TripleO.

*Pros*

- Retain the smooth, automated experience at production strength as opposed to the competitions - no complicated manual configurations nor boostrapping.

*Cons*

- Uses a platform that is on a subscription basis.



### Summary of manual installation flow

Even if you're not using this method, it is recommended to understand how things work under the hood - other installations are essentially automation of this flow.

There are three phases:

*Phase 1: Preparation and Install dependencies*: As mentioned in the section on reference architecture, you are responsible for configuring the physical networks of the node. The most important dependencies are the database (MySQL or mariadb) and the message queue.

*Phase 2: Main components install*: OpenStack Components available as OS official packages. This include `deb` (install via `apt` in Debian and Ubuntu) and `rpm` (install via `yum` in CentOS and Fedoras) packages. For API servers, they are implemented in Python using django, and integrates with SystemD - this provide the convinence that you can control them using `systemctl` and are also configured to start upon server bootup. Just like a typical web backend, you should bootstrap the Database scheme (and potentially dump some initial data).

*Phase 3: Admin user setup*: This happen on the user-side of OpenStack. OpenStack comes with a special class of end user (admin) that have privileged access to certain restricted class of actions. These actions are often related to setting up and configuring the cloud enviornment, and sometimes involve access to sensitive part of the cloud (e.g. access to underlying hypervisor or the SDN). This is done by using the `openstack` unified client, the same as what an actual end user would user.

<TODO: what about plugins/agents>


### Configurations


## Daily Life

### Tips on Troubleshooting

OpenStack should be thought of as a complete tech stack that spans all the way from the bottom (physical hardwares like routers, and say the supporting stuff like SAN or NAS with careful consideration of the interconnects) to the top (the web backend). One should take a holistic, or end-to-end approach to troubleshooting. It is important to always be clear about where in the stack, and which component, does a specific error message come from in order not to get lost. To this end it is useful to know the components involved in an action and the flow/interactions among the components.

In more recent version all logs are sent to the OS's builtin log facility. Thus instead of `grep`ing log files the old sysadmin way, one can do `journalctl`.

For a more sophisticated experience - if you have a powerful machine to use - consider turning on centralized logging so that you can view, search, filter, and trace the log from the comfort of a modern, Elasticsearch/Kibana dashboard.

Some Common Errors:

- Insufficient resources to satisfy request: This can be unintuitive as this error is masked from user, and requests for a VM goes through the placement service first.
- Wrong Network Type: Some resources can only be deployed on certain Network Type.


## Epilogue and Other Misc. Notes

### Some suggestions for customizations

- Keystone integration with external/third party Identity Providers
- LUKS encryption of volume in Cinder and integration with Barbican
- BGP announcement of floating IP
- GPU instance types
- SR-IOV

### Why Distributions didn't solve the on-ramp issue



