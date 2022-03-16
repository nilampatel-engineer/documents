# Argo Rollouts

Vendor website: [argoproj.github.io/argo-rollouts](https://argoproj.github.io/argo-rollouts/)

A `Rollout` resource is designed to be a direct replacement for a `Deployment`.
The yaml syntax for the pod `template:` should be identical, but there's an
additional `strategy:` block that the Argo Rollouts controller uses.

Essentially, you define two Services (and optionally Gateways if you want to
internet access) and one Rollout.  When a new version of the Rollout yaml is
applied (eg pointing to a new image), the controller creates a new replicaSet
using the canaryService, allowing you to test it, then it progressively directs
more and more traffic for the stableService to the new replicaSet.  Finally
both canaryService and stableService are directing 100% of their traffic to
the new replicaSet and the old one is deleted.

## Controller installation

* Create the `argo-rollouts` namespace, where the controller will be installed.   
[namespace-argo-rollouts.yaml](https://github.com/fetchai/infra-testnet-v2-deployment/blob/master/manifests/common/namespace-argo-rollouts.yaml)

* Add the vendor helm repo to the list (we don't have a local mirror)  
[00-repos.yaml](https://github.com/fetchai/infra-testnet-v2-deployment/blob/739df187f54dd0a11e4a3d5116126a0549a455df/source/helmfile.d/00-repos.yaml#L20)

* Add a Values file (likely identical in all environments)  
[values-argo-rollouts.yaml](https://github.com/fetchai/infra-testnet-v2-deployment/blob/master/source/values/values-argo-rollouts.yaml)

* Add the helmfile  
[23-argo-rollouts.yaml](https://github.com/fetchai/infra-testnet-v2-deployment/blob/master/source/helmfile.d/23-argo-rollouts.yaml)

* Example PRs  
sandbox  
testnet-v2  
[mainnet-v2](https://github.com/fetchai/infra-mainnet-v2-deployment/pull/25/commits/2fe776e8c3363db409bdcccde983cc0f3771c712) (NB this PR also includes external-secrets)

## Writing / converting helm charts to use Canary Rollouts

* Create *two* Services for your application, not the usual one.  Name them something like `...-stable` and `...-canary`.  
[services.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/services.yaml)  
<sup>(Before: [istio-http.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/0fc523cd7773ca3aa66c75bf20001c7ae7e10b17/charts/stargate-blockexplorer/1.1.0/templates/istio-http.yaml#L72) )</sup>

* Assuming you want both stable and canary services to be accessible without port-forwarding,
create two Gateways.  (Technically, if you don't want to expose the canary service, the preview gateway is optional,
but is recommended for easy access to canary versions of the website, even if you hide it behind a
Fetch.ai-only IngressGateway)  
[gateway.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/gateway.yaml)  
<sup>(Before:  [istio-http.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/0fc523cd7773ca3aa66c75bf20001c7ae7e10b17/charts/stargate-blockexplorer/1.1.0/templates/istio-http.yaml#L15) )</sup>  

* And create DNS end points / Certificates for each Gateway.  
[dnsendpoint.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/dnsendpoint.yaml)  
[cert.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/cert.yaml)

* Create a VirtualService for each of your Gateways.  The public-facing VSC should include both
-stable and -canary services, and the initial weights should send all traffic to -stable.  The
preview gateway should only include the canary service.  (If you want, you could create a third
[gateway, dnsendpoint, cert, virtualservice] set which only points to the -stable service, for
ease of comparison, but this was considered overkill for the stargate-blockexplorer. YMMV!)  
[virtualservice.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/virtualservice.yaml)  
<sup>(Before: [istio-http.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/0fc523cd7773ca3aa66c75bf20001c7ae7e10b17/charts/stargate-blockexplorer/1.1.0/templates/istio-http.yaml#L51) )</sup>  

* Create an `argoproj.io/v1alpha1/Rollout` (where previously you would have used an `apps/v1/Deployment`)  
[rollout.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/rollout.yaml)  
<sup>(Before: [deployment.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/0fc523cd7773ca3aa66c75bf20001c7ae7e10b17/charts/stargate-blockexplorer/1.1.0/templates/_deployment.yaml#L6))</sup>  
The `.spec.template` block will be identical to that used in a Deployment < [docs](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec) >  
The rollout strategy is described in the `.spec.strategy` block.  There are two strategies available: [BlueGreen](https://argoproj.github.io/argo-rollouts/features/bluegreen/) and [Canary](https://argoproj.github.io/argo-rollouts/features/canary/).  Here we consider the Canary strategy, which was used for the stargate-blockexplorer.  
The strategy needs to be directed to the two Services created above.  Also, the istio VirtualService, whose parameters
the Argo Rollouts controller will modify at runtime to control the traffic management, is referenced here.  
With the traffic management plumbing in place, the `spec.strategy.canary.steps` block
defines each step that a deployment will take on its way to stable.  A `setWeight` parameter defines the
percentage of traffic the new version will receive.  A `setCanaryScale` ensures a particular number of replicas.
A `pause` waits a specified period of time.  
At the time of writing only the above steps have been tried,
however a major feature of Argo Rollouts is the ability to [automatically decide](https://argoproj.github.io/argo-rollouts/features/analysis/)
whether to progress, or rollback, a new deployment, based on metrics received from eg [prometheus](https://argoproj.github.io/argo-rollouts/analysis/prometheus/).  This is definitely an area to pursue!

* Create an HPA, if required.  
[hpa.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.3.3/templates/hpa.yaml)  
<sup>(Before: [hpa.yaml](https://github.com/fetchai/infra-helm-repo-private/blob/master/charts/stargate-blockexplorer/1.1.0/templates/hpa.yaml))</sup>  
Being essentially just a subclass of Deployment, Rollouts also support Horizontal Pod Autoscalers.  Simply refer
to the correct resource in the `scaleTargetRef` field.  

## Rollout notifications in Slack

The setup is out of scope for this document, but note that it is usual to configure Argo Rollouts with a reference to
Argo Notifications, so that messages are sent to a particular Slack channel whenever a rollout progresses, or otherwise
changes state.  Please contact Amit / Ian if further documentation of how to set this up is required.

## Using Rollouts in practice

### Initial deployment

Just apply the manifests as usual.  A single replicaset will be created, and all traffic for both stable and canary
services directed to it.  (To avoid transient annoyances, please do ensure that the initial weights on the VirtualService
are set correctly, and that the replicas field in the Rollout spec is no lower than the minReplicas field in the
HPA (or indeed higher than maxReplicas)).

### Deploying a new version

** Please note that Argo Rollouts is primarily designed to handle simple everyday changes, such as bumping up the
tag version on an image.  Handling other changes to the Rollout yaml, such as changing CPU resources, should also
be fine.  But if you're wanting to update other resources, eg the values in a ConfigMap, then you will need
to jump through hoops (eg to ensure that multiple versions of the ConfigMap exist for the duration of the rollout,
then create a further change down the road to remove the old versions).  On the spectrum between "bump the
blockexplorer image from v0.4.5 to v0.4.6" and "move mainnet to the new version of cosmos", Argo Rollouts is
very much designed for the former!

* Apply the new version of your rendered rollout manifest, ideally using ArgoCD.  
(or to cheat, for testing purposes only...)  
`kubectl argo rollouts set image blockexplorer blockexplorer=gcr.io/fetch-ai-images/cosmos-explorer:v0.4.6`


### Monitor the progress of a rollout

<details><summary markdown="span"><sub>kubectl argo plugin installation</sub></summary>

If the command below fails with an "unknown command" error, follow the
[installation instructions](https://argoproj.github.io/argo-rollouts/installation/) to install the plugin.
(NB take care to remove the architecture suffix from the downloaded filename).

</details>

```bash
kubectl argo rollouts list rollouts
kubectl argo rollouts get rollout blockexplorer
```

You should see both images listed, with (stable) and (canary) beside the appropriate image.  
The Strategy output
will indicate which Step of the rollout you are on, with Status indicating whether the rollout is Progressing,
Unhealthy or Healthy.  
At the bottom you will see the ReplicaSets that are running, together with a list of
pods.  (Unless overridden in the Step yaml, the HPAs of the canary will try to match those of stable).  

### Hurry a rollout

The canary rollout likely includes long pauses between each step, to give time for eg i) functional testing to
be carried out before any public access is provided; ii) performance testing to be carried out before significant
traffic is applied; iii) scaleability and load testing to be carried out before full-volume traffic is applied.

If you are in a bit of a hurry, the following command will bump the deployment on to the next step...

```bash
kubectl argo rollouts promote blockexplorer
```

If you're in a real hurry, run the following to immediately move all traffic to the new version...  
(It will wait until the new pods are ready, of course, so no need to worry about downtime / capacity issues).

```bash
kubectl argo rollouts promote blockexplorer --full
```

(NB this is good if the stable version is currently broken, but if you're habitually using --full you
should question why you've chosen the overhead of a Rollout vs just using a Deployment).

### Roll back a failed rollout

If at any time you need to go back to the previous version, run the following...

```bash
kubectl argo rollouts undo blockexplorer
```

NB this creates a new revision (using the previous replicaset, if available... depending on how far the
rollout progressed, the old pods may or may not be still be around to be reused).  Think of this as
doing a controlled roll-forward, but to the previous image tag (eg if a minor bug has been discovered).

### Abort / pause / retry a failing rollout

If bad things are happening, you can abort the canary rollout...

```bash
kubectl argo rollouts abort blockexplorer
```

...or if you're not sure...

```bash
kubectl argo rollouts pause blockexplorer
```

...and if you change your mind...

```bash
kubectl argo rollouts retry rollout blockexplorer
```

...and all sorts of other CLI features (see the built-in help).  

If you're more of a GUI-person,
or you need eye candy for a presentation, there's also
`kubectl argo rollouts dashboard` available for a browser-based app running on localhost.
(Expect battles with web proxies if you're on a private cluster, and please document here
if you find a headache-free solution!).