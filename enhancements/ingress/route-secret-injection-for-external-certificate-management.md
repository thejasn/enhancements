---
title: route-certificate-reference-for-external-certificate-management
authors:
  - '@thejasn'
reviewers:
  - '@Miciah'
  - '@alebedev87'
  - '@tgeer'
  - '@joelspeed'
  - '@deads2k'
approvers:
  - '@Miciah'
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
  - '@joelspeed'
creation-date: 2022-12-13
last-updated: 2023-01-27
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - https://issues.redhat.com/browse/CFE-704
---

# Route Secret Injection For External Certificate Management

## Summary

Currently, users of OpenShift cannot very easily integrate OpenShift Routes
with third-party certificate management solutions like [cert-manager](https://github.com/cert-manager/cert-manager).
And this is mainly due to the design of Routes API which deliberately requires the certificate
data to be present in the Route object as opposed to having a reference. This is especially
problematic when third-party solutions also manage the life cycle (create/renew/delete)
of the generated certificates which OpenShift Routes does not support and requires
manual intervention to manage certificate life cycle.

This enhancement aims to provide a solution where OpenShift Routes can support
integration with third-party certificate management solutions like cert-manager and
avoid manual certificate management by the user which is more error prone.

## Motivation

OpenShift customers currently manually manage certificates for user workloads
by updating OpenShift Routes with the updated certificate data during expiry/renew
workflow. This is cumbersome activity if users have a huge number of workloads and
is also error prone.

This enhancement adds the support to OpenShift Routes for third-party certificate
management solutions like cert-manager.

### User Stories

- As an end user of Route API, I want to be able to provide a tls secret reference on
  OpenShift Routes so that I can integrate with third-party certificate management solution

- As an OpenShift cluster administrator, I want to use third-party solutions like cert-manager
  for certificate management of user workloads on OpenShift so that no manual process is
  required to renew expired certificates.

- As an end user of Route API, I want OpenShift Routes to support both manual and managed
  mode of operation for certification management so that I can switch between manual certificate
  management and third-party certificate management.

- As an OpenShift engineer, I want to be able to update Route API so that I can integrate
  OpenShift Routes with third-party certificate management solutions like cert-manager.

- An an OpenShift engineer, I want to be able to watch routes and process routes having valid
  annotations so that the certificate data is injected into the route CR.

- As an OpenShift engineer, I want to be able to run e2e tests as part of openshift/origin
  so that testcases are executed to signal feature health by CI executions.

### Goals

- Provide users with a configurable option in Route API to provide externally managed certificate data.
- Provide users with a mechanism to switch between using externally managed certificates and
  manually managed certificates on OpenShift routes and vice-versa.
- Provide latest certificate type information on the Route.

### Non-Goals

- Provide certificate life cycle management controls on the Route API (expiryAfter, renewBefore, etc).
- Provide migration of certificates from current solution.
- Provide smooth roll out of new certificates on OpenShift router when managed certificates
  are renewed.

## Proposal

This enhancement proposes extending the openshift/router to read serving certificate data either
from the Route `.spec.tls.certificate` and `.spec.tls.key` or from a new field `.spec.tls.certificateRef`.
This `certificateRef` field will enables the users to provide a reference to a Secret containing
the serving cert/key pair that will be parsed and served by OpenShift router.

### Workflow Description

End users have 2 possible variations for the creation of the route:

- Create a route for the user workload and this is completely managed by the end user.
- Create an ingress for the user workload and a managed route is created automatically
  by the ingress-to-route controller.

Both these workflows will support integrating with third party certificate management
solutions like cert-manager with the secret-reference annotation described under [API Extensions](#api-extensions).

**When a Route is directly used to expose user workload**

- The end user must have generated the serving certificate as a pre requisite
  using third-party systems like cert-manager.
- In cert-manager's case, the [Certificate](https://cert-manager.io/docs/usage/certificate/#creating-certificate-resources) CR must be created in the same namespace
  where the Route is going to be created.
- The end user must create a role and in the same namespace as the secret,
  ```
  oc create role secret-reader --verb=get,list,watch --resource=secrets --resourceName=<secret-name>
  ```
- The end user must create a rolebinding in the same namespace as the secret
  and bind the router serviceaccount to the above created role.
  ```
  oc create rolebinding foo-secret-reader --role=secret-reader --serviceaccount=openshift-ingress:router --namespace=<current-namespace>
  ```
- To expose a user workload, the user would create a new Route with the
  `.spec.tls.certificateRef` referencing the generated secret that was created
  in the previous step.
- If the secret that is referenced exists and has a successfully generated
  cert/key pair, the router will serve this certificate.

**When an Ingress is used to expose user workload**

- The end user must have generated the serving certificate as a pre requisite
  using third-party systems like cert-manager.
- In cert-manager's case, the [Certificate](https://cert-manager.io/docs/usage/certificate/#creating-certificate-resources) CR must be created in the same namespace
  where the Ingress is going to be created.
- To expose a user workload, a new Ingress with the generated secret
  referenced in `.spec.tls.secretName` needs to be created.
- If the secret CR that is referenced exists and has a successfully generated
  cert/key pair, the ingress-to-route controller adds this secret name to
  the created route `.spec.tls.certificateRef`.

#### Certificate Life cycle

Post integrating with third-party solutions like cert-manager, the end user can also
disable automatic certificate management. This can be done by,

- The end user deletes the secret that was associated with the Certificate CR.

**Deleting the generated secret that contains the serving cert/key pair**
The end user can delete the secret referenced on the Route containing the certificate data,
this will result in the router using the default certificates that are generated by
the cluster-ingress-operator. Since the referenced secret is missing, the router raises an event
containing information as to why it switched to using default certificates.

#### Variation [optional]

N.A

### API Extensions

A `.spec.tls.certificateRef` field is added to Route `.spec.tls` which can be used to provide a secret name
containing the certificate data instead of using `.spec.tls.certificate` and `spec.tls.key`.

```go

type TLSConfig struct {
	// ...

	CertificateRef *corev1.LocalObjectReference `json:"certificateRef,omitempty" protobuf:"bytes,7,opt,name=certificateRef"`
}
```

_Note_: The default value would be `nil`. The secret is required to be created
in the same namespace as that of the Route. The secret must be of type
`kubernetes.io/tls` and the tls.key and the tls.crt key must be provided in
the `data` (or `stringData`) field of the Secret configuration.

If neither `.spec.tls.certificateRef` or `.spec.tls.certificate` and `.spec.tls.key` are
provided the router will serve the default generated secret.

The Route API will also be updated to denote certificate status under `.status`,

```go

// RouteStatus provides relevant info about the status of a route, including which routers
// acknowledge it.
type RouteStatus struct {
    // ...

    // CertificateConditions describes the type of certificate that is being served
    // by the router(s) associated with this route.
    CertificateConditions []metav1.Condition `json:"certificateConditions,omitempty" protobuf:"bytes,2,rep,name=certificate"`
}


const (
    // DefaultCertificateCondition denotes that the user has not provided
    // any custom certificate and the default generated certificate is being
    // served on the route.
    DefaultCertificateCondition  = "Default"

    // CustomCertificateCondition denotes that there is a custom certificate
    // being served on the route.
    CustomCertificateCondition = "Custom"

    // ManagedCertificateCondition denotes that a certificateRef is present 
    // on the route and is used to reference a third-party managed secret
    // containing the certificate data that is being served on the route.
    ManagedCertificateCondition = "Managed"
)

```

#### Variation

N.A

### Implementation Details/Notes/Constraints [optional]

The router will read the secret referenced in `.spec.tls.certificateRef` if present and
if all pre-conditions are met it uses this certificate us configure haproxy.

The pre-conditions required for this to work are,

- The secret created should be in the same namespace as that of the route.
- The secret created is of type `kubernetes.io/tls`.
- The router serviceaccount must have permission to read this secret particular secret.
  - The role and rolebinding to provide this access must be provided by the user.

The router will will not have any active watches on the secret and will only
do a single look when there is any update on routes. The router will maintain
a secret hash in order to be able to reload if the secret content has changed.

A route validating admission webhook verifies if the router associated with the
route has permissions to read the secret that is referenced at `.spec.tls.certificateRef`.
This validation is only performed if `.spec.tls.certificateRef` is non-nil and non-empty.

### Risks and Mitigations

The TechPreview feature will not handle secret updates, meaning upon certificate renewal/rotation
the router will not load the new certificates until the route is updated.

### Drawbacks

The user will need to manually create provide and maintain the rbac required by the 
router so that it can access secrets securely. This becomes tedious when users have
1000s of Routes.

## Design Details

### Open Questions [optional]

- Performance testing of openshift-router?
- What does taking Tech Preview to GA look like?

### Test Plan

This enhancement will be tested in isolation of cert-manager-operator as part of
core OCP payload. This will also be tested with cert-manager-operator as part of the
operators e2e test suite.

1. Test Route API without interfacing with Ingresses

   a. Create a edge terminated route that is using default certificates.
   b. Update route with the certificate reference and ensure route has updated `.status`
   c. Verify that the associated openshift routers are also serving the updated
   certificates and not the default certificates.
   d. Update certificate data and verify if the certificates served by the router
   are updated as well.
   d. Delete the certificate reference and ensure that the route `.status`
   denotes the usage of default certificates. Also verify the associated
   openshift router is serving the default certificates.

Various other transitions, `Custom`->`Managed`-`Custom` will also be tested.

### Graduation Criteria

This feature will initially be released as Tech Preview only.

#### Dev Preview -> Tech Preview

N/A. This feature will go directly to Tech Preview.

#### Tech Preview -> GA

The router will need additional logic to handle secret updates using single item
list/watch for every referenced secret. This pattern needs to be brought over
from kubelet's [secret_manager.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/secret/secret_manager.go)

#### Removing a deprecated feature

N/A.

### Upgrade / Downgrade Strategy

On downgrades, all routes using `route.openshift.io/tls-secret-name` annotation will continue to use the custom certificates
indefinitely unless the route is manually edited and the `.spec.tls` is updated.

### Version Skew Strategy

This feature will be added as a TechPreviewNoUprade.

### Operational Aspects of API Extensions

N.A

#### Failure Modes

When using routes with third-party certificate management solutions like cert-manager, this
adds a hard dependency in order of operation. In scenarios where the secret has not been created
by third-party solutions like cert-manager, the route would not be successfully created due
to the dependency on the route.

> A solution to this could be that upon failure to use the secret due to various reasons,
> the controller defaults to using the default generated certificates and updating the route
> status appropriately. The controller also publishes an event indicating the switch has been made.

#### Support Procedures

The new introduced `RouteCertificateConditionType` types will provide the required information into
the current state of route objects with respect to certificates. While using third-party solutions
like cert-manager for certificate management the `RouteCertificateCondition` will indicate which type
of certificate is associated with the route.

## Implementation History

N.A

## Alternatives

N.A

## Infrastructure Needed [optional]

N.A
