---
layout: post
title:  "Managing multibranching projects on Kubernetes"
date:   2018-03-02 08:08:39 +0000
categories: kubernetes git version gae
---

## Let's try something different

After months working with [Google App Engine][gae] using both standard and flex environments, we decide to try some
of our apps to [Kubernetes][kubernetes], where we have more flexibility in terms of libraries (crypt), services (redis),
local disk usage, etc. We wanted to keep our multibranching strategy (multiple versions deployed at the same time, each
one with its own endpoint, some of them sharing common database and third party services).

Kubernetes, out-of-the-box, provided us most of the stuff we required to make our apps work (blue/green deployments,
versions, etc) on dev environments (first rule: we don't talk about Production environments). Within this post I'll try
to cover some stones we found on our path, as well as delve into the fixes or workarounds we implemented.

## Main caveat: DNS

When you are dealing with external DNS, Kubernetes is not the king fo the party. Services like Google App Engine
provide you external dns enpoints automatically for every version you deploy, while with Kubernetes you don't get more
than a public IP. Even though there are some extensions (have a look at [external-dns][externaldns]), the rule here is
_do it yourself_. 

At the moment, we've dealing with IPs only (since it is develop), but we have some ideas on the oven. IP works for us
since we only access features from our laptops, so after a new feature is deployed we fetch the Load Balancer IP created
by Kubernetes and we update our config. But the problem raises with service discovery comes to play.

As I said before, on Google App Engine you get endpoints like _your-version-your-project.appspot.com_, with HTTPS
support, and they are discovered automatically. However, if you want to use your own dev domain, you still need to work
out a solution that requires API calls (GoDaddy, CloudFlare, etc). That will work for Kubernetes too, so let's give it a
go! From our CD server (Jenkins), one of the latest steps on our deployment jobs is to update DNS provider with feature
endpoint, no matter if we are using Google App Engine or Kubernetes, and so far it's been working like a charm! 

## Feature branches

Based on [continuous-delivery-jenkins-kubernetes-engine][cdkubernetes] post, we've implemented multibranching (features)
following the same criteria: using *namespaces*. Each feature has its own namespace where we can create pods, secrets, 
configmaps, etc.

![Continuous Deployment using versions](/continuous-deployment.png)

We currently deploy in our Pods a frontend container (Nginx with static content, domain config, proxy, etc) and backend 
container (python app). We also added a DB sidecar container (see DB access section below) to connect to Google SQL
endpoints.

## DB access

Another caveat. From Google Kubernetes Engine you cannot connect directly to Cloud SQL instances, you need a sidecar
container with granted permissions and especial config (have a look at [Connect to Google SQL from Kubernetes][connectkubernetes]
for more details). This involves adding an extra container to each pod (even though it is a really low resource
consuming container, it creates another error point) but that's how it works on Google Cloud. Nothing else to say.

## Secrets

We required three set of secrets for our deployments:

 * *TLS*: nginx ends SSL connections with our own SSL certificate; in that way we could use custom domains instead of
   google ones.
 * *Google Container Registry*: images are pulled from a private container registry on Google. Our CI images are pushed
   to our central repository after being tested, but we need this secret in order to pull them from different clusters.
 * *CloudSQL*: two secrets are required in order to make the connection between the app and the MySQL server through SQL
   proxy (_cloudsql-instance-credentials_ and _cloudsql-db-credentials_).


## Maintenace tasks

What happens when a feature is deleted from the repo? Who handles deprecated DNS records? Are old images discontinued?
All these things (among some others) are handled by a trigger from Github (when a branch is deleted); we trigger a job
in Jenkins which takes care of this actions. Don't forget it if you don't want to run out of resources or have a huge
bill in the future!

## What else?

Well, I'm pretty sure there is still a lot of room for improvement. We'll post more updates about this work in the
upcoming weeks, but to give you a glimpse we've working on feature-monitoring, so stay tuned!

[gae]:https://cloud.google.com/appengine/
[kubernetes]:https://kubernetes.io
[externaldns]:https://github.com/kubernetes-incubator/external-dns
[cdkubernetes]:https://cloud.google.com/solutions/continuous-delivery-jenkins-kubernetes-engine
[connectkubernetes]:https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine
