The Kubernetes crew just dropped the latest version, k8s 1.26 a few days ago, and it's packed with some seriously cool new features. One that's catching my eye is CEL for admission control - it allows us to create a ValidatingAdmissionPolicy, taking our cluster security to the next level.

> Validating admission policies offer a declarative, in-process alternative to validating admission webhooks.
>
> Validating admission policies use the Common Expression Language (CEL) to declare the validation rules of a policy.

### What is CEL?

> The Common Expression Language (CEL) implements common semantics for expression evaluation, enabling different applications to more easily interoperate.
> https://github.com/google/cel-spec

It is a programming language used in Kubernetes to create custom admission control policies. It allows for creating complex, fine-grained policies to enforce security and compliance in a cluster.

In a previous [article](https://www.kungfudev.com/posts/implementing-k8s-admission-controller/), we explored how Kubernetes admission controllers can be used to enforce validations, such as preventing the use of the latest tag for container images. In this post, we'll delve into the new ValidatingAdmissionPolicy feature, which allows us to take our validation capabilities even further in a much simple way without dealing with webhooks!


## Testing K8s 1.26 locally

For local testing, you can use tools like `kind` or `microk8s`.

Before using either `kind` or `microk8s`, make sure to enable the `ValidatingAdmissionPolicy` feature gate and the `admissionregistration.k8s.io/v1alpha1` API.

### microk8s - https://microk8s.io

```bash
microk8s install --channel 1.26
```

You'll need to edit the kube-apiserver file located at the address specified in the snippet below to activate the necessary features.

```text
$ vim /var/snap/microk8s/current/args/kube-apiserver

...
--feature-gates=ValidatingAdmissionPolicy=true
--runtime-config=admissionregistration.k8s.io/v1alpha1=true
...
```


If you're using a Mac and have microk8s installed, it uses multipass as VM. To access the VM, run the following command:

```bash
$ multipass list
Name                    State             IPv4             Image
microk8s-vm             Running           192.168.64.5     Ubuntu 18.04 LTS

$ multipass shell microk8s-vm
```

### kind - https://kind.sigs.k8s.io

With kind, you can create a cluster using a configuration file written in YAML that contains all the necessary settings.

```yaml
kind: Cluster
name: demo-cel
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  "ValidatingAdmissionPolicy": true
runtimeConfig:
  "admissionregistration.k8s.io/v1alpha1": true
nodes:
- role: control-plane
  image: kindest/node:v1.26.0
```

Run

```bash
$ kind create cluster --config=cluster.yaml
```


For both `kind` and `microk8s`, you can use kubectl to check if all required features are enabled.

```bash
$ kubectl api-versions | grep admissionregistration

admissionregistration.k8s.io/v1alpha1

$ kubectl api-resources | grep ValidatingAdmissionPolicy

validatingadmissionpolicies                      admissionregistration.k8s.io/v1alpha1   false        ValidatingAdmissionPolicy
validatingadmissionpolicybindings                admissionregistration.k8s.io/v1alpha1   false        ValidatingAdmissionPolicyBinding
```


Now that you have your cluster set up, you are ready to start experimenting with `ValidatingAdmissionPolicy`.

## Playing

To create a policy, you need to create the following:

1.  A ValidatingAdmissionPolicy that outlines the abstract logic of the policy (e.g. "this policy ensures that a specific label is set to a specific value").
2.  And a ValidatingAdmissionPolicyBinding that connects the ValidatingAdmissionPolicy to a specific scope or context."

Let's say you want to create a policy that ensures high availability in production deployments by requiring a minimum of 3 replicas. This policy could be implemented using the following `ValidatingAdmissionPolicy` and `ValidatingAdmissionPolicyBinding` resources:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "force-ha-in-prod"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas >= 3"
      message: "All production deployments should be HA with at least three replicas"
```


The YAML above shows how you can specify which resources and operations will be affected by the policy. Within the `spect.validations` section, you can use CEL to define the expressions that the resource must satisfy to be accepted by the policy. If an expression evaluates to false, the validation check is enforced according to the `spec.failurePolicy` field.

According to the official Kubernetes [documentation](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/#validation-expression), CEL expressions can access the contents of Admission requests and responses through a number of variables:

-   'object': The object from the incoming request (null for DELETE requests).
-   'oldObject': The existing object (null for CREATE requests).
-   'request': Attributes of the admission request.
-   'params': The parameter resource referred to by the policy binding being evaluated (null if ParamKind is unset).

Now that you have your first policy, you need to bind it using a `ValidatingAdmissionPolicyBinding`:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "force-ha-in-prod-binding"
spec:
  policyName: "force-ha-in-prod"
  matchResources:
    namespaceSelector:
      matchLabels:
        env: production
```

The `MatchResources` decides whether to run the admission control policy on an object based on whether it meets the match criteria. In this case, you're using `namespaceSelector` to apply the policy to all namespaces labeled with `env: production`.

Maybe you want to use `objectSelector`, which decides whether to run the validation based on if the object has matching labels. It is evaluated against both the old and new versions of an object, allowing you to set up validations that ensure that resources are not changed in certain ways.

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "force-ha-in-prod-object-binding"
spec:
  policyName: "force-ha-in-prod"
  matchResources:
    objectSelector:
      matchLabels:
        from: kungfudev
```


Of course, you can also use an empty `LabelSelector`, which will match all objects. This can be useful if you want your policy to run on all resources, regardless of their labels.

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "force-ha-in-prod-for-all-binding"
spec:
  policyName: "force-ha-in-prod"
  matchResources:
```

To test the policy you can try to apply the following YAML, which includes a namespace and a deployment with only two replicas.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments-system
  labels:
    env: production
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: payments-system
  name: payment-gateway
  labels:
    app: nginx
    from: kungfudev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23.3
        ports:
        - containerPort: 80
```

Testing it:

```bash
$ kubectl apply -f force-ha-in-prod.yaml
validatingadmissionpolicy.admissionregistration.k8s.io/force-ha-in-prod created

$ kubectl apply -f force-ha-in-prod-binding.yaml
validatingadmissionpolicybinding.admissionregistration.k8s.io/force-ha-in-prod-binding created

$ kubectl apply -f payment-deployment.yaml
The deployments "payment-gateway" is invalid: : ValidatingAdmissionPolicy 'force-ha-in-prod' with binding 'force-ha-in-prod-binding' denied request: All production deployments should be HA with at least three replicas
```

The policy rejects the deployment because it does not meet the requirements defined in the policy's validation. However, if you modify the deployment to use 4 replicas instead of 2, the policy should allow it.

```yaml
...
spec:
  replicas: 4
  selector:
    matchLabels:
...
```


```bash
$ kubectl apply -f payment-deployment.yaml
deployment.apps/payment-gateway created
```

Wow, we just made sure all our production deployments are highly available with a few simple moves! Our policy is pretty basic for now, but there's no stopping us from creating more complex validations. Imagine not allowing the "latest" tag, only allowing images from our own registry, or making sure all containers have CPU limits set. The possibilities are endless with the power of `ValidatingAdmissionPolicy` at our fingertips!

By harnessing the full capabilities of CEL, we can chain multiple expressions to create more intricate validations.

To learn about additional features and capabilities offered by CEL, please visit this [link](https://github.com/google/cel-spec/blob/master/doc/langdef.md).

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "prod-ready-policy"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.template.spec.containers.all(c, !c.image.endsWith(':latest'))"
      message: "cannot use the latest tag"
    - expression: "object.spec.template.spec.containers.all(c, c.image.startsWith('myregistry.com/'))"
      message: "image from an untrusted registry"
    - expression: "has(object.metadata.labels.env) && object.metadata.labels.env in ['prod', 'production']"
      message: "deployment should have an env and have to be prod or production"
    - expression: |
        object.spec.template.spec.containers.all(
          c, has(c.resources) && has(c.resources.limits) && has(c.resources.limits.cpu)
        )
      message: "container does not have a cpu limit set"
---
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "prod-reay-policy-binding"
spec:
  policyName: "prod-reay-policy"
  matchResources:
    namespaceSelector:
      matchExpressions:
      - key: "env"
        operator: "In"
        values: [prod, production,]
```

In the code above, you'll notice that we can use `matchExpressions` during binding to give our policy a more dynamic scope.

The ValidatingAdmissionPolicy feature also includes Parameter resources, which allow you to separate the policy configuration from its definition. This means that, for example, instead of hardcoding the minimum number of replicas for all namespaces in the HA Policy, you can use a parameter and configure different values for different environments. Check out the official [documentation](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/#parameter-resources) for more information on how to use this feature.

## In Conclusion

With CEL and the new ValidatingAdmissionPolicy feature in k8s 1.26, we now have the power to create custom, complex policies to enforce security and compliance in our clusters.

We know that Open Policy Agent (OPA) is widely used, but ValidatingAdmissionPolicy + CEL is designed to be more lightweight and easier to use than OPA. ValidatingAdmissionPolicy is natively integrated with Kubernetes, which means that it can be used directly within the Kubernetes API server without requiring a separate policy engine.

There are pros and cons to using either OPA or ValidatingAdmissionPolicy + CEL for policy enforcement in Kubernetes. OPA could be a more feature-rich and powerful policy engine, but it may require more setup and maintenance. ValidatingAdmissionPolicy + CEL is easier to use and integrates more seamlessly with Kubernetes, but it may not have all the advanced features of OPA. Ultimately, the choice of which policy engine to use will depend on the specific requirements and preferences of the user.
