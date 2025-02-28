# Cloud Native Canary Deployment Strategy using Argo Rollouts

## Introduction

One important topic in the `Cloud Native` is the `Microservice Architecture`. We are not any more dealing with one monolithic application. We have several applications that have dependencies on each other and also have other dependencies like brokers or databases.
 
Applications have their own life cycle, so we should be able to execute independent canary deployment. All the applications and dependencies will not change their version at the same time.
 
Another important topic in the `Cloud Native` is `Continuous Delivery`. If we are going to have several applications doing canary deployment independently we have to automate it. We will use **Helm**, **Argo Rollouts**, **Openshift GitOps**, and of course **Red Hat Openshift** to help us.

[**Argo Rollouts**](https://argoproj.github.io/argo-rollouts/) is a Kubernetes controller and set of CRDs which provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.
In this demo we are going to use canary capabilities.
 
**In the next steps we will see a real example of how to install, deploy and manage the life cycle of Cloud Native applications doing canary deployment using Argo Rollouts.**

Let's start with some theory...after it, we will have the **hands-on example**.

## Canary Deployment

A canary deployment is a strategy where the operator releases a new version of their application to a small percentage of the production traffic. This small percentage may test the new version and provide feedback. If the new version is working well the operator may increase the percentage, till all the traffic is using the new version. Unlike Blue/Green, canary deployments are smoother, and failures have limited impact. 

## Shop application
 
We are going to use very simple applications to test canary deployment. We have created two Quarkus applications `Products` and `Discounts`
 
![Shop Application](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/Shop.png)
 
`Products` call `Discounts` to get the product`s discount and expose an API with a list of products with its discounts.
 
## Shop Canary
 
To achieve canary deployment with `Cloud Native` applications using **Argo Rollouts**, we have designed this architecture.

![Shop initial status](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/canary-rollout-initial.png)
 
OpenShift Components - Online

- Routes and Services declared with the suffix -online
- Routes mapped only to the online services
- Services mapped to the rollout.

In Blue/Green deployment we always have an offline service to test the version that is not in production. In the case of canary deployment we do not need it because progressively we will have the new version in production. 


We have defined an active or online service 'products-umbrella-online'. Final user will always use 'products-umbrella-online'. When a new version is deployed **Argo Rollouts** create a new revision (ReplicaSet). The number of replicas in the new release increases based on the information in the steps, the number of replicas in the old release decreases in the same number. We have configured a pause duration between each step. To learn more about **Argo Rollouts**, please read [this](https://argoproj.github.io/argo-rollouts/features/canary/).


## Shop Umbrella Helm Chart
 
One of the best ways to package `Cloud Native` applications is `Helm`. In canary deployment it makes even more sense.
We have created a chart for each application that does not know anything about canary. Then we pack everything together in an umbrella helm chart.

![Shop Umbrella Helm Chart](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/Shop-helm-canary-rollouts.png)

In the `Shop Umbrella Chart` we use several times the same charts as helm dependencies but with different names.
 
We have packaged both applications in one chart, but we may have different umbrella charts per application.

## Demo!!

### Prerequisites:

- **Red Hat Openshift 4.10** with admin rights. 
   - You can download [Red Hat Openshift Local for OCP 4.10](https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/2.6.0). 
   - [Getting Started Guide](https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.5/html/getting_started_guide/using_gsg)
- [Git](https://git-scm.com/)
- [GitHub account](https://github.com/)
- [oc 4.10 CLI](https://docs.openshift.com/container-platform/4.10/cli_reference/openshift_cli/getting-started-cli.html)
- [Argo Rollouts CLI](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation )

We have a GitHub [repository](https://github.com/davidseve/cloud-native-deployment-strategies) for this demo. As part of the demo, you will have to do some changes and commits. So **it is important that you fork the repository and clone it in your local**.

```
git clone https://github.com/your_user/cloud-native-deployment-strategies
```

If we want to have a `Cloud Native` deployment we can not forget `CI/CD`. **Red Hat OpenShift GitOps** will help us.
 
### Install OpenShift GitOps
 
Go to the folder where you have cloned your forked repository and create a new branch `canary`
```
git checkout -b canary
git push origin canary
```
 
Log into OpenShift as a cluster admin and install the OpenShift GitOps operator with the following command. This may take some minutes.
```
oc apply -f gitops/gitops-operator.yaml
```
 
Once OpenShift GitOps is installed, an instance of Argo CD is automatically installed on the cluster in the `openshift-gitops` namespace and a link to this instance is added to the application launcher in OpenShift Web Console.
 
![Application Launcher](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/gitops-link.png)
 
### Log into Argo CD dashboard
 
Argo CD upon installation generates an initial admin password which is stored in a Kubernetes secret. In order to retrieve this password, run the following command to decrypt the admin password:
 
```
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```
 
Click on Argo CD from the OpenShift Web Console application launcher and then log into Argo CD with `admin` username and the password retrieved from the previous step.
 
![Argo CD](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/ArgoCD-login.png)
 
![Argo CD](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/ArgoCD-UI.png)
 
### Configure OpenShift with Argo CD
 
We are going to follow, as much as we can, a GitOps methodology in this demo. So we will have everything in our Git repository and use **ArgoCD** to deploy it in the cluster.
 
In the current Git repository, the [gitops/cluster-config](https://github.com/davidseve/cloud-native-deployment-strategies/tree/main/gitops/cluster-config) directory contains OpenShift cluster configurations such as:

- namespaces `gitops`.
- role binding for ArgoCD to the namespace `gitops`.
- Argo Rollouts project.
 
Let's configure Argo CD to recursively sync the content of the [gitops/cluster-config](https://github.com/davidseve/cloud-native-deployment-strategies/tree/main/gitops/cluster-config) directory into the OpenShift cluster.
 
Execute this command to add a new Argo CD application that syncs a Git repository containing cluster configurations with the OpenShift cluster.
 
```
oc apply -f canary-argo-rollouts/application-cluster-config.yaml
```
 
Looking at the Argo CD dashboard, you would notice that an application has been created.

You can click on the `cluster-configuration` application to check the details of sync resources and their status on the cluster.

### Create Shop application

We are going to create the application `shop`, that we will use to test canary deployment. Because we will make changes in the application's GitHub repository, we have to use the repository that you have just forked. Please edit the file `canary-argo-rollouts/application-shop-canary-rollouts.yaml` and set your own GitHub repository in the `reportURL`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop
  namespace: openshift-gitops
spec:
  destination:
    name: ''
    namespace: gitops
    server: 'https://kubernetes.default.svc'
  source:
    path: helm/quarkus-helm-umbrella/chart
    repoURL:  https://github.com/change_me/cloud-native-deployment-strategies.git
    targetRevision: canary
    helm:
      parameters:
      - name: "global.namespace"
        value: gitops
      valueFiles:
        - values/values-canary-rollouts.yaml
  project: default
  syncPolicy:
    automated:
      prune: false
      selfHeal: false

```

```
oc apply -f canary-argo-rollouts/application-shop-canary-rollouts.yaml
```

Looking at the Argo CD dashboard, you would notice that we have a new `shop` application.

![Argo CD - Cluster Config](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/ArgoCD-Applications.png)

## Test Shop application
 
We have deployed the `shop` with ArgoCD. We can test that it is up and running.
 
We have to get the Online route
```
curl "$(oc get routes products-umbrella-online -n gitops --template='https://{{.spec.host}}')/products"
```

Notice that in each microservice response we have added metadata information to see better the `version` of each application. This will help us to see the changes while we do the canary deployment.
We can see that the current version is `v1.0.1`:
```json
{
   "products":[
      {
         ...
         "name":"TV 4K",
         "price":"1500€"
      }
   ],
   "metadata":{
      "version":"v1.0.1", <--
      "colour":"none",
      "mode":"online"
   }
}
```

We can also see the rollout`s status[^note].

[^note]:
    Argo Rollouts offers a Kubectl plugin to enrich the experience with Rollouts https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation 

```
kubectl argo rollouts get rollout products --watch -n gitops
```

```
NAME                                  KIND        STATUS     AGE  INFO
⟳ products                            Rollout     ✔ Healthy  38s  
└──# revision:1                                                   
   └──⧉ products-67fc9fb79b           ReplicaSet  ✔ Healthy  38s  stable
      ├──□ products-67fc9fb79b-4ql4z  Pod         ✔ Running  38s  ready:1/1
      ├──□ products-67fc9fb79b-7c4jw  Pod         ✔ Running  38s  ready:1/1
      ├──□ products-67fc9fb79b-lz86j  Pod         ✔ Running  38s  ready:1/1
      └──□ products-67fc9fb79b-xlkhp  Pod         ✔ Running  38s  ready:1/1
```

 
## Products Canary deployment
 
We have already deployed the products version v1.0.1 with 4 replicas, and we are ready to use a new products version v1.1.1 that has a new `description` attribute.

This is our current status:
![Shop initial status](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/canary-rollout-step-0.png)

This is how we have configure **Argo Rollouts** for this demo:
```yaml
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause:
            duration: 30s
        - setWeight: 50
        - pause:
            duration: 30s
```

We have split a `Cloud Native` Canary deployment into three automatic step:

1. Deploy canary version for 10%
2. Scale canary version to 50%
3. Scale canary version to 100%

This is just an example. The key point here is that, very easily we can have the canary deployment that better fits our needs. In order to make this demo faster we have not set a pause with out duration in any step, so  **Argo Rollouts** will go throw each step automatically.
### Step 1 - Deploy canary version for 10%
 
We will deploy a new version v1.1.1. To do it, we have to edit the file `helm/quarkus-helm-umbrella/chart/values/values-canary-rollouts.yaml` under `products-blue` set `tag` value to `v.1.1.1`

```yaml
products-blue:
  mode: online
  image:
    tag: v1.1.1
```

**Argo Rollouts** will automatically deploy a new products revision. The canary version will be 10% of the replicas. In this demo we are no using [traffic management](https://argoproj.github.io/argo-rollouts/features/traffic-management/). **Argo Rollouts** makes a best effort attempt to achieve the percentage listed in the last setWeight step between the new and old version. This means that it will create only one replica in the new revision, because it is rounded up. All the requests are load balanced between the old and the new replicas.

Push the changes to start the deployment.
```
git add .
git commit -m "Change products version to v1.1.1"
git push origin canary
```

 ArgoCD will refresh the status after some minutes. If you don't want to wait you can refresh it manually from ArgoCD UI or configure the Argo CD Git Webhook.[^note2].
 
[^note2]:
    Here you can see how to configure the Argo CD Git [Webhook]( https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/)
    ![Argo CD Git Webhook](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/ArgoCD-webhook.png)

![Refresh Shop](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/ArgoCD-Shop-Refresh.png)

This is our current status:
![Shop Step 1](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/canary-rollout-step-1.png)

```
kubectl argo rollouts get rollout products --watch -n gitops
```
```
NAME                                  KIND        STATUS     AGE    INFO
⟳ products                            Rollout     ॥ Paused   3m13s  
├──# revision:2                                                     
│  └──⧉ products-9dc6f576f            ReplicaSet  ✔ Healthy  8s     canary
│     └──□ products-9dc6f576f-fwq8m   Pod         ✔ Running  8s     ready:1/1
└──# revision:1                                                     
   └──⧉ products-67fc9fb79b           ReplicaSet  ✔ Healthy  3m13s  stable
      ├──□ products-67fc9fb79b-4ql4z  Pod         ✔ Running  3m13s  ready:1/1
      ├──□ products-67fc9fb79b-lz86j  Pod         ✔ Running  3m13s  ready:1/1
      └──□ products-67fc9fb79b-xlkhp  Pod         ✔ Running  3m13s  ready:1/1
```

In the products url`s response you will have the new version in 25% of the requests.

New revision:
```json
{
  "products":[
     {
        "discountInfo":{...},
        "name":"TV 4K",
        "price":"1500€",
        "description":"The best TV" <--
     }
  ],
  "metadata":{
     "version":"v1.1.1", <--
  }
}
```

Old revision:
```json
{
  "products":[
     {
        "discountInfo":{...},
        "name":"TV 4K",
        "price":"1500€"
     }
  ],
  "metadata":{
     "version":"v1.0.1", <--
  }
}
```

### Step 2 - Scale canary version to 50%
After 30 seconds **Argo Rollouts** automatically will increase the number of replicas in the new release to 2. Instead of increase automatically after 30 seconds we can configure **Argo Rollouts** to wait indefinitely until that `Pause` condition is removed. But this is not part of this demo.
This is our current status:
![Shop Step 2](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/canary-rollout-step-2.png)

```
kubectl argo rollouts get rollout products --watch -n gitops
```
```
NAME                                  KIND        STATUS     AGE    INFO
⟳ products                            Rollout     ॥ Paused   3m47s  
├──# revision:2                                                     
│  └──⧉ products-9dc6f576f            ReplicaSet  ✔ Healthy  42s    canary
│     ├──□ products-9dc6f576f-fwq8m   Pod         ✔ Running  42s    ready:1/1
│     └──□ products-9dc6f576f-8qppq   Pod         ✔ Running  6s     ready:1/1
└──# revision:1                                                     
   └──⧉ products-67fc9fb79b           ReplicaSet  ✔ Healthy  3m47s  stable
      ├──□ products-67fc9fb79b-lz86j  Pod         ✔ Running  3m47s  ready:1/1
      └──□ products-67fc9fb79b-xlkhp  Pod         ✔ Running  3m47s  ready:1/1
```

### Step 3 - Scale canary version to 100%
After other 30 seconds **Argo Rollouts** will increase the number of replicas in the new release to 4 and scale down the old revision.

This is our current status:
![Shop Step 1](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/canary-rollout-step-3.png)

```
kubectl argo rollouts get rollout products --watch -n gitops
```
```
NAME                                 KIND        STATUS        AGE    INFO
⟳ products                           Rollout     ✔ Healthy     4m32s  
├──# revision:2                                                       
│  └──⧉ products-9dc6f576f           ReplicaSet  ✔ Healthy     87s    stable
│     ├──□ products-9dc6f576f-fwq8m  Pod         ✔ Running     87s    ready:1/1
│     ├──□ products-9dc6f576f-8qppq  Pod         ✔ Running     51s    ready:1/1
│     ├──□ products-9dc6f576f-5ch92  Pod         ✔ Running     17s    ready:1/1
│     └──□ products-9dc6f576f-kmvdh  Pod         ✔ Running     17s    ready:1/1
└──# revision:1                                                       
   └──⧉ products-67fc9fb79b          ReplicaSet  • ScaledDown  4m32s  
```

**We have in the online environment the new version v1.1.1!!!**
```json
{
  "products":[
     {
        "discountInfo":{...},
        "name":"TV 4K",
        "price":"1500€",
        "description":"The best TV" <--
     }
  ],
  "metadata":{
     "version":"v1.1.1", <--
  }
}
```

### Rollback

Imagine that something goes wrong, we know that this never happens but just in case. We can do a very `quick rollback` just undoing the change.

**Argo Rollouts** has an [undo](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_undo/) command to do the rollback. In my opinion, I don't like this procedure because it is not aligned with GitOps. The changes that **Argo Rollouts** do does not come from git, so git is OutOfSync with what we have in Openshift.
In our case the commit that we have done not only changes the ReplicaSet but also the ConfigMap. The `undo` command only changes the ReplicaSet, so it does not work for us.

I recommend doing the changes in git. We will revert the last commit
```
git revert HEAD --no-edit
```

If we just revert the changes in git we will go back to the previous version. But **Argo Rollouts** will take this revert as a new release so it will do it throw the steps that we have configured. We want a `quick rollback` we don't want a step by step revert. To achieved the `quick rollback` we will configure **Argo Rollouts** without steps for the rollback.

Because we have our **Argo Rollouts** configuration as values in our Helm Chart, we have just to edit the values.yaml that we are using.

In the file `helm/quarkus-helm-umbrella/chart/values/values-canary-rollouts.yaml` under `products-blue` under the `steps` delete all the steps and only set one step `- setWeight: 100`

`helm/quarkus-helm-umbrella/chart/values/values-canary-rollouts.yaml` should looks like:
```yaml
products-blue:
  mode: online
  image:
    tag: v1.0.1
  version: none
  replicaCount: 4
  fullnameOverride: "products"
  rollouts:
    enabled: true
    canary:
      steps:
        - setWeight: 100
```

Execute those command to push the changes:
```
git add .
git commit -m "delete steps for rollback"
git push origin canary
```
**ArgoCD** will get the changes and apply them. **Argo Rollouts** will create a new revision with the previous version.

The rollback is done!
![Shop Step Rollback](https://github.com/davidseve/cloud-native-deployment-strategies/raw/main/images/canary-rollout-step-Rollback.png)
```json
{
  "products":[
     {
        "discountInfo":{...},
        "name":"TV 4K",
        "price":"1500€"
     }
  ],
  "metadata":{
     "version":"v1.0.1", <--
  }
}
```
To get the application ready for a new release we should configure again the  **Argo Rollouts** with the steps.

## Delete environment
 
To delete all the things that we have done for the demo you have to:

- In GitHub delete the branch `canary`
- In ArgoCD delete the application `cluster-configuration` and `shop`
- In Openshift, go to project `openshift-operators` and delete the installed operators **Openshift GitOps**.



