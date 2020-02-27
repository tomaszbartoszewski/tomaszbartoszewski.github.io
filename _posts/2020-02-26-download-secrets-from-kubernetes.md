---
layout: post
title:  "Download secrets from Kubernetes"
date:   2020-02-26 20:00:00 +0000
categories: [kubernetes]
tags: [secrets, bash, minikube, kubectl]
---

When your service doesn't work as expected and you want to run it locally, being able to use same secrets can be useful. I will show you how to download them from Kubernetes with kubectl.

If you only want to get a script, jump to [Full download secrets from Kubernetes script](#full-download-secrets-from-kubernetes-script)

## Why can it be useful

If you use Kubernetes for running your services, you likely need some configuration for them. In my case there is often GCP service account key, certificates to connect to Kafka, some config file for the application, sometimes flyway config. As usual everything is great when it works, as soon as something breaks you likely have to debug your application. We generate our config files and secrets with terraform, we have templates so all environments are the same, only values which we pass to those secrets are different. That makes it tricky to generate exactly the same files, it would be likely very time consuming.

If you have your secrets in kubernetes and can connect with kubectl, you can see what’s there.

## Requirements

For this demo I'm going to use Minikube. I used it on Ubuntu and MacOS, on many occasions it was a great way to test something, without affecting anybody else. If you want to install it, you can find instructions [here](https://kubernetes.io/docs/tasks/tools/install-minikube/)

After installation you should get it set as your current context. To check you are connected to the correct cluster run:
{% highlight shell %}
kubectl config get-contexts
{% endhighlight %}
You should see similar output (a star indicates current cluster):
```
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   
```

If you are using MacOS and have Docker Desktop installed, when you press the icon in top bar, you should see Kubernetes and you can switch between contexts there, if you prefer to use terminal run:
{% highlight shell %}
kubectl config set-context minikube
{% endhighlight %}

I highly recommend setting autocompletion for kubectl, if you are using it often, you can find instructions [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)

For iterating over json fields I'm using `jq`, you can read more about it [here](https://stedolan.github.io/jq/)

## Let's create our secret

You may likely have different namespaces in your cluster, so we will create one to simulate real life situation. To check what namespaces you have in your cluster run:
{% highlight shell %}
kubectl get namespaces
{% endhighlight %}
That should give you something similar to:
```
NAME              STATUS   AGE
default           Active   46m
kube-node-lease   Active   46m
kube-public       Active   46m
kube-system       Active   46m
```

Now we can create a new namespace.
{% highlight shell %}
kubectl create namespace download-demo
{% endhighlight %}

For this demo I will create a secret, you can use yaml file if you like, I will use commands as it’s easier to copy, paste and run.

Let's create locally a directory and two files which we will use as secrets.
{% highlight shell %}
mkdir secretsdirectory && \
echo "{ \"message\": \"very secret\" }" > secretsdirectory/config.json && \
echo "Vanitas vanitatum et omnia vanitas" > secretsdirectory/quote.txt
{% endhighlight %}

Now we can create a secret in Kubernetes.
{% highlight shell %}
kubectl --namespace download-demo \
create secret generic my-application-secrets \
--from-file=secretsdirectory/config.json \
--from-file=secretsdirectory/quote.txt
{% endhighlight %}

## Extract value from a secret

Now when you get secrets, you should see it on a list.
{% highlight shell %}
kubectl --namespace download-demo get secrets
{% endhighlight %}

Previous command was useful to see all secrets but how about the content?
{% highlight shell %}
kubectl --namespace download-demo get secrets my-application-secrets -o yaml
{% endhighlight %}

Gives you information about your secrets and content:
```
apiVersion: v1
data:
  config.json: eyAibWVzc2FnZSI6ICJ2ZXJ5IHNlY3JldCIgfQo=
  quote.txt: VmFuaXRhcyB2YW5pdGF0dW0gZXQgb21uaWEgdmFuaXRhcwo=
kind: Secret
metadata:
  creationTimestamp: "2020-02-26T21:55:54Z"
  name: my-application-secrets
  namespace: download-demo
  resourceVersion: "13263"
  selfLink: /api/v1/namespaces/download-demo/secrets/my-application-secrets
  uid: 1acbc325-369c-4adc-bce7-93f927415d0f
type: Opaque
```

You can see some metadata, type, kind, apiVersion, but what is really important for this post is the data. You can copy any of those two values and decode it with base64.

{% highlight shell %}
echo "eyAibWVzc2FnZSI6ICJ2ZXJ5IHNlY3JldCIgfQo=" | base64 --decode
{% endhighlight %}

You should get:
```
{ "message": "very secret" }
```

This is exactly what we had in our file `config.json`. If you want to access one specific value you can use jsonpath and specify a path to your property as a template. .data reference to our data level and gives you a map. If we want to see specific property inside, we have to specify our key, because `.` is a separator, if it's part of a key we have to escape it with `\` so our file `config.json` is represented as `config\.json`. Try it for yourself.

{% highlight shell %}
kubectl --namespace download-demo \
get secrets my-application-secrets \
-o jsonpath \
--template '{.data.config\.json}'
{% endhighlight %}

That will give you value we just saw encoded in base64, you can pipe it to see the content.

{% highlight shell %}
kubectl --namespace download-demo \
get secrets my-application-secrets \
-o jsonpath \
--template '{.data.config\.json}' |\
base64 --decode
{% endhighlight %}

While this works, it is quite manual, you have to check all properties first. I wanted something automatic, ideally going through properties and creating the same files on my local machine. I wrote a script which gets secrets as json ( -o json ), it let me process it with jq, which results with text in line per file in a format `file_name:content`

Run this command if you want to see what it does.
{% highlight shell %}
kubectl --namespace download-demo \
get secrets my-application-secrets \
-o json |\
jq -r '.data | keys[] as $k | "\($k):\(.[$k])"'
{% endhighlight %}

```
config.json:eyAibWVzc2FnZSI6ICJ2ZXJ5IHNlY3JldCIgfQo=
quote.txt:VmFuaXRhcyB2YW5pdGF0dW0gZXQgb21uaWEgdmFuaXRhcwo=
```

Now we have to go line by line and create files, you can see full script below.

## Full download secrets from Kubernetes script

{% highlight bash %}
#!/bin/sh

namespace=$1
secret_name=$2
directory_to_save=${3-$2}

mkdir -p $directory_to_save

values=$(kubectl --namespace $namespace get secrets $secret_name -o json \
    | jq -r '.data | keys[] as $k | "\($k):\(.[$k])"')
for file_content in $values
do
    file_name=$(echo $file_content | cut -d':' -f1)
    echo $file_content | cut -d':' -f2 | base64 --decode > $directory_to_save/$file_name
    echo "Created $directory_to_save/$file_name"
done
{% endhighlight %}

To run this script you have to specify your namespace (I have it hardcoded in my script, as I always use same namespace), then your secret name and optional directory where you want to save files, if it’s not specified the script will use directory with same name as secrets.

{% highlight shell %}
./get_secrets.sh download-demo my-application-secrets
{% endhighlight %}

You can now see that files have same content like our original files

{% highlight shell %}
cat my-application-secrets/config.json
{% endhighlight %}

Output:
```
{ "message": "very secret" }
```

{% highlight shell %}
cat my-application-secrets/quote.txt
{% endhighlight %}

Output:
```
Vanitas vanitatum et omnia vanitas
```

In this blog post we looked at displaying information about secrets, how we can use some different outputs, there is a lot more you can do, so see what’s out there. Final part was about parsing the output, so we can generate files based on our kubernetes secrets.
