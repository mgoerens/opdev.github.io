---
title:
date:
description:
categories:
---

The [subreconciler](https://github.com/opdev/subreconciler) library is a
useful addition to a Go operator project. It allows developers to improve
code readability by structuring the reconciliation logic
into "subreconcilers". Each subreconciler is a independent function that
handles a specific task.

In this blog post, we'll take a look at a concrete example how developers can
add the subreconciler library to their operator project and how to make the
most out of it.

# Memached-operator

Implementing the memcached-operator is a common learning exercise when getting
started with Go operators. This is in fact the tutorial of operator-sdk.
We'll first take a look at the code and see how we can transition to
subreconcilers to improve structure and code readability of the
memached-operator.

To follow along, each step of this process is implemented as a separate commit
in
[this repository](https://github.com/opdev/memached-operator-with-subreconcilers/compare/master...subreconcilers-fn-with-request).

# The world before subreconcilers

[Here](https://github.com/opdev/memached-operator-with-subreconcilers/blob/4d6dbacf9f9a19c69483efda5d9e104f62788bde/controllers/memcached_controller.go)
is the starting state of the memcached-operator.

The main Reconcile function is over 200 lines long. It's composed of a few
distinct tasks like adding a finalizer, updating the CR status and
reconciling a Deployment child resource.

It's of course completely fine to keep it as it is, but as the project grows,
this function will grow bigger and bigger and it will get more and more
difficult to keep thinks tidy. In particular, while each task is independent,
they all are implemented in the same function.

So let's break the main reconciler into subreconcilers.

# Step 1: defining how to split the code

We identified 5 distinct tasks that are performed by the main reconcile
function:
* The controller first sets the status of the resource to "Unknown".
* Then it adds a finalizer to the Memcached resource.
* The controller then checks if the resource is marked to be deleted and
  performs cleanup actions before removing the finalizer.
* A child resource is reconciled next. If it doesn't already exist, the
  controller creates a Deployment for Memcached. Otherwise, it checks that
  the Deployment is correctly configured.
* Finally the controller updates the resource status with information on the
  Memcached deployment.

[Here](https://github.com/opdev/memached-operator-with-subreconcilers/commit/093584533508f95b61ec85849c44837167a78bdb)
is how we aim at splitting the code.

We will create a subreconciler for each of these steps.

# Step 2: creating our first subreconciler

[This commit](https://github.com/opdev/memached-operator-with-subreconcilers/commit/769fe39b8a4d68ff60b10c094049d905d6b2a4a1)
demonstrates the full implementation of our first subreconciler,
`setStatusToUnknown`.

This is the actual task that we want to offload to a dedicated subreconciler:

```go
  // Let's just set the status as Unknown when no status are available
  if memcached.Status.Conditions == nil || len(memcached.Status.Conditions) == 0 {
    meta.SetStatusCondition(&memcached.Status.Conditions, metav1.Condition{Type: typeAvailableMemcached, Status: metav1.ConditionUnknown, Reason: "Reconciling", Message: "Starting reconciliation"})
    if err = r.Status().Update(ctx, memcached); err != nil {
      log.Error(err, "Failed to update Memcached status")
      return ctrl.Result{}, err
    }
```

First we create an empty subreconciler that follows the function type
`subreconciler.FnWithRequest`.

The subreconciler library comes with a set of "flow control functions", which
indicates to the main reconciler (i.e. the Reconcile() function) whether
reconciliation should continue, halt, throw an error, requeue, etc.

> It is necessary to add a
> `return subreconciler.ContinueReconciling()` statement at the end of each
> subreconciler to notify the main reconciler to continue the reconciliation
> logic.

```go
// setStatusToUnknown is a function of type subreconciler.FnWithRequest
func (r *MemcachedReconciler) setStatusToUnknown(ctx context.Context, req ctrl.Request) (*ctrl.Result, error) {
  return subreconciler.ContinueReconciling()
}
```

The Memached resource is **not** passed from the main reconciler to the
subreconciler. This is a design decision as it is considered best practice to
treat subreconcilers as independent "reconcilers". It is therefore necessary
to fetch the latest version of the Memcached resource at the beginning of our
(sub)reconciliation.

Our subreconciler becomes:

```go
// setStatusToUnknown is a function of type subreconciler.FnWithRequest
func (r *MemcachedReconciler) setStatusToUnknown(ctx context.Context, req ctrl.Request) (*ctrl.Result, error) {
  log := log.FromContext(ctx)
  memcached := &cachev1alpha1.Memcached{}

  // Fetch the latest version of the memcached resource
  if err := r.Get(ctx, req.NamespacedName, memcached); err != nil {
    if apierrors.IsNotFound(err) {
      // If the custom resource is not found then, it usually means that it was deleted or not created
      // In this way, we will stop the reconciliation
      log.Info("memcached resource not found. Ignoring since object must be deleted")
      return subreconciler.DoNotRequeue()
    }
    // Error reading the object - requeue the request.
    log.Error(err, "Failed to get memcached")
    return subreconciler.RequeueWithError(err)
  }

  return subreconciler.ContinueReconciling()
```

> Here we see two new flow control functions:
> - `return subreconciler.DoNotRequeue()` indicates to the main reconciler
>   that the reconciliation should stop, and that the request should not be
>   requeued. It essentially replaces a `return ctrl.Result{}, nil` in the
>   main reconciler.
> - `return subreconciler.RequeueWithError(err)` indicates to the main
>   reconciler that an error occured and should be logged, and to requeue the
>   request. It essentially replaces a `return ctrl.Result{}, err` in the
>   main reconciler.

We are now only missing the actual reconciliation logic: setting the Status
to "Unknown".

```go
// setStatusToUnknown is a function of type subreconciler.FnWithRequest
func (r *MemcachedReconciler) setStatusToUnknown(ctx context.Context, req ctrl.Request) (*ctrl.Result, error) {
  log := log.FromContext(ctx)
  memcached := &cachev1alpha1.Memcached{}

  // Fetch the latest version of the memcached resource
  if err := r.Get(ctx, req.NamespacedName, memcached); err != nil {
    if apierrors.IsNotFound(err) {
      // If the custom resource is not found then, it usually means that it was deleted or not created
      // In this way, we will stop the reconciliation
      log.Info("memcached resource not found. Ignoring since object must be deleted")
      return subreconciler.DoNotRequeue()
    }
    // Error reading the object - requeue the request.
    log.Error(err, "Failed to get memcached")
    return subreconciler.RequeueWithError(err)
  }

  // Let's just set the status as Unknown when no status are available
  if memcached.Status.Conditions == nil || len(memcached.Status.Conditions) == 0 {
    meta.SetStatusCondition(&memcached.Status.Conditions, metav1.Condition{Type: typeAvailableMemcached, Status: metav1.ConditionUnknown, Reason: "Reconciling", Message: "Starting reconciliation"})
    if err := r.Status().Update(ctx, memcached); err != nil {
      log.Error(err, "Failed to update Memcached status")
      return subreconciler.RequeueWithError(err)
    }
  }

  return subreconciler.ContinueReconciling()
}
```

And, *voila*, we created our first subreconciler !

Now, let's see how to call it from the main reconciliation loop. This is a two
step process.

First we have to call the subreconciler:

```go
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  [...]

  // Run the setStatusToUnknown subreconciler
  result, err := r.setStatusToUnknown(ctx, req)

  [...]
```

Then we need to check the values returned by the subreconciler to either
requeue, halt or continue the reconciliation. The subreconciler library
provides a function for exactly this purpose:

`ShouldHaltOrRequeue returns true if reconciler result is not nil or the err
is not nil`

If we are indeed in a situation that calls for a requeue or a halt of the
reconciliation, we `return` from the main reconcile function. Here we use
the handy `Evaluate` function from the subreconciler library:

`Evaluate returns the actual reconcile struct and error. Wrap helpers in this
when returning from within the top-level Reconciler.`

```go
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  [...]

  // Run the setStatusToUnknown subreconciler
  result, err := r.setStatusToUnknown(ctx, req)

  // Stop the reconciliation if needed.
  if subreconciler.ShouldHaltOrRequeue(result, err) {
    return subreconciler.Evaluate(result, err)

  [...]
}
```

This can also be written:

```go
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  [...]

  // Run the setStatusToUnknown subreconciler
  if r, err := r.setStatusToUnknown(ctx, req); subreconciler.ShouldHaltOrRequeue(r, err) {
    return subreconciler.Evaluate(r, err)
  }

  [...]
}
```

> Note that because we don't pass the the Memcached resource to
> subreconcilers, the main reconcile function doesn't have the latest version
> of our resource. This will lead to issues as the rest of the Reconcile()
> function works on an outdated version of the Memacached resource, and
> the Kubernetes API will reject attempts to modify the resource. We can
> solve this issue in two ways:
> - Force a requeue after modifying the resource in the subreconciler.
> - Ensure that we work on the latest version of the resource in the main
>   reconciler by fetching it again after having called the subreconciler.
> 
> We are going with the second option for now. We'll see how that becomes
> obsolete after the next step.

# Step 3: Creating other subreconcilers

We now repeat the same process for the other 4 other parts of the reconcile
function that we previously identified. We create 4 new subreconcilers:
- `addFinalizer`
- `handleDelete`
- `reconcileDeployment`
- `updateStatus`

[Here is the implementation of this step]
(https://github.com/opdev/memached-operator-with-subreconcilers/commit/b20e4fe18972d3a7004a45cac31d472ab4166a66)

> Again, two new flow control functions are introduced in this step:
> - `return subreconciler.Requeue()` forces a requeue not caused by an error.
>   This is the equivalent of `return ctrl.Result{Requeue: true}, nil` in the
>   main reconciler.
> - `return subreconciler.RequeueWithDelay(time.Minute)` requeues the request
>   after a delay. It replaces a
>   `return ctrl.Result {RequeueAfter: time.Minute}, nil` statement.

Note that each subreconciler starts with the same logic: fetching the latest
version of the Memcached instance. It is therefore not necessary to requeue
each time a subreconciler update this object, as it is guaranteed that those
changes are will be picked up by other subreconcilers.

# Step 4: Some refactoring

Arguably this can be done along the way, but for clarity we kept this step
separated. There are a couple of things we can do:

First we can put all our subreconcilers in a list and loop over it to call
subreconcilers. Our Reconcile() function becomes even shorter:

```go
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  [...]

  // The list of subreconcilers for Memached
  subreconcilersForMemcached := []subreconciler.FnWithRequest{
    r.setStatusToUnknown,
    r.addFinalizer,
    r.handleDelete,
    r.reconcileDeployment,
    r.updateStatus,
  }

  // Run all subreconcilers sequentially
  for _, f := range subreconcilersForMemcached {
    if r, err := f(ctx, req); subreconciler.ShouldHaltOrRequeue(r, err) {
      return subreconciler.Evaluate(r, err)
    }
  }

  [...]
}
```

See [this commit](
https://github.com/opdev/memached-operator-with-subreconcilers/commit/1fa1894eef5f5acfc5786faef524fe18290291ab)
for the implementation of this first improvement.

Second, we can use the subreconciler library in the `return` statement of
`Reconcile()` for consistency:

```go
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  [...]

  return subreconciler.Evaluate(subreconciler.DoNotRequeue())
}
```

See how it is done in [this commit](
https://github.com/opdev/memached-operator-with-subreconcilers/commit/45e45a6b9a28c24c7d21e710b3c4f0d4146869bc).

Finally, we have noticed that each of our subreconciler starts with the same:
getting the latest version of the Memcached instance. We can create a small
function which does just that:

```go
func (r *MemcachedReconciler) getLatestMemached(ctx context.Context, req ctrl.Request, memcached *cachev1alpha1.Memcached) (*ctrl.Result, error) {
  log := log.FromContext(ctx)

  // Fetch the latest version of the memcached resource
  if err := r.Get(ctx, req.NamespacedName, memcached); err != nil {
    if apierrors.IsNotFound(err) {
      // If the custom resource is not found then, it usually means that it was deleted or not created
      // In this way, we will stop the reconciliation
      log.Info("memcached resource not found. Ignoring since object must be deleted")
      return subreconciler.DoNotRequeue()
    }
    // Error reading the object - requeue the request.
    log.Error(err, "Failed to get memcached")
    return subreconciler.RequeueWithError(err)
  }

  return subreconciler.ContinueReconciling()
}
```

And call this function in our subreconcilers as follow:

```go
func (r *MemcachedReconciler) setStatusToUnknown(ctx context.Context, req ctrl.Request) (*ctrl.Result, error) {
  [...]

  memcached := &cachev1alpha1.Memcached{}

  // Fetch the latest Memcached
  // If this fails, bubble up the reconcile results to the main reconciler
  if r, err := r.getLatestMemached(ctx, req, memcached); subreconciler.ShouldHaltOrRequeue(r, err) {
    return r, err
  }

  [...]
}
```

In a traditional approach (without subreconcilers), the main reconciler first
gets the latest version of Memcached. This acts as a validation that the CRD
as been correctly installed and that the Memcached instance exists.

This tasks is now redundant as this is effectively done at the start of each
subreconciler.

For an implementation of this improvement in the memcached operator, see
[this commit]
(https://github.com/opdev/memached-operator-with-subreconcilers/commit/b4ba2ea4e7376edb9d0e809bf4813b0fff1c89cf).

# Final result

[Here is how the reconciliation logic looks like]
(https://github.com/opdev/memached-operator-with-subreconcilers/blob/b4ba2ea4e7376edb9d0e809bf4813b0fff1c89cf/controllers/memcached_controller.go)
now that we have introduced the subreconciler library into the memached
operator project.

We can see that the structure of our code has been significantly improved.
Each task that is performed as part of the reconciliation of a Memcached
instance is logically separated and can be developed independently from the
other. The Reconcile() function went from being over 200 lines long to less
the 20.

As a tradeoff, we have to ensure, at the beginning of each subreconciler, that
we are working with the latest version of the Memcached instance. This
increases the amount of network calls and the load on the Kubernetes API.
Also, the file `memcached_controller.go` grew from 450 lines to 529.
