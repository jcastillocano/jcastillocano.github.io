---
layout: post
title:  "Manage Azure Monitor Datastores in Grafana automatically with Terraform"
date:   2018-01-10 08:08:39 +0000
categories: terraform grafana azure monitor
---

## Problem

Working with [Grafana][grafana] we recently discovered this datasource: [Azure Monitor
Datasource][azure-monitor-datasource], which covers most of the monitor use cases we have on our Azure environments
([Application Insights][application-insights] for app's metrics and [Azure Monitor API][azure-monitor] for system metrics). Since we are already using [terraform
grafana][terraform-grafana] provider to provision our Grafana installation with standard datasources (such as Prometheus) and
custom dashboards, we gave this custom datasource a go. 

Given the fact this datasource is not a standard one, sadly it's not covered by current fields that resource [data\_source][datasource]
provides out of the box. Checking Azure datasource documentation, we will need to make use of both *json_data* and *secure_json_data* but in a different way than terraform does: we don't require *auth_type*, *default*, *access_key* and *secret_key*, in fact, we need new fields like *client_id*, *tenant_id*, etc. so we cannot use this terraform resource straight away.

NOTE: you need to install Azure Monitor Datasource in advance. Have a look at [Grafana documentation][install-plugins] in order to install plugins
on your current server (using grafana cli, docker env variables, etc).

## Workaround: API calls

Grafana provides an [HTTP API][grafana-api] we can use to handle our custom datasources. Default values for Azure
Datasource parameters:

 * _type_: grafana-azure-monitor-datasource
 * _access_: proxy
 * _url_: https://management.azure.com
 * _basicAuth_: false

and require parameters:

 * *[Application Insights][config-app-insights]*
   * _appInsightsAppId_
   * _appInsightsApiKey_
 * *[Azure Monitor API][config-azure-monitor]*
   * _clientId_
   * _subscriptionId_
   * _tenantId_
   * _clientSecret_

One datasource can cover both (Insights and Monitor API) or just one of the endpoints. To create a new datasource, we need
either authorization token or user/pass (seek Grafana documentation) to authorize our requests. A simple scenario using
auth token:

```
curl -H "Content-Type: application/json" -H "Authorization: Bearer MyUltraSecretTokenForGrafana" -X POST -d @azure-insights-datasource.json http://127.0.0.1:3000/api/datasources
```

where the content of _azure-insights-datasource.json_ is:

#### azure-insights-datasource.json
{% highlight JSON %}
{
  "name": "AzureTest",
  "type": "grafana-azure-monitor-datasource",
  "access": "proxy",
  "url": "https://management.azure.com",
  "basicAuth": false,
  "isDefault": false,
  "jsonData": {
    "appInsightsAppId": "xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx"
  },
  "secureJsonData": {
    "appInsightsApiKey": "abcd1234efgh5678jklm9012nopq"
  }
}
{% endhighlight %}

this creates a new Azure Monitor datasource as follow:

![Azure Monitor datasource](/datasource.png)

Adding Azure Monitor API fields we can update our datasource to fetch both App Insights and Monitor metrics.

## And now in Terraform

First, let's say we need terraform code to provision grafana (default datasources, custom dashboards, etc.). Define a
_provider_ specifying url and auth (token, previously generated) as follow:

{% highlight HCL %}
provider "grafana" {
  url  = "http://127.0.0.1:3000/"
  auth = "eyJrIjoic29mRkF6bUxCTXV4a3U5Tk5Rb0U0a1pPaDFZQjVZNVAiLCJuIjoiVGVycmFmb3JtIiwiaWQiOjF9"
}
{% endhighlight %}

After that, you can run `terraform init` and `terraform plan` to check everything works (there is nothing to apply yet
though). We need to define our custom datasources, but as we saw above we cannot use *grafana_data_source* resources.
But don't panic, here is where [template_file][template-file] and [null_resources][null-resource] come to help! Using
both _template\_file_ (to create a template of payloads) and _null\_resource_ (to curl Grafana API) block we can create
many custom datasources:

{% highlight HCL %}
resource "template_file" "my_azure_monitor_template" {
  filename = "azure_monitor_api_datasource.json"

  vars {
    name            = "AzureMonitorDS"
    client_id       = "${var.monitor_client_id}"
    subscription_id = "${var.monitor_subscription_id}"
    tenant_id       = "${var.monitor_tenant_id}"
    client_secret   = "${var.monitor_client_secret}"
  }
}

