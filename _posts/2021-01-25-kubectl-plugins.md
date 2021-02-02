---
layout: posts
title: Kubectl Plugins
author: Michal Stovicek
---
## What are kubectl plugins

The primary tool for controlling Kubernetes is (as I am sure you all know) `kubectl`. The `kubectl` client tool provides a command line way how to interact with your cluster to get information from it as well as to create the Kubernetes resources such as **deployments**, **services**, **ingres**, etc.

Although the `kubectl` has many capabilities, there is always a space for improvements and there are different needs of administrators and developers who are working with Kubernetes on a daily basis. And so here comes `kubectl` plugin system. Plugins can be written in any language which can be used to write a command line application. It can be BASH, PowerShell, but also Python, Javascript, GO or even Java if you want (even though I don't personally think that Java is a good option for command line apps - that's my opinion only (-: ). Such a plugin will extend the functionality of `kubectl`.

## Installing plugins

The easiest way of plugins installation is to use a tool called `krew` (you can [download Krew here](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)). This tool will allow you to search for plugins and to install them as well.

You may also want to use a plugin which is not part of the **krew** repository (or you may not want to use krew at all). In such situation, you may install the plugin manually. What you need to do is to move your plugin to any directory which is in your `PATH` variable and make it executable. The plugins' name must start with `kubectl-` followed by an arbitrary number of words composing the name where all the words must be delimited by `-`. The last condition is that the plugin file must be executable so that `kubectl` can invoke it. Once that's done, you can use your `kubectl` plugin! Easy enough, right?

## Writing a custom deployment scale plugin

Enough talking, let's dive into little bit of coding! I wrote a simple `kubectl` plugin available in my [GitHub repository](https://github.com/misastovicek/kubectl-groupscale) which scales Kubernetes **deployment** resources to a specified number of Pods. This is similar to what `kubectl scale` does, but this plugin scales all deployments across all the namespaces which have a specified label set. Consider this example:

Three deployments of Nginx exist in a cluster:

* **nginx-ingress** in namespace **ingress** has 1 replica
* **nginx-1** in namespace **app-1** has 2 replicas
* **nginx-2** in namespace **app-2** has 5 replicas

Now all of the deployment resources have a label `scale: yes` set and I want all of those Nginx deployments to have 2 replicas. I can run `kubectl groupscale -replicas 2 -label "scale=yes"` and all of the deployments will be scaled to two replicas in a single command.

So, let's break down the code from the repository mentioned above. Since this is a CLI app, I am using `flag` package from the GO standard library which is awesome in my opinion and very easy to use (as [Todd MacLeod](https://github.com/GoesToEleven) says - GO is about ease of programming (-; ):

```go
package main

import (
	"flag"
	"os"
	"path/filepath"
	"strings"

	groupscale "github.com/misastovicek/kubectl-groupscale/cmd"

	"k8s.io/client-go/util/homedir"
)

func main() {
    var kubeconfig *string
    if home := homedir.HomeDir(); home != "" {
    	kubeconfig = flag.String(
    		"kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
    } else {
    	kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
    }

    replicas := flag.Int("replicas", 1, "(optional) number of replicas for each matched deployment")
    label := flag.String("label", "", "Key/Value pair used to identify the kubernetes applications whichshould be scaled")
    flag.Parse()

    if flag.NFlag() == 0 {
    	flag.PrintDefaults()
    	os.Exit(0)
    }

    xLabel := strings.SplitN(*label, "=", 2)
	labelKey := xLabel[0]
	labelValue := xLabel[1]

	groupscale.GroupScale(kubeconfig, labelKey, labelValue, *replicas)
}
```

The code part above does a single thing - it defines named arguments for the app. Since the `flag` methods return pointers on a type I have defined a `kubeconfig` variable of type pointer to string which then contains memory address which holds the actual data (the path to the kubeconfig file in this case). Since GO compiler is clever enough, type does not need to be specified for the other flags since those are directly assigned and then I just run `flag.Prse()`, which does the actual parsing of input parameters so that they can be used throughout the rest of the program. I am also checking if number of flags is equal to zero in which case the program prints `help` (notice the `flag.PringDefaults()`) and exits.

Alright, so now the command line arguments are parsed and I can go ahead and call the `GroupScale` function from the package `groupscale`. But before I do that, the label is being split into **key** and **value** so that it is easier to work with them later. Of course, this can be handled in many different ways. Notice here that I am passing `kubeconfig` which is pointer to a string, `labelKey` and `labelValue` which are strings and I am **dereferencing pointer to `replicas`** because the `GroupScale` function takes type `int` - not `*int`.

Next step is the actual logic of scaling. First of all, this logic is in a separated GO package named `groupscale`. Let's see the code:

```go
package groupscale

import (
	"context"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

type deploymentRelease struct {
	name      string
	namespace string
}

func (d *deploymentRelease) scaleDeployment(clientset *kubernetes.Clientset, scaleTo int32) {
	scale, err := clientset.AppsV1().Deployments(d.namespace).GetScale(
		context.TODO(), d.name, metav1.GetOptions{})
	genericErrorHandler(err)
	scale.Spec.Replicas = scaleTo
	_, err = clientset.AppsV1().Deployments(d.namespace).UpdateScale(
		context.TODO(), d.name, scale, metav1.UpdateOptions{})
	genericErrorHandler(err)
}

func genericErrorHandler(err error) {
	if err != nil {
		panic(err.Error())
	}
}

func scaleLabelExists(mResourceLables map[string]string, labelKey string, labelValue string) bool {
	if val, ok := mResourceLables[labelKey]; ok && val == labelValue {
		return true
	}
	return false
}

// GroupScale scales Kubernetes deployment based on label key/value pair to number of replicas
func GroupScale(kubeconfigPath *string, labelKey string, labelValue string, replicas int) {
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfigPath)
	genericErrorHandler(err)

	clientset, err := kubernetes.NewForConfig(config)
	genericErrorHandler(err)

	deployments, err := clientset.AppsV1().Deployments("").List(context.TODO(), metav1.ListOptions{})
	genericErrorHandler(err)

	var deploymentsToScale []deploymentRelease

	for _, deployment := range deployments.Items {
		if scaleLabelExists(deployment.Labels, labelKey, labelValue) {
			deploymentsToScale = append(deploymentsToScale, deploymentRelease{
				name:      deployment.Name,
				namespace: deployment.Namespace,
			})
		}
	}
	for _, deployment := range deploymentsToScale {
		deployment.scaleDeployment(clientset, int32(replicas))
	}
}
```

The code above defines a simple structure `deploymentRelease` (which doesn't have a good name by the way (-: ) and that structure has a method associated to it (the method has receiver pointer to deploymentRelease). The method does not have to have pointer receiver in this case since it does not modify the struct data in any way. The most interesting part here is the `GroupScale` function which is exported and so it is available to use by other GO packages. The function takes four arguments which I have described before at the function call and it builds a client to a Kubernetes cluster pointed to by `kubeconfigPath`. Now that it has connection to the cluster, it can go ahead and do the actual scaling. You may notice the `genericErrorHandler` function which I have defined so that I don't need to always write the `if err != nil` pattern which is how you usually handle errors in GO.

The main stuff here is that for each deployment the program checks whether the specified label exists and appends a struct (defined at the top) to a slice of such structs. After this is done the program just iterates through the deployments which should be scaled and scales them to the specified number of replicas. Once done, you can check the number of replicas with `kubectl get deploy --all-namespaces`. All deployments which have the specified label should be scaled to the number of replicas you passed to the plugin.

Let's see it in action now. I am running **Docker for desktop** with Kubernetes enabled, so I have a single node Kubernetes running on my system. First of all, I'll create couple of namespaces for the testing deployments:

```bash
$ for ns in `seq 1 3`; do kubectl create ns test-$ns; done
namespace/test-1 created
namespace/test-2 created
namespace/test-3 created
```

Now that I have namespaces let's prepare create three deployments. Deployment deffinition is as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-1 # app-2 and app-3 for the other two deployments
  namespace: test-1 # test-2 and test-3 for other two deployments
  labels:
    scale: "yes" # used by the groupscale plugin to match deployments
spec:
  replicas: 1 # 3 for second deployment, 5 for third deployment
  selector:
    matchLabels:
      app: app-1 # app-2 and app-3 for the other two deployments
  template:
    metadata:
      labels:
        app: app-1 # app-2 and app-3 for the other two deployments
    spec:
      containers:
      - name: app-1 # app-2 and app-3 for the other two deployments
        image: nginx:latest
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

I have crated three files with the content from the above YAML deffinition to make it easy to deploy. Alright, we have the deffinition and can contine to deployment:

```bash
$ for f in `ls -1 | grep "^deploy-"`; do echo kubectl apply -f $f; done
deployment.apps/app-1 created
deployment.apps/app-2 created
deployment.apps/app-3 created

$ kubectl get deploy --all-namespaces
NAMESPACE     NAME      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   coredns   2/2     2            2           6d
test-1        app-1     1/1     1            1           45s
test-2        app-2     3/3     3            3           45s
test-3        app-3     4/4     4            4           44s
```

From the above output we can see that **app-1** has 1 replica, **app-2** has 3 replicas and **app-3** has 4 replicas. Now we'll use the **groupscale** plugin:

```bash
$ kubectl groupscale -replicas 2 -label "scale=yes"

$ kubectl get deploy --all-namespaces
NAMESPACE     NAME      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   coredns   2/2     2            2           6d
test-1        app-1     2/2     2            2           4m2s
test-2        app-2     2/2     2            2           4m2s
test-3        app-3     2/2     2            2           4m1s
```

And here we go! all three deployments are scaled to 2 replicas as stated by the **groupscale** plugin! Easy enough, right?
Now you know it is pretty easy to write a custom plugin. Of course you don't need to use GO, but you've got the idea how to extend your `kubectl` now!
