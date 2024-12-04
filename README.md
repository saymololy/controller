# controller
// TODO(user): Add simple overview of use/purpose

## Description
// TODO(user): An in-depth paragraph about your project and overview of use

## Getting Started

### Prerequisites
- go version v1.22.0+
- docker version 17.03+.
- kubectl version v1.11.3+.
- Access to a Kubernetes v1.11.3+ cluster.

### Init Project

```
kubebuilder init --domain mydomain.com --repo github.com/syamololy/controller
kubebuilder edit --multigroup=true

kubebuilder create api --group blog --version v1 --kind Foo
kubebuilder create webhook --kind Foo --group blog --version v1 --defaulting --programmatic-validation
```

### Update mutating & validating logic code

**NOTE:** Edit `api/blog/v1/foo_webhook.go`

**update mutating logic code**

```
// +kubebuilder:webhook:path=/mutate-blog-mydomain-com-v1-foo,mutating=true,failurePolicy=fail,sideEffects=None,groups=blog.mydomain.com,resources=foos,verbs=create;update,versions=v1,name=mfoo.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &Foo{}

// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *Foo) Default() {
	foolog.Info("default", "name", r.Name)

	// TODO(user): fill in your defaulting logic.
	if r.Spec.Replicas == 0 {
		r.Spec.Replicas = 2
	}
}
```

**update validating logic code**

```
// +kubebuilder:webhook:path=/validate-blog-mydomain-com-v1-foo,mutating=false,failurePolicy=fail,sideEffects=None,groups=blog.mydomain.com,resources=foos,verbs=create;update,versions=v1,name=vfoo.kb.io,admissionReviewVersions=v1

var _ webhook.Validator = &Foo{}

// ValidateCreate implements webhook.Validator so a webhook will be registered for the type
func (r *Foo) ValidateCreate() (admission.Warnings, error) {
	foolog.Info("validate create", "name", r.Name)

	// TODO(user): fill in your validation logic upon object creation.
	return validateReplicas(r)
}

// ValidateUpdate implements webhook.Validator so a webhook will be registered for the type
func (r *Foo) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
	foolog.Info("validate update", "name", r.Name)

	// TODO(user): fill in your validation logic upon object update.
	return validateReplicas(r)
}

// ValidateDelete implements webhook.Validator so a webhook will be registered for the type
func (r *Foo) ValidateDelete() (admission.Warnings, error) {
	foolog.Info("validate delete", "name", r.Name)

	// TODO(user): fill in your validation logic upon object deletion.
	return validateReplicas(r)
}

func validateReplicas(r *Foo) (admission.Warnings, error) {
	if r.Spec.Replicas > 5 {
		return nil, fmt.Errorf("foo replicas cannot be more than 5")
	}
	return nil, nil
}
```

### To Deploy on the cluster
**Eable the cert-manager and webhook via kubebuilder:**

1. Edit `config/crd/kustomization.yaml`, enable `- path: patches/webhook_in_blog_foos.yaml`  and `- path: patches/cainjection_in_blog_foos.yaml`

```
patches:
- path: patches/webhook_in_blog_foos.yaml
- path: patches/cainjection_in_blog_foos.yaml
```

2. Edit `config/default/kustomization.yaml`, under `resources` section, enabled `../certmanager` and `../webhook`

```
resources:
- ../webhook
- ../certmanager
```


3. Edit `config/default/kustomization.yaml`, under `patches` section, enabled `- path: manager_webhook_patch.yaml` and `- path: webhookcainjection_patch.yaml`

```
patches:
- path: manager_webhook_patch.yaml
- path: webhookcainjection_patch.yaml
```

4. Edit `config/default/kustomization.yaml`, enabled section `replacements`

