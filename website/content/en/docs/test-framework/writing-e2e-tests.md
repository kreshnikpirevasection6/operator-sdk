---
title: Writing E2E Tests
linkTitle: Writing E2E Tests
weight: 1
---
# Using the Operator SDK's Test Framework to Write E2E Tests

End-to-end tests are essential to ensure that an operator works
as intended in real-world scenarios. The Operator SDK includes a testing
framework to make writing tests simpler and quicker by removing boilerplate
code and providing common test utilities. The Operator SDK includes the
test framework as a library under `pkg/test` and the e2e tests are written
as standard go tests.

## Components

The test framework includes a few components. The most important to talk
about are Framework and Context.

### Framework

[Framework][framework-link] contains all global variables, such as the kubeconfig, kubeclient,
scheme, and dynamic client (provided via the controller-runtime project).
It is initialized by the MainEntry function and can be used anywhere in the tests.

**Note:** several required arguments are initialized and added by `MainEntry()`. Do not attempt to
use `testing.M` directly.

### Context

[Context][context-link] is a local context that stores important information for each test, such
as the namespace for that test and the cleanup functions. By handling
namespace and resource initialization through Context, we can make sure that all
resources are properly handled and removed after the test finishes.

## Walkthrough: Writing Tests

In this section, we will be walking through writing the e2e tests of the sample
[memcached-operator][memcached-sample].

### Main Test

The first step to writing a test is to create the `main_test.go` file. The `main_test.go`
file simply calls the test framework's main entry that sets up the framework and then
starts the tests. It should be pretty much identical for all operators. This is what it
looks like for the memcached-operator:

```go
package e2e

import (
    "testing"

    f "github.com/operator-framework/operator-sdk/pkg/test"
)

func TestMain(m *testing.M) {
    f.MainEntry(m)
}
```

### Individual Tests

In this section, we will be designing a test based on the [memcached_test.go][memcached-test-link] file
from the [memcached-operator][memcached-sample] sample.

#### 1. Import the framework

Once MainEntry sets up the framework, it runs the remainder of the tests. First, make
sure to import `testing`, the operator-sdk test framework (`pkg/test`) as well as your operator's libraries:

```go
import (
    "testing"

    cachev1alpha1 "github.com/operator-framework/operator-sdk-samples/go/memcached-operator/pkg/apis/cache/v1alpha1"
    "github.com/operator-framework/operator-sdk-samples/go/memcached-operator/pkg/apis"

    framework "github.com/operator-framework/operator-sdk/pkg/test"
)
```

#### 2. Register types with framework scheme

The next step is to register your operator's scheme with the framework's dynamic client.
To do this, pass the CRD's `AddToScheme` function and its List type object to the framework's
[AddToFrameworkScheme][scheme-link] function. For our example memcached-operator, it looks like this:

```go
memcachedList := &cachev1alpha1.MemcachedList{}
err := framework.AddToFrameworkScheme(apis.AddToScheme, memcachedList)
if err != nil {
    t.Fatalf("failed to add custom resource scheme to framework: %v", err)
}
```

We pass in the CR List object `memcachedList` as an argument to `AddToFrameworkScheme()` because
the framework needs to ensure that the dynamic client has the REST mappings to query the API
server for the CR type. The framework will keep polling the API server for the mappings and
timeout after 5 seconds, returning an error if the mappings were not discovered in that time.

#### 3. Setup the test context and resources

The next step is to create a Context for the current test and defer its cleanup function:

```go
ctx := framework.NewContext(t)
defer ctx.Cleanup()
```

Now that there is a `Context`, the test's Kubernetes resources (specifically the test namespace,
Service Account, RBAC, and Operator deployment in `local` testing; just the Operator deployment
in `cluster` testing) can be initialized:

```go
err := ctx.InitializeClusterResources(&framework.CleanupOptions{TestContext: ctx, Timeout: cleanupTimeout, RetryInterval: cleanupRetryInterval})
if err != nil {
    t.Fatalf("failed to initialize cluster resources: %v", err)
}
```

