---
title: Variables
description: >
    Data-driven policies for reuse and intelligent decision making
weight: 6
---

Variables make policies smarter and reusable by enabling references to data in the policy definition, the [admission review request](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#webhook-request-and-response), and external data sources like ConfigMaps and the Kubernetes API Server.

Variables are stored as JSON and Kyverno supports using [JMESPath](http://jmespath.org/)(pronounced "James path") to select and transform JSON data. With JMESPath, values from data sources are referenced in the format of `{{key1.key2.key3}}`. For example, to reference the name of an new/incoming resource during a `kubectl apply` action such as a Namespace, you would write this as a variable reference: `{{request.object.metadata.name}}`. The policy engine will substitute any values with the format `{{ <JMESPath> }}` with the variable value before processing the rule. Beginning with Kyverno v1.4.2, any non-resolved variable will be considered an empty string. This applies only to `preconditions` and `deny.conditions` blocks.

{{% alert title="Note" color="info" %}}
Variables are not currently allowed in `match` or `exclude` statements or `patchesJson6902.path`.
{{% /alert %}}

## Pre-defined Variables

Kyverno automatically creates a few useful variables and makes them available within rules:

1. `serviceAccountName`: the "userName" which is the last part of a service account (i.e. without the prefix `system:serviceaccount:<namespace>:`). For example, when processing a request from `system:serviceaccount:nirmata:user1` Kyverno will store the value `user1` in the variable `serviceAccountName`.

2. `serviceAccountNamespace`: the "namespace" part of the serviceAccount. For example, when processing a request from `system:serviceaccount:nirmata:user1` Kyverno will store `nirmata` in the variable `serviceAccountNamespace`.

3. `request.roles`: a list of roles stored in an array the given account may have. For example, `["foo:dave"]`.

4. `request.clusterRoles`: a list of cluster roles stored in an array. For example, `["dave-admin","system:basic-user","system:discovery","system:public-info-viewer"]`

5. `images`: a map of container image information, if available. See [Variables from container images](#variables-from-container-images) for more information.

{{% alert title="Note" color="warning" %}}
One of either `request.roles` or `request.clusterRoles` will be substituted as variables but not both.
{{% /alert %}}

## Variables from policy definitions

Kyverno policy definitions can refer to other fields in the policy definition as a form of "shortcut". This can be a useful way to analyze and compare values without having to explicitly define them.

In order for Kyverno to refer to these existing values in a manifest, it uses the notation `$(./../key_1/key_2)`. This may look familiar as it is essentially the same way Linux/Unix systems refer to relative paths. For example, consider the policy manifest snippet below.

```yaml
validationFailureAction: enforce
rules:
- name: check-tcpSocket
  match:
    resources:
      kinds:
      - Pod
  validate:
    message: "Port number for the livenessProbe must be less than that of the readinessProbe."
    pattern:
      spec:
        ^(containers):
        - livenessProbe:
            tcpSocket:
              port: "$(./../../../readinessProbe/tcpSocket/port)"
          readinessProbe:
            tcpSocket:
              port: "3000"
```

In this above example, for any containers found in a Pod spec, the field `readinessProbe.tcpSocket.port` must be `3000` and the field `livenessProbe.tcpSocket.port` must be the same value. The lookup expression can be thought of as a `cd` back three levels and down into the `readinessProbe` object.

Operators also work on manifest lookup variables as well so the previous snippet could be modified as such.

```yaml
- livenessProbe:
    tcpSocket:
      port: "$(<./../../../readinessProbe/tcpSocket/port)"
  readinessProbe:
    tcpSocket:
      port: "3000"
```

In this case, the field `livenessProbe.tcpSocket.port` must now be **less** than the value specified in `readinessProbe.tcpSocket.port`.

For more information on operators see the [Operators](/docs/writing-policies/validate/#operators) section.

## Variables from admission review requests

Kyverno operates as a webhook inside Kubernetes. Whenever a new request is made to the Kubernetes API server, for example to create a Pod, the API server sends this information to the webhooks registered to listen to the creation of Pod resources. This incoming data to a webhook is passed as a [`AdmissionReview`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#webhook-request-and-response) object. There are four commonly used data properties available in any AdmissionReview request:

- `{{request.operation}}`: the type of API action being performed (`CREATE`, `UPDATE`, `DELETE`, or `CONNECT`).
- `{{request.object}}`: the object being created or modified. It is null for `DELETE` requests.
- `{{request.oldObject}}`: the object being modified. It is null for `CREATE` and `CONNECT` requests.
- `{{request.userInfo}}`: contains information on who/what submitted the request which includes the `groups` and `username` keys.

Here are some examples of looking up this data:

1. Reference a resource name (type string)

`{{request.object.metadata.name}}`

2. Reference the metadata (type object)

`{{request.object.metadata}}`

3. Reference the name of a new Namespace resource being created

`{{request.object.name}}`

4. Reference the name of a user who submitted a request

`{{request.userInfo.username}}`

Variables from the `AdmissionReview` can also be combined with user-defined strings to create values for messages and other fields.

1. Build a name from multiple variables (type string)

`"ns-owner-{{request.object.metadata.namespace}}-{{request.userInfo.username}}-binding"`

Let's look at an example of how this AdmissionReview data can be used in Kyverno policies.

In the below `ClusterPolicy`, we wish to know which account created a given Pod resource. We can use information from the `AdmissionReview` contents, specifically the `username` key, to write this information out in the form of a label. Apply the following sample.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: who-created-this
spec:
  background: false
  rules:
  - name: who-created-this
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            created-by: "{{request.userInfo.username}}"
```

This sample will mutate all incoming Pod creation requests with a label named `created-by` and the value of the authenticated user based on their `kubeconfig`.

Create a simple Pod resource.

```sh
kubectl run busybox --image busybox:1.28
```

Now `get` the newly-created `busybox` Pod.

```sh
kubectl get po busybox --show-labels

NAME       READY   STATUS    RESTARTS   AGE   LABELS
busybox   0/1     Pending   0          25m   created-by=kubernetes-admin,run=busybox
```

In the output, we can clearly see the value of our `created-by` label is `kubernetes-admin` which, in this case, is the user who created the Pod.

## Variables from container images

Kyverno extracts image data from the AdmissionReview request and makes this available as a variable named `images` of type map in the rule context. Here is an example:

```json
{
  "containers": {
    "tomcat": {
      "registry": "https://ghcr.io",
      "name": "tomcat",
      "tag": "9"
    }
  },
  "initContainers": {
    "vault": {
      "registry": "https://ghcr.io",
      "name": "vault",
      "tag": "v3"
    }
  }
}
```

Whenever an AdmissionReview request has `containers` or `initContainers` defined, the `images` variable can be referenced as shown in the examples below:

Reference the image properties of container `tomcat`:

1. Reference the registry URL

`{{images.containers.tomcat.registry}}`

2. Reference the image name

`{{images.containers.tomcat.name}}`

3. Reference the image tag

`{{images.containers.tomcat.tag}}`

4. Reference the digest

`{{images.containers.tomcat.digest}}`

Reference the image properties of initContainer `vault`:

1. Reference the registry URL

`{{images.initContainers.vault.registry}}`

2. Reference the image name

`{{images.initContainers.vault.name}}`

3. Reference the image tag

`{{images.initContainers.vault.tag}}`

4. Reference the digest

`{{images.initContainers.vault.digest}}`

Kyverno by default sets an empty registry to `docker.io` and an empty tag to `latest`.

{{% alert title="Note" color="info" %}}
Note that certain characters must be escaped for JMESPath processing (ex. `-` in the case of container's name), escaping can be done by using double quotes with double escape character `\`, for example, `{{images.containers.\"my-container\".tag}}`.
{{% /alert %}}

You can also fetch image properties of all containers for further processing. For example, `{{ images.containers.*.name }}` creates a string list of all image names.

## Variables from external data sources

Some policy decisions require access to cluster resources and data managed by other Kubernetes controllers or external applications. For these types of policies, Kyverno allows HTTP calls to the Kubernetes API server and the use of ConfigMaps.

Data fetched from external sources is stored in a per-rule processing context that is used to evaluate variables by the policy engine. Once the data from external sources is stored in the context, it can be referenced like any other variable data.

Learn more about ConfigMap lookups and API Server calls in the [External Data Sources](/docs/writing-policies/external-data-sources/) section.

## Nested Lookups

It is also possible to nest JMESPath expressions inside one another when mixing data sourced from a ConfigMap and AdmissionReview, for example. By including one JMESPath expression inside the other, Kyverno will first substitute the inner expression before building the outer one as seen in the below example.

{{% alert title="Note" color="info" %}}
Nesting JMESPath expressions is supported in Kyverno version 1.3.0 and higher.
{{% /alert %}}

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: resource-annotater
spec:
  background: false
  rules:
  - name: add-resource-annotations
    context:
    - name: LabelsCM
      configMap:
        name: resource-annotater-reference
        namespace: default
    match:
      resources:
        kinds:
        - Pod
    mutate:
      overlay:
        metadata:
          annotations:
            foo: "{{LabelsCM.data.{{ request.object.metadata.labels.app }}}}"
```

In this example, AdmissionReview data is first collected in the inner expression in the form of `{{request.object.metadata.labels.app}}` while the outer expression is built from a ConfigMap context named `LabelsCM`.

## Evaluation Order

Kyverno policies can contain variables in:

* Rule context
* Rule preconditions
* Rule definitions:
  * Validation patterns
  * Validation deny rules
  * Mutate strategic merge patches (`patchesStrategicMerge`)
  * Generate resource data definitions

Variables are not supported in the `match` and `exclude` elements, so that rules can be matched quickly without having to load and process data. Variables are also not supported in the `patchesJson6902.path` key.

Since variables can be nested, it is important to understand the order in which the variables are evaluated. During admission control, here is how the engine processes rules:

1. The set of matching rules is determined by creating a hash from the request information to retrieve all matching rules based on the rule and resource types.
2. Each matched rule is further processed to fully evaluate the match and retrieve conditions.
3. The rule context is then evaluated and variables are loaded from data sources.
4. The preconditions are then checked.
5. The rule body is processed.

This ordering makes it possible to use request data when defining the context, and context variables in preconditions. Within the context itself, each variable is evaluated in the order of definition. Hence, if required, a variable can reference a prior variable but attempts to use a subsequent definition will result in errors.

## JMESPath custom functions

In addition to the list of [built-in functions JMESPath](https://jmespath.org/specification.html#builtin-functions) offers, Kyverno augments these by adding several others which make it even easier to craft Kubernetes policies.

```
base64_decode(string) string
base64_encode(string) string
compare(string, string) bool
contains(string, string) bool
equal_fold(string, string) bool
replace(str string, old string, new string, n float64) string
replace_all(str string, old string, new string) string
to_upper(string) string
to_lower(string) string
trim(str string, cutset string) string
split(str string, sep string) []string
regex_replace_all(regex string, src string|number, replace string|number) string (converts all parameters to string)
regex_replace_all_literal(regex string, src string|number, replace string|number) string (converts all parameters to string)
regex_match(string, string|number) bool
label_match(object, object) bool (object arguments must be enclosed in backticks; ex. `{{request.object.spec.template.metadata.labels}}`)
add(number, number) number
subtract(number, number) number
multiply(number, number) number
divide(number, number) number (divisor must be non zero)
modulo(number, number) number (divisor must be non-zero, arguments must be integers)
```

The special variable `{{@}}` may be used to refer to the current value in a given field, useful for source values.

To find examples of some of these functions in action, see the [Kyverno policies library](https://kyverno.io/policies/).
