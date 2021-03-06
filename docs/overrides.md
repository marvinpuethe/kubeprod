# Modifying the default BKPR configuration

BKPR uses [jsonnet](https://jsonnet.org/) to describe Kubernetes manifests. Jsonnet is a simple programming language that generates JSON. If you are new to jsonnet and want to extend BKPR to suit your particular use case, we recommend to go through the [jsonnet basic tutorial](https://jsonnet.org/learning/tutorial.html) first, in order to get familiar with the syntax.

When deploying BKPR with `kubeprod install` two files are generated in your current directory: `kubeprod-autogen.json` with your cloud settings, and `kubeprod-manifest.jsonnet` with the root manifest for the Kubernetes objects.

To modify the default configuration you will need to edit the `kubeprod-manifest.jsonnet` manifest and add a set of jsonnet "overrides". In most cases, we will be using the jsonnet `+` operator, which will merge the JSON after that `+` operator into the parent object.

When BKPR is first deployed in a GKE cluster the following `kubeprod-manifest.jsonnet` is generated:

```jsonnet
// Cluster-specific configuration
(import "https://releases.kubeprod.io/files/v1.1.1/manifests/platforms/gke.jsonnet") {
	config:: import "kubeprod-autogen.json",
	// Place your overrides here
}
```

This snippet basically imports the default root manifest for your platform (in this example, GKE) and applies the generated cloud specific configuration.

If you want to check what manifests this default configuration would produce, you can use [`kubecfg`](https://github.com/ksonnet/kubecfg) a tool to manage Kubernetes resources as code and that understands `jsonnet` (actually, `kubeprod` uses `kubecfg` libraries internally):

```bash
kubecfg show kubeprod-manifest.jsonnet
```

## Example 1: Modifying Maximum Number of Replicas for `oauth2-proxy`

This example modifies the maximum number of replicas for the `oauth2-proxy` Deployment.

This is a snippet of the YAML generated by `kubecfg show`, specifically the one for the `oauth2-proxy` HorizontalPodAutoscaler, which will scale the deployment based on load:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
 metadata:
   labels:
     name: oauth2-proxy
   name: oauth2-proxy
   namespace: kubeprod
 spec:
   maxReplicas: 10
   minReplicas: 1
   scaleTargetRef:
     apiVersion: apps/v1beta1
     kind: Deployment
     name: oauth2-proxy
```

That snippet comes from the [`oauth2-proxy.jsonnet`](https://github.com/marvinpuethe/kubeprod/blob/master/manifests/components/oauth2-proxy.jsonnet) file, that gets imported in the previous one. Specifically, this is the part about the HorizontalPodAutoscaler:

```jsonnet
hpa: kube.HorizontalPodAutoscaler($.p + "oauth2-proxy") + $.metadata {
  target: $.deploy,
  spec+: {maxReplicas: 10},
},
```

`hpa` is a variable name used in `oauth2-proxy.jsonnet`, and you can see that `oauth2-proxy.jsonnet` itself is imported in a variable name called `oauth2_proxy` in the [main platform manifest](https://github.com/marvinpuethe/kubeprod/blob/master/manifests/platforms/gke.jsonnet#L28).

Let's imagine we want to limit the number of replicas of `oauth2-proxy` to 5 replicas. In order to do that, we would need to edit the `kubeprod-manifest.jsonnet` file to the following:

```jsonnet
// Cluster-specific configuration
(import "https://releases.kubeprod.io/files/v1.1.1/manifests/platforms/gke.jsonnet") {
    config:: import "kubeprod-autogen.json",
    // Place your overrides here
    oauth2_proxy+: {
        hpa+: {
            spec+: {
                maxReplicas: 5
            },
        },
    },
}
```

We are using the `+` operator to merge back to the object we are specifying. We are saying to merge `maxReplicas: 5` into the `spec` object, into the object `hpa` (the variable we used for the HorizontalPodAutoscaler) and up to the `oauth2_proxy` object (the one with the imported `oauth2-proxy.jsonnet`).

If we regenerate the YAML manifests for our `kubeprod-manifest.jsonnet` (`kubecfg show kubeprod-manifest.jsonnet`) we will now get the following description for the `oauth2-proxy` HorizontalPodAutoscaler:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  labels:
    name: oauth2-proxy
  name: oauth2-proxy
  namespace: kubeprod
spec:
  maxReplicas: 5
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: oauth2-proxy
```

The specification is unchanged, except the value of `maxReplicas` is now 5. Our override is the only change we will need to maintain ourselves, and we will be able to take advantage of future BKPR upgrades to other parts of the manifests without further work.

To apply these changes, run `kubeprod install` again from the folder where the modified `kubeprod-manifest.jsonnet` and original `kubeprod-autogen.json` are located.

## Example 2: Using the Staging Server for Let's Encrypt

This example modifies the cert-manager configuration to use the staging server for Let's Encrypt (useful for testing).

This is a snippet of the YAML generated with `kubecfg show`, specifically the part for the `cert-manager` deployment:

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    name: cert-manager
  name: cert-manager
  namespace: kubeprod
spec:
  minReadySeconds: 30
  replicas: 1
  selector:
    matchLabels:
      name: cert-manager
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "9402"
        prometheus.io/scrape: "true"
      labels:
        name: cert-manager
    spec:
      containers:
      - args:
        - --cluster-resource-namespace=$(POD_NAMESPACE)
        - --default-issuer-kind=ClusterIssuer
        - --default-issuer-name=letsencrypt-prod
        - --leader-election-namespace=$(POD_NAMESPACE)
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: bitnami/cert-manager:0.5.2-r37
        imagePullPolicy: IfNotPresent
        name: cert-manager
        ports:
        - containerPort: 9402
          name: prometheus
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
        stdin: false
        tty: false
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: cert-manager
      terminationGracePeriodSeconds: 30
      volumes: []
```

That snippet comes from the [`cert-manager.jsonnet`](https://github.com/marvinpuethe/kubeprod/blob/master/manifests/components/cert-manager.jsonnet) file, which is imported by the main platform file (eg `gke.jsonnet`). Specifically, this section of the file describes the pre-configured Let's Encrypt environments:

```jsonnet
  // Letsencrypt environments
  letsencrypt_environments:: {
    "prod": $.letsencryptProd.metadata.name,
    "staging": $.letsencryptStaging.metadata.name,
  },
  // Letsencrypt environment (defaults to the production one)
  letsencrypt_environment:: "prod",
```

There is a variable called `letsencrypt_environment` that has the value set to `prod` by default. That variable will be then be used in the argument for the container called `default-issuer-name`.

To customize BKPR and use the Let's Encrypt staging environment, modify the `kubeprod-manifest.jsonnet` like this:

```jsonnet
// Cluster-specific configuration
(import "manifests/platforms/gke.jsonnet") {
    config:: import "kubeprod-autogen.json",
    // Place your overrides here
    cert_manager+: {
        letsencrypt_environment:: "staging",
    }
}
```

The previous block of code does the following:

1. Imports the manifest at `manifests/platforms/gke.jsonnet` which describes how to deploy BKPR on GKE.
1. Under the `cert_manager` key (which implements the `cert-manager` component), overrides the `letsencrypt_environment` property to `staging` from its default value of `prod`.

To ensure the override is working as expected, let's evaluate the manifest again using `kubecfg` and filter the output searching for the Let's Encrypt environment. This time we will see that `cert-manager` will use the staging environment instead of the production environment:

```console
$ kubecfg show kubeprod-manifest.jsonnet | grep -- --default-issuer-name
        - --default-issuer-name=letsencrypt-staging
```

To apply these changes, run `kubeprod install` again from the folder where the `kubeprod-manifest.jsonnet` and `kubeprod-autogen.json` are located.

## More examples

There are more examples of other tested overrides in the [components section of the BKPR documentation](components.md).
