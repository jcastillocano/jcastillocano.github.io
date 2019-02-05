---
layout: post
title:  "FOSDEM notes 2019"
date:   2019-02-03 14:08:39 +0000
categories: fosdem open source
---

## FOSDEM 2019 notes

This was my very first time at [FOSDEM][fosdem], although I've been following it since a while ago (thanks FOSDEM fellas for streaming 
all the main talks!). This year I had the chance to come over, thanks to my current employeer [System73][system73], along with some
coworkers. Main target this year for me was Infrastructure, Monitoring and Testing talks, and I had plenty of them (such as live
talks, offline chats or even just following links from some slides). Here are my top notch talks:

## Terraform

Two main talks about [Terraform][terraform], both driven by [Anton Babenko][anton] (such a good fella, main contributor on Terraform
AWS modules): one about good practices on Terraform, and another as round table session about testing in terraform. 

* [Codifying infrastructure with Terraform for the future][terraform_best_practices] 

Good talk from Anton, where he exposed some good practices about reusing code, [best practices][bestpractices] and some tips writing
Terraform code. He also showed us some interesting tools to incorporate into our ecosystem at System73 (see _Tools_ section below). 
Best quote of the talk was this one:

![Less code](/terraform.jpeg)

* Testing in terraform

This one was more like an interactive chat (like open sessions in DevOps days) where Anton drove the session, but anyone could express
and ask. It was a bit disappointed because I thought more testing stuff will be shown (in fact half of the room was against testing 
Terraform code, best way to test it is to actually apply and see what happens), but on the other hand some good tips where given, like
how to structure your modules to make them easy to reuse, reduce blast radius applying small changes and how to handle non-backward 
compatible changes from providers' APIs. 

### Tools

* [Atlantis][atlantis]: it was introduced as a collaborative tool for developing and applying terraform code. It seems it allows teams
to run terraform based on webhooks from pull requests, adding plan and apply results to code repository, thus increasing visibility and
enabling collaboration within your team mates in an easy and standard way. Honestly, an interesting one, if we manage to use it well it
will solve some of the problems we already have with terraform (code sharing, review, testing, etc).

* [Terragrunt][terragrunt]: another interesting tool, this one helps keeping clean our terraform configurations, allowing us to work with
terraform modules in a less messy way. Still not convinced until I give it a go, but pretty sure it will be a helping one given the fact
we're starting to have reusable code elsewhere.

## K8S

Almost everything on FOSDEM turned around [Kubernetes][kubernetes], so many talks that I couldn't cope with all of them. These are the
ones I could attend and what I learned.

* [Automate Kubernetes Workloads with Ansible][kubernetesworkloads]

How we can craft custom [K8S][kubernetes] controllers with [Ansible][ansible] [Operators][ansible-operators] to deploy and manage custom
workload in a cluster. It seemed to me a bit tricky at the beginning, but after a while I got some pretty neat advantages automating your
k8s controller with Ansible (easier to write, manage and customize resources if you already know Ansible).

* [Thanos - Transforming Prometheus to a Global Scale in a Seven Simple Steps][thanosglobalprometheus]

I could only watch last 10 minutes, so not too much feedback from my side. Perhaps that made me wonder even more how [Thanos][thanos] works and if
it makes sense to our organization (given the fact we have multiple Prometheus clusters world-wide), let's see in a few weeks :) 

## Prometheus

Like Kubernetes, there was a huge hype about Prometheus, with some talks related to it. Again, I managed to listen just one, but enough
to convince me Prometheus is the future for metrics.

* [Deep Dive: Kubernetes Metrics with Prometheus][deepdive]

It was for me an overview of how to retrieve K8S metrics automatically and query them on Prometheus, with some tips and tricks around
labelling and filtering. Very interesting if you're new on Prometheus scrapping K8S metrics out-of-the-box. It also linked me to 
[kube-prometheus][kube-prometheus] repo, with all required stuff to deploy an end-to-end kubernetes cluster monitoring.

## Extra stuff

* [Hackers gotta eat. Building a Company Around an Open Source Project][hackerseat] 

Cool talk from our inspiring Kohsuke Kawaguchi about how to build a business around an open software project: Jenkins.
He talked about all the different profitable steps Cloud Bees has followed until it became what it is nowadays. Besides,
I managed to grab a brand new Jenkins X T-shirt! 

* [To the future with Grav CMS][toolthedocs]

I guess the main takeaway from this presentation was we are not far from what other people do with docs at System73: treat them
as you will treat code. Apply same agile principles (verify, build, deploy) in an automated fashion way. In that way not only QA or
managers but devs will join keeping doc up to date without too much hassle.

* Brussels

FOSDEM is hosted in Brussels, one of the best european capitals I've ever visited, and it always welcomes FOSDEM with
its traditional belgian fries, waffels, beers and this year, snow! Here you have some pictures and a video of our 
Brussels' adventure.

![Brussels](/python.jpeg)

![Some Belgian beers](/beer.jpeg)

[![Snow](/snow.png)](https://www.juancarloscastillocano.es/snow.mp4)

## What else?

After two intense days, more coming at Config Management Camp this week! For those who will attend it, see you there!

[fosdem]:https://fosdem.org
[system73]:https://system73.com
[kubernetes]:https://kubernetes.io
[ansible]:https://www.ansible.com
[terraform_best_practices]:https://fosdem.org/2019/schedule/event/terraform_best_practices/
[bestpractices]:https://www.terraform-best-practices.com/
[terraform]:https://www.terraform.io
[atlantis]:https://github.com/runatlantis/atlantis
[terragrunt]:https://github.com/gruntwork-io/terragrunt
[anton]:https://fosdem.org/2019/schedule/speaker/anton_babenko/
[deepdive]:https://fosdem.org/2019/schedule/event/deep_dive_kubernetes_metrics_with_prometheus/
[ansible-operators]:https://opensource.com/article/18/10/ansible-operators-kubernetes
[kubernetesworkloads]:https://fosdem.org/2019/schedule/event/automate_kubernetes_ansible/
[thanosglobalprometheus]:https://fosdem.org/2019/schedule/event/thanos_transforming_prometheus_to_a_global_scale_in_a_seven_simple_steps/
[thanos]:https://github.com/improbable-eng/thanos
[multicloudkubernetes]:https://gitlab.com/multicloud-openstack-k8s/clusters
[hackerseat]:https://fosdem.org/2019/schedule/event/community_hackers_gotta_eat/
[kube-prometheus]:https://github.com/coreos/kube-prometheus
[toolthedocs]:https://fosdem.org/2019/schedule/event/gravcms/
