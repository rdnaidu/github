---
layout: single
classes: wide
title:  "How to Setup an static ip address to a Google Compute Engine Instance"
date:   2017-08-25 20:18 -0500
tags:
  - Google Cloud Platform
  - Google Compute Engine
  - Google Cloud Console
  - VM
  - Virtual Box
  - network config
categories: google-cloud-platform google-compute-engine
excerpt: "Setup an static ip in a Google Compute Engine instance"
---
By default Google Compute Engine instances are created with ephimeral IP address. What if you want to set to them an static IP that doesnt change? 
For instance, the most common scenario would be to link to that IP a catchy <abr title="Domain Name System">DNS</abr> for your web application or API.

I found a solution in 2 steps:

## Delete the current possible ephimeral ip configuration:

{% highlight bash %}
gcloud compute instances delete-access-config <instance> --access-config-name "External NAT"
{% endhighlight %}

Where `<instance>` is the name of the instance you want to update, and **External NAT** is the name of the configuration, which probably has that value because of its the default one. And you can check it running this:

{% highlight bash %}
gcloud compute instances describe <instance> --zone=us-west1-a
{% endhighlight %}


## And add the static one 

If you want to bind an static address, probably to bind it to a DNS address, execute something like this

{% highlight bash %}
gcloud compute instances add-access-config <instance> --access-config-name="External NAT" --address=xxx.xxx.xxx.xxx
{% endhighlight %}

Remember to always append the `--zone` to any **gcloud** command to avoid any ambiguity. You can get the `address` from the valid addresses configuration you have in your project, which you can be gotten like this:

{% highlight bash %}
gcloud compute addresses list
{% endhighlight %}

Dont use the `NAME` but the `ADDRESS`. You should pick an address in the same zone of your instance. When the address be attached you will see that in the `STATUS` field of the last query it will say **IN USE**. 

Et voila!