# Provisioning a demo Web project in Google Compute Platform

## Overview

This project aims to demonstrate how to set up an infrastructure for a small web application that has multiple http
webservers (with load-balancing and auto-scaling), and a single database backend.

Although it is not that difficult to build this via the GUI, the secondary goal is to accomplish all this in an
automated manner, so this whole infrastructure shall be replicable easily and without human interaction.


## The architecture

The installation of this infrastructure is based on the Google Cloud Platform, and all its parts will be deployed in a
single Project, so it will be completely separate from any other activities of our cloud account.

The files `provisioner.pem`, `provisioner.openssh.pub`, `service_account.json` contain credential information,
which has to be actualised with valid values before use.

For the autoscaling feature the webservers must be fully cloneable, so they will be a Managed Instance Pool, and for
that we need an Instance Template.

That Instance Template requires a Disk Image, which we create by provisioning a temporary ('golden') instance,
installing and configuring it as needed, and after creating the image and the template, we discard it automatically.

Strictly speaking, the Instance Template isn't part of our environment to install, on the per-environment basis we'll
only refer to it.

### Per-environment resources

An Instance Group Manager that (surprise...) manages a Managed Instance Group, whose members will be the
webservers, all of them created from an Instance Template (like the one we created above).

Additionally, there is one DB server, which is just a plain Instance we create right here (though we could've created
an Instance Template for that as well).

And finally, there the load balancing GCP resources: Backend Service, Target HTTP Proxy, URL Map, Forwarding Rule.


## Credentials

As the provisioner user must have access to the instances of the project, its ssh key needs to be added on project
level.

