# Fun With Openstack - Deploying Workload with Terraform

After installing Openstack and playing around with deploying cloud workload (Virtual Machine Instances) using both the web UI ([Horizon](https://docs.openstack.org/horizon/latest/)) and [command line client](https://docs.openstack.org/python-openstackclient/latest/) [^1], the next step in the game is to try to automate this manual procedure. Afterall, doing it manually can become rather tedious - Just look at the number of config options you need for a VM, not to mention virtual networks!

> In this post, we will be studying a third-party sample scripts - I did *not* write any of it. The repo is at [https://github.com/diodonfrost/terraform-openstack-examples](https://github.com/diodonfrost/terraform-openstack-examples)

The script we will be looking at is for [Terraform](https://www.terraform.io/). Another option that achieve similar effect is Openstack's own [Heat](https://wiki.openstack.org/wiki/Heat) orchestration engine. Heat has its own template language called [HOT](https://docs.openstack.org/heat/latest/template_guide/hot_guide.html), among other languages it supports. However, we won't be talking about it (or use it in real life) [^2] for the reason that Heat is specific to Openstack only, while Terraform is multi-cloud and also an important, general purpose tool that is part of the DevOps "Standard Toolbox".


## Background: What is Terraform?

The [official documentation](https://www.terraform.io/intro/index.html) says that "Terraform is an infrastructure as code (IaC) tool that allows you to build, change, and version infrastructure safely and efficiently". I personally prefer a more specific description (in my own words):

> Terraform is an **Infrastructure as Code (IaC)** tool that focuses on *provisioning infrastructure*, with supports for various clouds through **Provider**. It takes a **declarative** approach, with a template language (Terraform Configuration Language) that uses variables, modules, etc to encourage clean, reusable code. Moreover, Terraform automatically track resources dependencies, and uses **state** to ensure correctness in case of changes. In particular, Terraform is **idempotent** like other decent IaC tools.

That's a lot to stomach for a beginner, so let's unpack it a bit:

* Infrastructure as Code is a set of practise to 1) manage infrastructure automatically, and 2) make such automation programmable. This is usually achieved by providing some form of programming language to describe the infrastructures. While automation is an important benefit, it is the shift in mindset to programming that is the most important, as it implies bringing forth well known best practise in coding - version control, clean code/modularity/refactoring etc.
* Provisioning is highlighted as this is the area of focus of Terraform. For instance, Ansible, another popular IaC tool, is more on the Configuration Management side. To put it in layman's term, Terraform let you spin up cloud resources including VM, and Ansible then ssh into the VM and perform server setup tasks (like installing software packages) automatically.
* Provider is a kind of Plugin in Terraform. Due to the large number of different cloud providers/vendors, each with their own API, it makes more sense for Terraform Core to implement a framework, and let each provider handle the mapping into suitable API calls.
* Like in much of the tooling scene, there is usually two contrasting approach to code: An imperative style let user specify how to do certain actions, sometimes in a low level manner; a declarative style on the other hand try to absract away from this and let user specify what the goal/thing is. In general, declarative style is more intrinsic and can feel more elegant, and is usually more maintainable provided one has not stretched its limit. However, for complicated situations, one usually hits up limit of the declarative language and for those cases an imperative style can be the right thing to do.
* State: TODO
* Idempotency means that repeated application of the same procedure is equivalent in effect to applying it just once. It is an important goal of IaC tool. To see why, imagine a rudimentary automation for deploying to server using shell scripts. If any one were to accidentally call it, unaware that it has already been executed before, the resulting unintended side effects can be devastating. Idempotency would help insure against this. In an age where continuous integration, automated testing, and repeatability of IaC automation is the norm, this property is crucial to letting us run the code frequently without worry.


## Basic Example

Let's start with a toy example that is still immediately useful. We will provision a single VM, no frills or tricks. But before reading the code, make sure you setup your enviornment: follows the instruction on the main page of the repo (i.e. `README.md`). Openstack authentication is needed because Terraform ultimately ends up sending API calls to your Openstack installation, which is of course protected. You can get your credentials from the Openstack dashboard (Choose “Access & Security” -> "Download OpenStack RC File". The file is a shell script that exports enviornmental variables containing the required info)

Go to the directory `01-sample-instance/`. For comparison, we will also look at the [training tutorial of Galaxy Project](https://galaxyproject.github.io/training-material/topics/admin/tutorials/terraform/tutorial.html) side by side.

### Variables and Syntax

The first file is `00-variables.tf`.

```hcl
# Params file for variables

#### GLANCE
variable "image" {
  type    = string
  default = "Centos 7"
}
```

The `#` symbol is for comments, and we see that this file contains all the variables. This is a good practise as it isolates the parts that may change frequently. Otherwise, the file is unremarkable for seasoned programmer even if you've never worked with HCL before: variable declaration contains the type, as well as a value (it can be overriden in the command line directly, or using a `.tfvars` file, etc). The apparently strange syntax is here because **Terraform Resources** also uses the same syntax - we will see soon enough.

Skipping to the end for an example of support for composite data type:

```hcl
variable "network_http" {
  type = map(string)
  default = {
    subnet_name = "subnet-http"
    cidr        = "192.168.1.0/24"
  }
}
```

Now is a good time to check the [docs](https://www.terraform.io/docs/language/index.html) for the formal syntax. The gist of it is summarized with this code block:

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.base_cidr_block
}

<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION> # Argument
}
```

And this paragraph:

> Blocks are containers for other content and usually represent the configuration of some kind of object, like a resource. Blocks have a block type, can have zero or more labels, and have a body that contains any number of arguments and nested blocks. Most of Terraform's features are controlled by top-level blocks in a configuration file.

So a block is sorta like a hashmap, but not quite.

Note the following points:

* The "value of a key" is an expression and support programming features like variables, substitution, functions etc. This is also a way for Terraform to [discover implicit dependencies between resources](https://learn.hashicorp.com/tutorials/terraform/dependencies).
* testing

In later scripts, we will refer back to these variables, so keep this in mind and look it up on this file if needed.

### How to work with Resources

As we mentioned in the background section, Terraform relies heavily on Providers to implement integrations with various cloud vendors. It can be argued that this is what gives Terraform its true power - being able to work with various vendors on a uniform platform is essential to enabling the multi-cloud use case, and even if you're not in that use case, being able to learn a single language and then goes on to being able to operate against many clouds is a big plus.

Each provider implement a set of *resources*, with each resource mapping to concepts in the respective clouds. For example, in the Galaxy Project Tutorial, the VM instance is instantiated using the following snippet of code:

```hcl
resource "openstack_compute_instance_v2" "test" {
  name            = "test-vm"
  image_name      = "denbi-centos7-j10-2e08aa4bfa33-master"
  flavor_name     = "m1.tiny"
  key_pair        = "${openstack_compute_keypair_v2.my-cloud-key.name}"
  security_groups = ["default"]

  network {
    name = "public"
  }
}
```

As we can see, compute instance is one resource. And there are many others. How do we work with them? Fortunately, Terraform have an [online registry](https://registry.terraform.io/). So we can search for it there - enter the keyword "Openstack", and we end up on [this page](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest). Now click "Documentation" on the top right hand corner, then on the left panel, click "Resources" to see a list of resources it provides. The filter can also be used: enter "compute" to narrow the list down to those related to the Nova (Compute) Project.

> Tips: If you know the exact name of the resources, you can also search for it directly.

The resource's documentation page is [here](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs/resources/compute_instance_v2), and we can see that some common examples are given.

For in-depth understanding, please read the "Argument Reference" section there.

In principle, other resources' documentation can be found in the same way.

Going back to our business. The above example is a pretty minimal/self-contained one, and even then we see that creating an instance is not exactly trivial. At minimum, we have the following dependencies:

- **A SSH key-pair:** it is best practise to use a public/private key pair to login to VM, instead of setting up a password, because password has many potential weakness. Such as human being prone to using weak password, or the fact that we will be entering secret materials into a plain text file (HCL) that may even get into the repo for source control (given our DevOps practise). By contrast, keys are generated client-side and usually, your private key will *never* leave your own computer.
- **Security Group:** For security, by default VMs in Openstack are behind a restrictive firewall policy that forbids all traffic. To actually begin using it, we should add a Security Group that permit in-bound traffic for the SSH port (so we can login to our VM through SSH), and out-bound traffic to the public internet (so that the VM has internet access). Further ports should be opened depending on use case - if we want to use it as a http server, we should allow in-bound traffics from http(80)/https(443) ports.
- **Networking:** In short, it is complicated. Openstack has its own networking component called *Neutron*, which we will get to shortly.

In case it isn't obvious by now, all three dependencies above are also separate Resources. This is why the example above is minimal - it effectively inlined all dependencies that can be inlined.

Now let's look at the two simpler resources. First SSH keys (`010-ssh-key.tf`):

```hcl
resource "openstack_compute_keypair_v2" "user_key" {
  name       = "user-key"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDAjpC1hwiOCCmKEWxJ4qzTTsJbKzndLotBCz5PcwtUnflmU+gHJtWMZKpuEGVi29h0A/+ydKek1O18k10Ff+4tyFjiHDQAnOfgWf7+b1yK+qDip3X1C0UPMbwHlTfSGWLGZqd9LvEFx9k3h/M+VtMvwR1lJ9LUyTAImnNjWG7TaIPmui30HvM2UiFEmqkr4ijq45MyX2+fLIePLRIF61p4whjHAQYufqyno3BS48icQb4p6iVEZPo4AE2o9oIyQvj2mx4dk5Y8CgSETOZTYDOR3rU2fZTRDRgPJDH9FWvQjF5tA0p3d9CoWWd2s6GKKbfoUIi8R/Db1BSPJwkqB"
}

```

and then security group (`030-security_group.tf`):

```hcl
resource "openstack_compute_secgroup_v2" "ssh" {
  name        = "ssh"
  description = "Open input ssh port"
  rule {
    from_port   = 22
    to_port     = 22
    ip_protocol = "tcp"
    cidr        = "0.0.0.0/0"
  }
}
```

Both are pretty self-explanatory. However, there is one subtle point that is general and crucial to remember: **The label/name pattern should be used for resource dependencies tracking to work in Terraform Core.**

Here is what I mean. If you look up the doc for the compute instance resource, you would find that usually an Openstack id or name is used to link to/refer to related resources. But we shouldn't hard-code it as Terraform doesn't know the details of third party implemented resources and wouldn't know of the dependencies. Instead, we do this:

- Assign the Openstack name of a dependent resource in the declaration of that resource. E.g. `name = "ssh"` for the security group.
- Add a second label to that dependent resource. To trim down on the number of names you need to invent, you can reuse the same name. Hence we have `resource "openstack_compute_secgroup_v2" "ssh"`.
- To get back the Openstack name while still allowing Terraform to track dependencies, use the Terraform Label as a filter/query to select the resource, then extract the name or id attribute. That is, `openstack_compute_secgroup_v2.ssh.id`.

### Setting up Network Resources in Openstack

There are quite a number of network related resources in Openstack. Here's a short summary for a refresher:

TODO

The Galaxy Project Tutorial aim for a barebone setup that attach VM directly to the external network. This is usually what you'd do for something that closely matches the classical "VPS" experience. If you want to do VM orchestration, then we'd want to prepare the ground work before hand and have something like a VPC network ready.

The script `020-network.tf` contains the following:

```hcl
#### NETWORK CONFIGURATION ####

# Router creation
resource "openstack_networking_router_v2" "generic" {
  name                = "router-generic"
  external_network_id = var.external_gateway
}

# Network creation
resource "openstack_networking_network_v2" "generic" {
  name = "network-generic"
}

#### HTTP SUBNET ####

# Subnet http configuration
resource "openstack_networking_subnet_v2" "http" {
  name            = var.network_http["subnet_name"]
  network_id      = openstack_networking_network_v2.generic.id
  cidr            = var.network_http["cidr"]
  dns_nameservers = var.dns_ip
}

# Router interface configuration
resource "openstack_networking_router_interface_v2" "http" {
  router_id = openstack_networking_router_v2.generic.id
  subnet_id = openstack_networking_subnet_v2.http.id
}
```

In prose, what it does is this: create a virtual router named `router-generic`, a network named `network-generic`, and a subnet inside our new (virtual) network, whose name is the variable `var.network_http["subnet_name"]`. A CIDR block is assigned to the subnet (configurable through variable). The router have two interfaces, one to the external network (through the `external_network_id` attribute), and another to our subnet (connected through an explicit `openstack_networking_router_interface_v2` resource).

Please keep this basic setup in mind, as we will build upon it for more elaborate scenario later.

### Putting it together

And finally to our instance! Look at `060-instance_http.tf` one section at a time:

```hcl
resource "openstack_networking_port_v2" "http" {
  name           = "port-instance-http"
  network_id     = openstack_networking_network_v2.generic.id
  admin_state_up = true
  security_group_ids = [
    openstack_compute_secgroup_v2.ssh.id,
    openstack_compute_secgroup_v2.http.id,
  ]
  fixed_ip {
    subnet_id = openstack_networking_subnet_v2.http.id
  }
}
```

TODO

Then our instance:

```hcl
resource "openstack_compute_instance_v2" "http" {
  name        = "http"
  image_name  = var.image
  flavor_name = var.flavor_http
  key_pair    = openstack_compute_keypair_v2.user_key.name
  user_data   = file("scripts/first-boot.sh")
  network {
    port = openstack_networking_port_v2.http.id
  }
}
```

Since we are using the virtual network approach, we need to assign public IP address separately. In Openstack, this is done through a *floating IP*:

```hcl
# Create floating ip
resource "openstack_networking_floatingip_v2" "http" {
  pool = var.external_network
}

# Attach floating ip to instance
resource "openstack_compute_floatingip_associate_v2" "http" {
  floating_ip = openstack_networking_floatingip_v2.http.address
  instance_id = openstack_compute_instance_v2.http.id
}
```

Note that this is a two step process: first we reserve a floating IP from the pool. Then we associate this IP to our instance.

### Volumes

In the second scenario of the repo, he showcased how to use Volumes. There are two cases:

- For the http server, an empheramal volume is attached to the instance. The chosen OS image is copied to the volume on start-up.
- For the DB server, a persistent volume is separately created and then attached.

In `060-instance_http.tf`:

```hcl
# Get the uiid of image
data "openstack_images_image_v2" "centos_current" {
  name        = var.image
  most_recent = true
}

...

resource "openstack_compute_instance_v2" "http" {
  ...

  # Install system in volume
  block_device {
    volume_size           = var.volume_http
    destination_type      = "volume"
    delete_on_termination = true
    source_type           = "image"
    uuid                  = data.openstack_images_image_v2.centos_current.id
  }
}
```

In `061-instance_db.tf`:

```hcl
#### VOLUME MANAGEMENT ####

# Create volume
resource "openstack_blockstorage_volume_v2" "db" {
  name = "volume-db"
  size = var.volume_db
}

# Attach volume to instance instance db
resource "openstack_compute_volume_attach_v2" "db" {
  instance_id = openstack_compute_instance_v2.db.id
  volume_id   = openstack_blockstorage_volume_v2.db.id
}
```

## More Realistic Examples

If you have been patiently following along, you probably know where this is going already.


## Extensions and further thought

### Cloud-init

Often we want to customize the VM instances, such as running script, installing packages, setting up a particular tech stacks, configuring OS level parameters, etc. While simple use cases can be handled using a shell script, and `ansible` can be used for systematic setups, there is a third option, `cloud-init`, that is an industry standard for the cloud settings. It also has the advantage that it supports tight integration with cloud capability.

Again, from the Terraform Docs for `openstack_compute_instance_v2`, an example is given:

```hcl
resource "openstack_compute_instance_v2" "instance_1" {
  name            = "basic"
  image_id        = "ad091b52-742f-469e-8f3c-fd81cadf0743"
  flavor_id       = "3"
  key_pair        = "my_key_pair_name"
  security_groups = ["default"]
  user_data       = "#cloud-config\nhostname: instance_1.example.com\nfqdn: instance_1.example.com"

  network {
    name = "my_network"
  }
}
```

It also note that

> `user_data` can come from a variety of sources: inline, read in from the `file` function, or the `template_cloudinit_config` resource.

https://registry.terraform.io/providers/hashicorp/cloudinit/latest/docs/data-sources/cloudinit_config

https://registry.terraform.io/providers/hashicorp/cloudinit/latest/docs

The cloud-init Terraform provider exposes the cloudinit_config data source, previously available as the template_cloudinit_config resource in the template provider, which renders a multipart MIME configuration for use with cloud-init.

Use the navigation to the left to read about the available data sources.



```hcl
data "cloudinit_config" "foo" {
  gzip = false
  base64_encode = false

  part {
    content_type = "text/x-shellscript"
    content = "baz"
    filename = "foobar.sh"
  }
}
```

https://registry.terraform.io/providers/hashicorp/template/latest/docs/data-sources/cloudinit_config

https://learn.hashicorp.com/tutorials/terraform/cloud-init


### Metadata, Barbican, and Passing Secrets Securely to VM

### Bastion Host

### VPNaaS and Hybrid Cloud

VPNaaS is a relatively obscure add-ons to Openstack Neutron, that adds ability to setup a site-to-site VPN - this "bridges" two geographically separated network together, while ensuring that traffics passing through the public internet are encrypted (Hence "virtual" and "private"). In the cloud context, a main use case for this is that it enables hybrid cloud - using both public cloud and private cloud together, with the VMs deployed on these two clouds being able to communicate as if they live on the same network.

Unfortunately, the VPNaaS software seems rather fragile from my personal experience trying to deploy it. Therefore, an alternative is to DIY. We follow the example at [UKCloud](https://docs.ukcloud.com/articles/openstack/ostack-how-configure-ipsec-vpn.html), but convert the heat template into terraform.



End

[^1]: Hello

[^2]: That being said, Heat is still a prerequisite to installing important components of Openstack, such as [Magnum](https://wiki.openstack.org/wiki/Magnum) for Kubernetes-as-a-service (similar to AKS, EKS, and GKE in public cloud). As Kubernetes is super-important, we can't completely ignore Heat either if we are serious about working with Openstack.
