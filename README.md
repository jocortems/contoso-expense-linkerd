# Linkerd Overview
Linkerd is a service mesh for Kubernetes and other frameworks. It makes running services easier and safer by giving you runtime debugging, observability, reliability, and security—all without requiring any changes to your code. Additionally Linkerd is very low touch and provides multiple functionalities out of the box, like mTLS and observability through Prometheus and Grafana. The reason we used Linkerd for this project over all other Service Mesh technologies available is because Linkerd has been donated to the CNCF, which guarantees that it will remain open source
## Linkerd Installation
As mentioned before, the installation of Linkerd is very low touch, it requires installing two components.

### CLI Installation

 First install the CLI in your local machine, if you are using Linux you can run the following command:

`curl -sL https://run.linkerd.io/install | sh`

You then need to add Linkerd to your PATH variable, note you will need to run this everytime you log into your shell:

`export PATH=$PATH:$HOME/.linkerd2/bin`

Alternatively if you are using Windows you can download the CLI directly from the [Linkerd Releases Page](https://github.com/linkerd/linkerd2/releases/)

Once the CLI has been installed you can verify it is running correctly:

`linkerd version`

### Control Plane Installation
In order to install Linkerd in your Kubernetes cluster your cluster must be running version 1.13 or above, you can validate that your cluster is configured properly for Linkerd installation by running:

`linkerd check --pre`

Note that Linkerd automatically generates TLS certificates for the sidecar proxies to provide automatic mTLS and while it automatically rotates those certificates every 24 hours it does not rotate the root TLS credentials used to issue these certificates; these credentials have a lifetime of 365 days, so if your cluster will outlive this timeframe you need to manually rotate them. Alternatively you can automate this procedure using [Cert-Manager](https://cert-manager.io/docs/) when you install Linkerd. The following steps will walk you through installing Linkerd to automatically rotate certificates with Cert-Manager. 

#### Prerequisites
- Cert-Manager is installed in your cluster
- An existing root certificate and its corresponding private key
- An intermediate certificate signed by the root CA and its corresponding private key
- All certificates must be using ECDSA P-256 algorithm

#### Passing the certificates to Linkerd
You can run the following command to pass the certificates to Linkerd. Note this can only be done at installation time

```
linkerd install \
  --identity-trust-anchors-file ca.crt \
  --identity-issuer-certificate-file issuer.crt \
  --identity-issuer-key-file issuer.key \
  | kubectl apply -f -
```

Since we have specified Linkerd to use a manually generated certificate rather than the automatically generated one we now need to create a new TLS secret to store the root CA; this secret will be later referenced by the cert-manager `Issuer` and used to sign TLS certificates issued to the proxies:

```
   linkerd-trust-anchor \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=linkerd
```

Let's create a cert-manager `Issuer` resource and reference our previously created secret:

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: linkerd-trust-anchor
  namespace: linkerd
spec:
  ca:
    secretName: linkerd-trust-anchor
EOF
```

Finally we need to create a cert-manager `Certificate` resource; this resource will be used to create a Certificate Signing Request that will be passed on to the `Issuer` we created in the previous step and in turn the `Issuer` will issue a properly signed certificate. This certificate will be stored in a secret named `linkerd-identity-issuer` in the Linkerd namespace

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: linkerd-identity-issuer
  namespace: linkerd
spec:
  secretName: linkerd-identity-issuer
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: linkerd-trust-anchor
    kind: Issuer
  commonName: identity.linkerd.cluster.local
  isCA: true
  keyAlgorithm: ecdsa
  usages:
  - cert sign
  - crl sign
  - server auth
  - client auth
EOF
```

#### Injecting the sidecar proxies

Now that we have installed Linkerd in our Kubernetes cluster we need to instruct it to inject the proxies as sidecars into our pods. There are three ways to accomplish this task:

- Annotating the namespace(s) we want to inject Linkerd proxies into with `linkerd.io/inject: enabled`
- Annotating the deployments we want to inject Linkerd proxies into with `linkerd.io/inject: enabled`
- Injecting the proxies using Linkerd CLI

The first two options above will only inject the Linkerd proxies to pods created after the annotation was added, so once you annotate the namespace and/or deployments you will need to delete the pods associated with them. Assuming you are not manually creating any pods the controllers will create new pods with the Linkerd proxy automatically. In order to inject the Linkerd proxies to running pods without having to delete them you can use the CLI as follows:

```
kubectl get -n conexp deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```

You can then verify the proxies were injected and are properly working:

`linkerd -n conexp check --proxy`

# Ingress Configuration
Unlike Istio, Linkerd doesn't provide its own ingress controller, instead Linkerd integrates with different ingress controllers. For this project we have decided to use Datawire [Ambassador Edge Stack (AES)](https://www.getambassador.io/products/), the main reasons we chose AES over other Ingress Controllers is because it has built in integration with Let's Encrypt (and any other ACME provider) which simplifies certificate management, it also has a built in OAuth2 Client and half of a Resource Server that can validate Access Tokens before allowing the request to the upstream service, and finally because AES is built on top of [Envoy proxy](https://www.envoyproxy.io/), which is part of CNCF.

## Ambassador Edge Stack Installation
There are mainly three ways to install AES on a Kubernetes cluster, either manually, using Helm or installing the AES Operator. For this project we have decided to install the AES operator in order to make sure AES is automatically updated and always running the latest version. For details about how to install the Ambassador Edge Stack Operator you can refer to the [documentation](https://github.com/datawire/ambassador-operator/blob/master/README.md#version-syntax).

AES is exposed through a `serviceType: LoadBalancer` Kubernetes object; you can retrieve the public IP address associated with it like you would retrieve any other Kubernetes service. This is the IP address that will be used to expose the services in the cluster:

```
jorge@Azure:~/cncf$ kubectl get svc -n ambassador
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ambassador                    LoadBalancer   10.0.250.182   52.185.68.167   80:31413/TCP,443:31360/TCP   2d1h
ambassador-admin              ClusterIP      10.0.61.179    <none>          8877/TCP                     2d1h
ambassador-operator-metrics   ClusterIP      10.0.209.182   <none>          8383/TCP,8686/TCP            2d1h
ambassador-redis              ClusterIP      10.0.111.59    <none>          6379/TCP                     2d1h
```
You can then browse to this IP address to get to the AES landing page and download `edgectl`, which is AES CLI. This is needed in order to access AES Policy Console.

![](https://yorchlogs.blob.core.windows.net/cncf/ambassador-landingpage.png?sp=r&st=2020-04-01T00:03:26Z&se=2021-04-01T08:03:26Z&spr=https&sv=2019-02-02&sr=b&sig=vysadoi45NQQ8f7vdcAYandEpLlLaQQ1l8pLv3HMF2I%3D)

From the Policy Consoloe you can request a free Community License, note this is required in order to be able to use the features we will be leveraging as part of this project.

![](https://yorchlogs.blob.core.windows.net/cncf/ambassador-policyconsole.png?sp=r&st=2020-04-01T00:07:07Z&se=2021-04-01T08:07:07Z&spr=https&sv=2019-02-02&sr=b&sig=RVyQpCiiqe8Orc8GmrcAmoCIKXvrkUgTPlslGQKdsdI%3D)

## Ingress Configuration
Unlike most Ingress Controllers, AES does not use an `Ingress` resource, instead it relies on a `Mapping` resource, this is a CRD created when AES is installed in the cluster. This resource can be defined either as a standalone resource or as a configuration block under a special annotation `getambassador.io/config` within the `Service` definition YAML resource we want to expose. Using an annotation has the advantage that the lifecycle of these resources is tightly coupled to the lifecycle of the `Service` resource, thus when the service is deleted the `Mapping` configuration is as well deleted from AES; however this is hard to troubleshoot since those resources are not exposed neither as Kubernetes objects nor as mappings in the AES console and the service definition YAML files can become cumbersome to read.

First we will create a standalone `Mapping` resource to access the Linkerd dashboard:

```
apiVersion: getambassador.io/v2
kind: Mapping
metadata:
  name: linkerd-dashboard
  namespace: conexp
spec:
  prefix: /
  service: linkerd-web.linkerd:8084
  host: linkerd-dashboard.srinipadala.xyz
  host_rewrite: linkerd-web.linkerd.svc.cluster.local
```
This will expose the Linkerd dashboard at http://linkerd-dashboard.srinipadala.xyz. Note we defined property `spec.service` as `serviceName.namespace:servicePort`; namespace and service port are optional but if they are not specified AES will only try to resolve serviceNames in the same namespace as the mapping resource and port will default to `Http`. Because Linkerd is deployed in its own namespace and it listens on port 8084 we must specify those values. Also note that we need to rewrite the `Host` header because the Linkerd dashboard rejects any requests whose `Host` header is not one of `localhost`, `127.0.0.1` or the service name `linkerd-web.linkerd.svc`. 

Also note that by default AES performs `Http` to `Https` redirection and it will present its own self signed certificate; however as we mentioned earlier AES offers built in integration with ACME protocol and as such it can leverage Let's Encrypt to request certificates for the services exposed through it. This is done through the `Host` CRD; same as with `Mapping`, `Host` can be defined as a standalone resource or as a configuration block under the `getambassador.io/config` annotation, however from the testing conducted as part of this project we noticed that when defining `Host` as a configuration block under the annotation AES won't use the certificate issued by Let's Encrypt and will continue to use its own self signed certificate. Following is the `Host` resource definition that we will use to get a certificate for our Linkerd dashboard:

```
apiVersion: getambassador.io/v2
kind: Host
metadata:
  name: linkerd-dashboard
# Must be in the same namespace as the Mapping object
  namespace: conexp
spec:
# Must match the host defined in the Mapping object
  hostname: linkerd-dashboard.srinipadala.xyz
  acmeProvider:
# Must be a valid email address for certificate management
    email: valid@email.com
# Authority is not needed and defaults to Let's Encrypt production endpoint
# Note however that the production endpoint has very aggressive rate limiting
# For testing it is recommended to use the staging endpoint
    authority: https://acme-staging-v02.api.letsencrypt.org/directory
# TLS Secret is not needed, AES will create one by default, however in order to
# Facilitate troubleshooting and enforce naming convention we are specifying it manually
  tlsSecret:
    name: linkerd-dashboard
```
Verify that the resources have been successfully created:
```
jorge@Azure:~/cncf$ kubectl get mapping -n conexp -o wide
NAME                PREFIX   SERVICE                    STATE     REASON
linkerd-dashboard   /        linkerd-web.linkerd:8084   Running

jorge@Azure:~/cncf$ kubectl get host -n conexp -o wide
NAME                  HOSTNAME                              STATE   PHASE COMPLETED   PHASE PENDING   AGE
linkerd-dashboard     linkerd-dashboard.srinipadala.xyz     Ready                                     3h39m
# If there are problems obtaining a certificate from Let's Encrypt STATE will not be ready
# You can describe the Host resource to understand why it is failing

jorge@Azure:~/cncf$ kubectl get secrets -n conexp | grep linkerd-dashboard
linkerd-dashboard                                                                             kubernetes.io/tls                     2      3h40m
```
We now have all of the resources we need to connect to our Linkerd dashboard from the Internet by browsing to http://linkerd-dashboard.srinipadala.xyz, note that because we are using Let's Encrypt staging endpoint rather than the production endpoint we will get a certificate warning because the certificate was issued by an untrusted CA, however if you look at the certificate details you will see the subject is indeed `linkerd-dashboard.srinipadala.xyz`.


![](https://yorchlogs.blob.core.windows.net/cncf/linkerd-certificate.png?sp=r&st=2020-04-01T00:24:02Z&se=2021-04-01T08:24:02Z&spr=https&sv=2019-02-02&sr=b&sig=DplPWPovTnUlVR3FVHI%2FlT8eXaJASAGTbangHRuimcg%3D)

Since the Linkerd dashboard provides observability into our services and allows to make some configuration changes to Linkerd, and since service meshes are a critical part of the system, it is a good idea to secure the dashboard with authentication. Earlier we mentioned that AES offers a built in `OAuth2` client and half of a `Resource Server`, it does so through two CRDs, an `OAuth2 Filter` which is used to specify the Identity Proivder parameters for the `OAuth2` client to authenticate with and a `FilterPolicy` that binds the `Filter` to a URL to be applied to for authentication. This allows us to implement authentication for our services without having to write the code for it. For this project we will use Azure AD as the Identity Provider, however AES supports multiple Identity Providers, you can read more about it [here](https://www.getambassador.io/docs/latest/topics/using/filters/oauth2/):

```
apiVersion: getambassador.io/v2
kind: Filter
metadata:
  name: azure-ad
  namespace: conexp
spec:
  OAuth2:
    authorizationURL: https://login.microsoftonline.com/{{TENANT_ID}}/v2.0
    clientURL: https://linkerd-dashboard.srinipadala.xyz
    clientID: {{APPLICATION_ID}}
    secret: {{APPLICATION_SECRET_ID}}
---
apiVersion: getambassador.io/v2
kind: FilterPolicy
metadata:
  name: azure-policy
  namespace: conexp
spec:
  rules:
    - host: https://linkerd-dashboard.srinipadala.xyz
      path: /
      filters:
        - name: azure-ad
```

Same as we did to expose the Linkerd dashboard, we will now expose the services that comprise our Contoso Expenses application using AES; however we are now going to define the `Mapping` objects as a configuration block under `getambassador.io/config` annotation within the `Service` definition YAML for the frontend service:

```
apiVersion: v1
kind: Service
metadata:
  name: ambassador-frontend
  namespace: conexp
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v2
      kind: Mapping
      name: backend-mapping
      service: backend-svc
      host: ambassador-frontend.srinipadala.xyz
      prefix: /Expenses
      add_linkerd_headers: true
      ---
      apiVersion: ambassador/v2
      kind: Mapping
      name: frontend-mapping
      service: ambassador-frontend
      host: ambassador-frontend.srinipadala.xyz
      prefix: /
      add_linkerd_headers: true
      ---
      apiVersion: ambassador/v2
      kind: Mapping
      name: frontend-files-mapping
      service: ambassador-frontend
      precedence: 10
      host: ambassador-frontend.srinipadala.xyz
      prefix: "/(js|css)/site\\.min\\.(js|css)"
      prefix_regex: true
      rewrite: ""
      add_linkerd_headers: true
      ---
      apiVersion: ambassador/v2
      kind: Mapping
      name: backend-files-mapping
      service: backend-svc
      host: ambassador-frontend.srinipadala.xyz
      prefix: "(/[a-z0-9-]+)*/[a-z0-9-]+(\\.[a-z]+)+"
      prefix_regex: true
      rewrite: ""
      add_linkerd_headers: true
  labels:
    app: conexp-frontend
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: conexp-frontend
```

There are multiple new properties used in the four different `Mapping` resources we are creating as part of this service definition YAML, so let's break them down.

The first thing to notice is that when defining a `Mapping` resource as part of the service annotation its structure is different, for instance `apiVersion` is `ambassador/v2` rather than `getambassador.io/V2`, also there are no `metadata` nor `spec` keys, however the values belonging to those still need to be defined, and their indentation is removed.

The next property we will look at is `add_linkerd_headers: true`, this causes AES to add a `l5d-dst-override` header to instruct Linkerd what service the request is destined for, this is what provides integration of AES with Linkerd.

One thing to be aware with AES is that it will strip the matched `prefix` out of the path when sending the request upstream, so if your application is configured to serve requests from the path matched by `prefix` you need to use `rewrite` property and either set it to the same value as `prefix` or set it to an empty string (`""`); as you can imagine you can also rewrite the matched `prefix` to an arbitrary value. Also note that `prefix` is case sensitve and is not an exact match, thus it will match on any path that starts with `prefix`; for example in our `backend-mapping` above `prefix: /Expenses` will match:
```
/Expenses
/Expensesallitems
/Expenses/new
/Expenses/v1/.../vn
```

As you can see from our configuration above, you can define `prefix` as a [C++ ECMAScript Regular Expression](https://en.cppreference.com/w/cpp/regex/ecmascript) by setting property `prefix_regex: true`. It is important to note that when using a regexp prefix it becomes an exact match, for example given `prefix: /[eE]xpenses` will only match one of either `/expenses` or `/Expenses`, however if requests for files under this path arrive -i.e. `/Expenses/index.html` AES will return `404` because it doesn't have a route for that path.

The other property being used is `precedence`. It instructs AES in which order to evaluate the mappings. By default AES processes more constrained mappings first and for the most part it is not necessary to specify a precedence; however when there are multiple mappings using `prefix_regex` and/or `host_regex` without additional constrains like matching on HTTP Headers and/or HTTP Methods it becomes important to specify a precedence to make sure the mappings are processed in the desired order, otherwise the order of evaluation of these mappings can yield unexpected results. The higher the precedence value the earlier the mapping will be evaluated, and when precedence is not specified the default precedence is `0`.

Finally, it is worth noting that because these `Mapping` resources are being defined within the `ambassador-frontend` service annotation their lifecycle is tightly coupled with the service lifecycle, meaning these mappings will only exist as long as the service exists and will automatically be removed when the service is deleted. Also note that because of this same reason mappings defined as part of a service annotation won't show up neither as Kubernetes resources nor on the AES Mappings dashboard, which makes it hard to troubleshoot:

```
jorge@Azure:~$ kubectl get mappings --all-namespaces
NAMESPACE    NAME                        PREFIX            SERVICE                    STATE     REASON
ambassador   ambassador-devportal        /documentation/   127.0.0.1:8500             Running
ambassador   ambassador-devportal-api    /openapi/         127.0.0.1:8500             Running
ambassador   ambassador-devportal-demo   /docs/            127.0.0.1:8500             Running
conexp       linkerd-dashboard           /                 linkerd-web.linkerd:8084   Running
```
![](https://yorchlogs.blob.core.windows.net/cncf/ambassador-mappings.png?sp=r&st=2020-03-30T02:15:35Z&se=2021-06-30T10:15:35Z&spr=https&sv=2019-02-02&sr=b&sig=ktPtiEdauj4bzHmuk0HViPNoezC%2Bv62ZR3LpFdLgs%2BU%3D)

Now that we have created the mappings to expose our Contoso Expenses services to the Internet through AES we will create a `Host` resource in order to get a certificate from Let's Encrypt for our hostname `ambassador-frontend.srinipadala.xyz`, this is the same as what we did for the Linkerd dashboard:

```
apiVersion: getambassador.io/v2
kind: Host
metadata:
  name: ambassador-frontend
  namespace: conexp
spec:
  hostname: ambassador-frontend.srinipadala.xyz
  acmeProvider:
    email: jocorte@microsoft.com
    authority: https://acme-staging-v02.api.letsencrypt.org/directory
  tlsSecret:
    name: ambassador-frontend
```
We can now connect to https://ambassador-frontend.srinipadala.xyz and validate everything is working properly:

![](https://yorchlogs.blob.core.windows.net/cncf/ambassador-frontend.png?sp=r&st=2020-04-01T00:52:50Z&se=2021-04-01T08:52:50Z&spr=https&sv=2019-02-02&sr=b&sig=1wdY2ZXifXMjo9L4x3Aako8OjMvJ9N8runHwc2b61nw%3D)

# References
[Linkerd Official Documentation](https://linkerd.io/2/overview/)

[Ambassador Edge Stack Official Documentation](https://www.getambassador.io/docs/)