```
replacements:
 - source: # Add cert-manager annotation to ValidatingWebhookConfiguration, MutatingWebhookConfiguration and CRDs
     kind: Certificate
     group: cert-manager.io
     version: v1
     name: serving-cert # this name should match the one in certificate.yaml
     fieldPath: .metadata.namespace # namespace of the certificate CR
   targets:
     - select:
         kind: ValidatingWebhookConfiguration
       fieldPaths:
         - .metadata.annotations.[cert-manager.io/inject-ca-from]
       options:
         delimiter: '/'
         index: 0
         create: true
     - select:
         kind: MutatingWebhookConfiguration
       fieldPaths:
         - .metadata.annotations.[cert-manager.io/inject-ca-from]
       options:
         delimiter: '/'
         index: 0
         create: true
     - select:
         kind: CustomResourceDefinition
       fieldPaths:
         - .metadata.annotations.[cert-manager.io/inject-ca-from]
       options:
         delimiter: '/'
         index: 0
         create: true
 - source:
     kind: Certificate
     group: cert-manager.io
     version: v1
     name: serving-cert # this name should match the one in certificate.yaml
     fieldPath: .metadata.name
   targets:
     - select:
         kind: ValidatingWebhookConfiguration
       fieldPaths:
         - .metadata.annotations.[cert-manager.io/inject-ca-from]
       options:
         delimiter: '/'
         index: 1
         create: true
     - select:
         kind: MutatingWebhookConfiguration
       fieldPaths:
         - .metadata.annotations.[cert-manager.io/inject-ca-from]
       options:
         delimiter: '/'
         index: 1
         create: true
     - select:
         kind: CustomResourceDefinition
       fieldPaths:
         - .metadata.annotations.[cert-manager.io/inject-ca-from]
       options:
         delimiter: '/'
         index: 1
         create: true
 - source: # Add cert-manager annotation to the webhook Service
     kind: Service
     version: v1
     name: webhook-service
     fieldPath: .metadata.name # namespace of the service
   targets:
     - select:
         kind: Certificate
         group: cert-manager.io
         version: v1
       fieldPaths:
         - .spec.dnsNames.0
         - .spec.dnsNames.1
       options:
         delimiter: '.'
         index: 0
         create: true
 - source:
     kind: Service
     version: v1
     name: webhook-service
     fieldPath: .metadata.namespace # namespace of the service
   targets:
     - select:
         kind: Certificate
         group: cert-manager.io
         version: v1
       fieldPaths:
         - .spec.dnsNames.0
         - .spec.dnsNames.1
       options:
         delimiter: '.'
         index: 1
         create: true

```

**Add cert-manager command via Makefile**

```
.PHONY: install-cert-manager
install-cert-manager: helm ## Install cert-manager using Helm.
	helm repo add jetstack https://charts.jetstack.io
	helm repo update
	helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.15.0 --set crds.enabled=true

.PHONY: uninstall-cert-manager
uninstall-cert-manager: helm ## Uninstall cert-manager using Helm.
	helm uninstall cert-manager --namespace cert-manager
	kubectl delete namespace cert-manager

```

**Install cert-manager**

```
make install-cert-manager
```

**Add image proxy via Dockerfile**

```
# Build the manager binary
FROM dockerproxy.net/library/golang:1.22 AS builder
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace
# Copy the Go Modules manifests
COPY go.mod go.mod
COPY go.sum go.sum
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
COPY vendor/ vendor/

# Copy the go source
COPY cmd/main.go cmd/main.go
COPY api/ api/
COPY internal/controller/ internal/controller/

# Build
# the GOARCH has not a default value to allow the binary be built according to the host where the command
# was called. For example, if we call make docker-build in a local env which has the Apple Silicon M1 SO
# the docker BUILDPLATFORM arg will be linux/arm64 when for Apple x86 it will be linux/amd64. Therefore,
# by leaving it empty we can ensure that the container and binary shipped on it will have the same platform.
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o manager cmd/main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM gcr.dockerproxy.net/distroless/static:nonroot
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532

ENTRYPOINT ["/manager"]

```

**Deploy Controller**

```
export IMG=ttl.sh/saymololy/controller:1h
make docker-build
make docker-push
make deploy

```

### Test

**Test mutating webhook**

```
apiVersion: blog.mydomain.com/v1
kind: Foo
metadata:
  labels:
    app.kubernetes.io/name: controller
    app.kubernetes.io/managed-by: kustomize
  name: foo-sample
spec:
  image: ttl.sh/saymololy/controller:1h
```

```
kubectl get foo.blog.mydomain.com foo-sample  -o jsonpath="{.spec.replicas}" 

2
```

**Test validating webhook**

```
apiVersion: blog.mydomain.com/v1
kind: Foo
metadata:
  labels:
    app.kubernetes.io/name: controller
    app.kubernetes.io/managed-by: kustomize
  name: foo-sample
spec:
  image: ttl.sh/saymololy/controller:1h
  replicas: 6
```

```
Error from server (Forbidden): error when creating "config/samples/blog_v1_foo.yaml": admission webhook "vfoo.kb.io" denied the request: foo replicas cannot be more than 5
```

## License

Copyright 2024.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