To do this, open the GCP web console, choose the project at the top, then on the side bar choose
[Compute Engine / Metadata / SSH Keys](https://console.cloud.google.com/compute/metadata/sshKeys).

Here you can *Edit* the list of keys, and *Add item* in the format `ssh-rsa AAAA.... username@somewhere.com`,
just like as you would add to a `.ssh/authorized_keys` file. The username will be the same as in the trailing
`user@...` part (the `...somewhere` part is not relevant).


## Creating the instance template of the webserver

This step has to be performed only once, and later the future webservers can just instantiate it.

That template is generated by creating a 'golden' instance, setting up all the packages and building all prerequisites
on it, and then using its disk to create a 'golden' image, and then we define the instance template as having the same
instance parameters and refer to this image when cloning the new disks on instantiation.

`ansible-playbook -i inventory.gcp_compute.yaml 0_create_instance_template.yaml -v`

(Or, with the log-recording helper script: `./run.sh 0_create_instance_template.yaml -v`)


There are some non-trivial things here in this step:


### Firewall rules

The module `gce_net` seems a bit unrelated to the `gcp_compute_*` modules, probably because it is based on `libcloud`.

Its attribute names are different (`project_id` and `credentials_file` instead of `project` and `service_account_file`),
and it explicitely requires a `service_account_email`, although it is already present in that .json file.

Therefore some templating trickery was needed to extract it and store it in a variable (`service_account_email`).

There is a TODO here: It works now, but if I eliminate that `service_account_content` variable by using a directly piped
`lookup(...) | json_query(...)` construct, it results an empty string. I'd like to find out why...


### Waiting for a newly created instance to become available

Specifying `status: RUNNING` will only ensure that the instance exists and is started, but not that its `sshd` is also
running and accepting connections.

That must be explicitely checked by that nice subsequent `wait_for` task, otherwise the next steps (like fact gathering)
would fail.

NOTE: That `wait_for` now uses the 0th public IP of the 0th interface, which is the intended address in this setup.
In another future setup it may be different (for example, when provisioning from within the local network), in that
case it should be modified accordingly.


### Refreshing the inventory

A newly created instance doesn't appear automatically, so we must explicitely trigger that refresh.
It's well documented, but I've rarely seen it in the examples on the net.

If there are multiple instances created, it's enough to refresh the inventory only once, when leaving this
locally ran play.


### Waiting for an instance to go down

Checking SSH availability isn't enough here, because then the instance is still STOPPING, so it holds a reference on
its boot disk, so we can't use that for creating an image.

We must explicitely wait for the TERMINATED status. It's nice to see that there is an easy, straightforward, documented
and efficient way to do this.


### The transitions between local, remote and again local plays

Each play must be separately executable, so they can't rely on information collected/produced by a 'prior' one.

This is why I'm rather collecting `gcp_compute_disk_facts` than just export them at the creation phase.


## Creating the DB server

No real surprise here, it's much simpler than the instance template above, see `1_create_db_server.yaml`.


## Creating the webservers and the load balancing

This was a complex one :D.

First of all, the load balancing concepts of GCP are by themselves complex enough (global vs. regional, external vs.
internal, premium vs. standard), and not all features are available in all scenarios.

Second, the current state of the `gcp_compute_*` modules doesn't cover everything, there are open issues.

We want a Managed Instance Group (indentical instances 'cloned' from an instance template), that needs an Instance
Group Manager, but that's still straightforward enough. Note though, that it (naturally) belongs to a *zone*.

The HTTP Health Check is also quite simple, it just contains parameters and doesn't refer to anything else.

The next level is the Backend Service.


### Backend Service

According to the GCP docs, a Backend Service can be either global or regional, but `gcp_compute_backend_service`
supports only the global one, see this [issue](https://github.com/ansible/ansible/issues/52319).

Not a problem here, because GCP supports HTTP load balancing (with session affinity controlled by cookie instead of
the plain src+dst+proto hash) only on global Backend Services, but I've ran a few circles until I figured it out that
I'll need a global one anyway.


### Target HTTP Proxy and URL Map

A Target HTTP Proxy contains *nothing* but a reference to a URL Map, so I don't really see the point in them
being separate entities. It seems a clear 1-1 relationship to me, at least as of now.

And the important information: Target HTTP Proxies are inherently global, they don't belong to any region.


### Forwarding Rule

And here we are :D.

Global vs. regional again: their targets must match their globalness. And since Target HTTP Proxies are global,
we need a **Global** Forwarding Rule here. And that implies the PREMIUM tier.

Just note that we need `gcp_compute_global_forwarding_rule` instead of `gcp_compute_forwarding_rule` here.

And another consequence of the concept: As this is a **global** resource, it takes some time while our changes propagate
worldwide, so our freshly created Forwarding Rule **won't just work immediately, but only after some 5 minutes**.

First when I saw that it was created but it wasn't working, I thought I mis-configured something, so I just destroyed the
infrastructure, read the docs, started to investigate and tried some alternatives, and again, and then again...
Of all these, maybe the 'read the docs' was the only positive thing, the rest was just wasting almost a day on chasing
shadows.

So, again: for Forwarding Rules **changes take time to propagate, >= 5 minutes before it starts working**.

The other thing to note is that the `target` attribute of these modules may refer not only to Target Pools, but to Target Proxies as well.

## Destroying the infrastructure

As the counterparts of the creator playbooks, there are destroyers as well:

* `7_destroy_webservers.yaml`
* `8_destroy_db_server.yaml`
* `9_destroy_instance_template.yaml`

They just define the same resources in reverse order, with `state: absent`.

Some of these resources have mandatory attributes that otherwise are irrelevant when destroying the resouce (like
`gcp_compute_target_http_proxy` must have a `url_map` reference anyway), in these cases these attributes must be
present but may be empty.


## Misc notes

### GCD terminology about Load Balancers

On the default web interface of [Load Balancing](https://console.cloud.google.com/net-services/loadbalancing)
Google tries to keep the complexities under the hood as much as possible, which is good from a user perspective,
but we need the Advanced menu instead.

There we see every entity we discussed above, except for the URL map. On the other hand, on the basic view we
see things that are called Load Balancers, while there are no such entities on the API. Well... they seem to mean
the same thing: if we create an URL map, it'll appear as the representative of this whole chain of entities that
are in fact the 'Load Balancer'.


### Why not Terraform

At first I tried to create the infrastructure by Terraform and do only the sw provisioning by Ansible.

Then I hit some problems where the cure would've been worse than the disease, so to speak, and the reason was the
difference between the nature of the problem and the nature of Terraform.

Terraform aims to achieve a 'desired' state of interdependent resources and it's quite an adequate tool for it.  But its
main concept is to be **declarative**, meaning that in the configuration we describe that single desired *state* of the
system, and not the way that leads to it.

You can't simply describe a *recipe* like:

1. Create an instance
2. Configure it
3. Create a template from it
4. Stop the instance
5. Destroy it

The reason of being unable to do so is that there is no single *desired* state of that instance. Step 1 wants to achieve
that it exists and is RUNNING, then Step 2 wants to achieve ...  something that is not represented in the state of the
instance, then Step 4 wants to achieve that it's STOPPED, and finally Step 5 wants to achieve that it doesn't even exists.

So, when we were to *declare* that instance in a Terraform file, what should we specify for its *state*?

Sure, it could be done (by using `null_resource`-s), but it means actively hacking the original concept,
essentially *tricking* the DAG-resolver to perform the *sequence of steps* we want.

Basically we would be mis-using a declarative mechanism to do some definitely programmatic task. Besides, its config language
HCL isn't quite orthogonal (a feature that works with *this* won't necessarily work with *that*), so IMO Terraform isn't
designed to describe and support infrastructures that

* Have arrays of higher-level resource structures, or
* Uses external tools/steps and wants to transfer information both to and from it, or
* Needs sequential *recipes* that do multiple state transition on the same resources

It can be hacked to do that, but that'll be much more counterintuitive to read, even than a shell script, and it would take
more effort to *force* Terraform to do what we want than what it takes to do the actual job itself.

It may be a good tool, but it's not the best one for this job.