The `InitializeClusterResources` function uses the custom `Create` function in the framework client to create the resources provided
in your namespaced manifest. The custom `Create` function use the controller-runtime's client to create resources and then
creates a cleanup function that is called by `ctx.Cleanup` which deletes the resource and then waits for the resource to be
fully deleted before returning. This is configurable with `CleanupOptions`. For info on how to use `CleanupOptions` see
[this section](#how-to-use-cleanup).

If you want to make sure the operator's deployment is fully ready before moving onto the next part of the
test, the `WaitForOperatorDeployment` function from [e2eutil][e2eutil-link] (in the sdk under `pkg/test/e2eutil`) can be used:

```go
// get namespace
namespace, err := ctx.GetOperatorNamespace()
if err != nil {
    t.Fatal(err)
}
// get global framework variables
f := framework.Global
// wait for memcached-operator to be ready
err = e2eutil.WaitForOperatorDeployment(t, f.KubeClient, namespace, "memcached-operator", 1, time.Second*5, time.Second*30)
if err != nil {
    t.Fatal(err)
}
```

#### 4. Write the test specific code

Since the controller-runtime's dynamic client uses go contexts, make sure to import the go context library.
In this example, we imported it as `goctx`:

##### <a id="how-to-use-cleanup"></a>How to use the Framework Client `Create`'s `CleanupOptions`

The test framework provides `Client`, which exposes most of the controller-runtime's client unmodified, but the `Create`
function has added functionality to create cleanup functions for these resources as well. To manage how cleanup
is handled, we use a `CleanupOptions` struct. Here are some examples of how to use it:

```go
// Create with no cleanup
Create(goctx.TODO(), exampleMemcached, &framework.CleanupOptions{})
Create(goctx.TODO(), exampleMemcached, nil)

// Create with cleanup but no polling for resources to be deleted
Create(goctx.TODO(), exampleMemcached, &framework.CleanupOptions{TestContext: ctx})

// Create with cleanup and polling wait for resources to be deleted
Create(goctx.TODO(), exampleMemcached, &framework.CleanupOptions{TestContext: ctx, Timeout: timeout, RetryInterval: retryInterval})
```

This is how we can create a custom memcached custom resource with a size of 3:

```go
// create memcached custom resource
exampleMemcached := &cachev1alpha1.Memcached{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "example-memcached",
        Namespace: namespace,
    },
    Spec: cachev1alpha1.MemcachedSpec{
        Size: 3,
    },
}
err = f.Client.Create(goctx.TODO(), exampleMemcached, &framework.CleanupOptions{TestContext: ctx, Timeout: time.Second * 5, RetryInterval: time.Second * 1})
if err != nil {
    return err
}
```

Now we can check if the operator successfully worked. In the case of the memcached operator, it should have
created a deployment called "example-memcached" with 3 replicas. To check, we use the `WaitForDeployment` function, which
is the same as `WaitForOperatorDeployment` with the exception that `WaitForOperatorDeployment` will skip waiting
for the deployment if the test is run locally and the `--up-local` flag is set; the `WaitForDeployment` function always
waits for the deployment:

```go
// wait for example-memcached to reach 3 replicas
err = e2eutil.WaitForDeployment(t, f.KubeClient, namespace, "example-memcached", 3, time.Second*5, time.Second*30)
if err != nil {
    return err
}
```

We can also test that the deployment scales correctly when the CR is updated:

```go
err = f.Client.Get(goctx.TODO(), types.NamespacedName{Name: "example-memcached", Namespace: namespace}, exampleMemcached)
if err != nil {
    return err
}
exampleMemcached.Spec.Size = 4
err = f.Client.Update(goctx.TODO(), exampleMemcached)
if err != nil {
    return err
}

// wait for example-memcached to reach 4 replicas
err = e2eutil.WaitForDeployment(t, f.KubeClient, namespace, "example-memcached", 4, time.Second*5, time.Second*30)
if err != nil {
    return err
}
```

Once the end of the function is reached, the Context's cleanup
functions will automatically be run since they were deferred when the Context was created.

## Running the Tests

To make running the tests simpler, the `operator-sdk` CLI tool has a `test` subcommand that can configure
default test settings, such as locations of your global resource manifest file (by default
`deploy/crd.yaml`) and your namespaced resource manifest file (by default `deploy/service_account.yaml` concatenated with
`deploy/rbac.yaml` and `deploy/operator.yaml`), and allows the user to configure runtime options.

To run the tests, run the `operator-sdk test local` command in your project root and pass the location of the tests
as an argument. You can use `--help` to view the other configuration options and use `--go-test-flags` to pass in arguments to `go test`. Here is an example command:

```shell
$ operator-sdk test local ./test/e2e --go-test-flags "-v -parallel=2"
```

### Image Flag

If you wish to specify a different operator image than specified in your `operator.yaml` file (or a user-specified
namespaced manifest file), you can use the `--image` flag:

```shell
$ operator-sdk test local ./test/e2e --image quay.io/example/my-operator:v0.0.2
```

### Namespace Flag

If you wish to run all the tests in 1 namespace (which also forces `-parallel=1`), you can use the `--namespace` flag:

```shell
$ kubectl create namespace operator-test
$ operator-sdk test local ./test/e2e --operator-namespace operator-test
```

### Up-Local Flag

To run the operator itself locally during the tests instead of starting a deployment in the cluster, you can use the
`--up-local` flag. This mode will still create global resources, but by default will not create any in-cluster namespaced
resources unless the user specifies one through the `--namespaced-manifest` flag.

**NOTE**: The `--up-local` flag requires the `--operator-namespace` flag and the command will NOT create the namespace. Then, be sure that you are specifying a valid namespace.

```shell
$ kubectl create namespace operator-test
$ operator-sdk test local ./test/e2e --operator-namespace operator-test --up-local
```

### No-Setup Flag

If you would prefer to create the resources yourself and skip resource creation, you can use the `--no-setup` flag:
```shell
$ kubectl create namespace operator-test
$ kubectl create -f deploy/crds/cache.example.com_memcacheds_crd.yaml
$ kubectl create -f deploy/service_account.yaml --namespace operator-test
$ kubectl create -f deploy/role.yaml --namespace operator-test
$ kubectl create -f deploy/role_binding.yaml --namespace operator-test
$ kubectl create -f deploy/operator.yaml --namespace operator-test
$ operator-sdk test local ./test/e2e --operator-namespace operator-test --no-setup
```

### Test Permissions

Executing e2e tests requires the permission to access, create, and delete resources on your cluster. Depending on what kind of Kubernetes cluster
you are using, this may require some manual setup. For example, OpenShift users are not created with cluster-admin access by default, so you would have
to manually add permissions to access these resources.

The simplest way to accomplish this is to bind the cluster-admin Cluster Role to the Service Account you will run the test under. 
If you are unable or unwilling to grant such access, a more limited permission set can be created and bound to your Service Account.
A good place to start would be the Role bound to your operator itself, such as [this role for the memcached operator example][memcached-role].
In addition, you might have to create a Cluster Role to allow your tests to create namespaces, like so:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: testuser
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - create
  - delete
  - get
  - list
  - watch
  - update
```

Note that this isn't an exhaustive permission set, and the e2e tests you write might require more or less access.

For more documentation on the `operator-sdk test local` command, see the [SDK CLI Reference][cli-test-local] doc.

### Skip-Cleanup-Error Flag

If the tests encounter an error, it is possible to tell the framework not to delete the resources. This behavior is enabled with the `--skip-cleanup-error` flag:

```shell
$ operator-sdk test local ./test/e2e --skip-cleanup-error
```

This is useful if after the error happens, you need to inspect the resources that were created in the test or if you have automated scripts that download all the logs from the pods at the end of the test run.

**NOTE**: The created resources will be deleted if the tests pass.

### Running Go Test Directly (Not Recommended)

For advanced use cases, it is possible to run the tests via `go test` directly. As long as all flags defined
in [MainEntry][main-entry-link] are declared, the tests will run correctly. Running the tests directly with missing flags
will result in undefined behavior. This is an example `go test` equivalent to the `operator-sdk test local` example above:

```shell
# Combine service_account, role, role_binding, and operator manifests into namespaced manifest
$ cp deploy/service_account.yaml deploy/namespace-init.yaml
$ echo -e "\n---\n" >> deploy/namespace-init.yaml
$ cat deploy/role.yaml >> deploy/namespace-init.yaml
$ echo -e "\n---\n" >> deploy/namespace-init.yaml
$ cat deploy/role_binding.yaml >> deploy/namespace-init.yaml
$ echo -e "\n---\n" >> deploy/namespace-init.yaml
$ cat deploy/operator.yaml >> deploy/namespace-init.yaml
# Run tests
$ go test ./test/e2e/... -root=$(pwd) -kubeconfig=$HOME/.kube/config -globalMan deploy/crds/cache.example.com_apps_crd.yaml -namespacedMan deploy/namespace-init.yaml -v -parallel=2
```

## Manual Cleanup

While the test framework provides utilities that allow the test to automatically be cleaned up when done,
it is possible that an error in the test code could cause a panic, which would stop the test
without running the deferred cleanup. To clean up manually, you should check what namespaces currently exist
in your cluster. You can do this with `kubectl`:

```shell
$ kubectl get namespaces

Example Output:
NAME                                            STATUS    AGE
default                                         Active    2h
kube-public                                     Active    2h
kube-system                                     Active    2h
main-1534287036                                 Active    23s
memcached-memcached-group-cluster-1534287037    Active    22s
memcached-memcached-group-cluster2-1534287037   Active    22s
```

The names of the namespaces will be either start with `main` or with the name of the tests and the suffix will
be a Unix timestamp (number of seconds since January 1, 1970 00:00 UTC). Kubectl can be used to delete these
namespaces and the resources in those namespaces:

```shell
$ kubectl delete namespace main-153428703
```

Since the CRD is not namespaced, it must be deleted separately. Clean up the CRD created by the tests using the CRD manifest `deploy/crd.yaml`:

```shell
$ kubectl delete -f deploy/crds/cache.example.com_memcacheds_crd.yaml
```

[memcached-sample]:https://github.com/operator-framework/operator-sdk-samples/tree/master/go/memcached-operator
[framework-link]:https://github.com/operator-framework/operator-sdk/blob/master/pkg/test/framework.go
[context-link]:https://github.com/operator-framework/operator-sdk/blob/master/pkg/test/context.go
[e2eutil-link]:https://github.com/operator-framework/operator-sdk/tree/master/pkg/test/e2eutil
[memcached-test-link]:https://github.com/operator-framework/operator-sdk-samples/blob/master/go/memcached-operator/test/e2e/memcached_test.go
[scheme-link]:https://github.com/operator-framework/operator-sdk/blob/master/pkg/test/framework.go#L109
[cli-test-local]: /docs/cli/operator-sdk_test_local
[main-entry-link]:https://github.com/operator-framework/operator-sdk/blob/master/pkg/test/main_entry.go#L25
[memcached-role]:https://github.com/operator-framework/operator-sdk-samples/blob/master/go/memcached-operator/deploy/role.yaml