resource "null_resource" "my_azure_monitor_dashboard" {
  provisioner "local-exec" {
    command     = "curl -X POST -H \"Content-Type: application/json\" -H \"Authorization: Bearer ${var.grafana_token}\" -d'${template_file.my_azure_monitor_template.rendered}' ${var.grafana_url}api/datasources"
    interpreter = ["/bin/bash", "-c"]
  }
}
{% endhighlight %}

be sure you define all datasource variables required for this approach (client\_id, tenant\_id, etc.) as well as Grafana
variables (*grafana_url* and *grafana_token*). In this case, instead of a plain json we will use _template\_file_ to
allow variable interpolation (thus secrets are stored in _.tfvars_). Current template files:

#### azure\_monitor\_api\_datasource.json
{% highlight JSON %}
{
  "name": "${name}",
  "type": "grafana-azure-monitor-datasource",
  "access": "proxy",
  "url": "https://management.azure.com",
  "basicAuth": false,
  "isDefault": false,
  "jsonData": {
    "clientId": "${client_id}",
    "subscriptionId": "${subscription_id}",
    "tenantId": "${tenant_id}"
  },
  "secureJsonData": {
    "clientSecret": "${client_secret}"
  }
}
{% endhighlight %}

#### azure\_insights\_datasource.json
{% highlight JSON %}
{
  "name": "${name}",
  "type": "grafana-azure-monitor-datasource",
  "access": "proxy",
  "url": "https://management.azure.com",
  "basicAuth": false,
  "isDefault": false,
  "jsonData": {
    "appInsightsAppId": "${app_insights_app_id}"
  },
  "secureJsonData": {
    "appInsightsApiKey": "${app_insights_api_key}"
  }
}
{% endhighlight %}

#### azure\_datasource.json
{% highlight JSON %}
{
  "name": "${name}",
  "type": "grafana-azure-monitor-datasource",
  "access": "proxy",
  "url": "https://management.azure.com",
  "basicAuth": false,
  "isDefault": false,
  "jsonData": {
    "appInsightsAppId": "${app_insights_app_id}",
    "clientId": "${client_id}",
    "subscriptionId": "${subscription_id}",
    "tenantId": "${tenant_id}"
  },
  "secureJsonData": {
    "appInsightsApiKey": "${app_insights_api_key}",
    "clientSecret": "${client_secret}"
  }
}
{% endhighlight %}

Refer each filename on _template\_file_ resources based on what datasources you want to create.

# Summary

We manage to handle azure datasources using terraform resources in a different way (templates and null resources). This
allowed us to close the circle and define our grafana installation 100% with code (terraform to provision
infrastructure, ansible to install Grafana and configuration, and terraform again to provision Grafana datasources and
dashboards).

To be said, this is just a shortcut for this particular issue (custom datasources not supported on Terraform). Ideally,
what we should do is to extend [terraform-grafana-provider][terraform-grafana-provider] to support custom datasources.
Feel free to open an issue or implement it if you are familiar with go and terraform providers!

We found this issue interesting, but we know there is a long way to go.
If you think on a different approach, or an improvement, please leave your comments on github
or send us an email, we'd like to know about other options!

[grafana]:https://grafana.com/grafana
[azure-monitor-datasource]:https://grafana.com/plugins/grafana-azure-monitor-datasource
[application-insights]:https://docs.microsoft.com/en-us/azure/application-insights/
[azure-monitor]:https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-get-started
[terraform-grafana]:https://www.terraform.io/docs/providers/grafana/index.html
[datasource]:https://www.terraform.io/docs/providers/grafana/r/data_source.html
[grafana-api]:http://docs.grafana.org/http_api/data_source/
[config-app-insights]:https://dev.applicationinsights.io/quickstart/
[config-azure-monitor]:https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-create-service-principal-portal
[template-file]:https://www.terraform.io/docs/providers/template/d/file.html
[null-resource]:https://www.terraform.io/docs/providers/null/resource.html
[install-plugins]:http://docs.grafana.org/plugins/installation/
[terraform-grafana-provider]:https://github.com/terraform-providers/terraform-provider-grafana
