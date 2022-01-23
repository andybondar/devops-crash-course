# Deployment Strategies

## Basic Deployment
All nodes in a target environment get simultaneously updated with a new version of service, application, component, etc.

![Basic Deployment](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/basic-deployment-general.png)

In a **Kubernetes** Deployment specification, it is defined with a strategy of type `Recreate`:

```yaml
spec:
  replicas: 4
  strategy:
    type: Recreate
```

![Basic Deployment - Kubernetes](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/basic-deployment-kubernetes.png)

**Pros:**

* Application gets updates quickly and entirely

**Cons:**

* Downtime is inevitable
* Not suitable for Production deployments

## Rolling Deployment
Other names: Rolling Update, Ramped Rollout. All instances are updated incrementally, one by one. Readiness of every updated instance is verified, the deployment proceeds if the updated instance is ready to work.

![Rolling Update](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/rolling-deployment-general.png)

In **Kubernetes,** this type of deployment is defined for a particular Deployment object with a strategy of type `RollingUpdate`:

```yaml
 
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # how many pods can be created over the desired number
      maxUnavailable: 0  # maxUnavailable define how many pods can be unavailable
                         # during the rolling update
```

![Rolling Update](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/rolling-deployment-kubernetes.png)

When the Deployment is used together with a [horizontal pod autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), it is recommended using percentage based values instead of absolute numbers for the [maxSurge](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge) and [maxUnavailable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable).


**Pros:**

* New version is slowly released across instances, its readiness is verified, new instances are not accepting traffic unless they are considered to be ready
* Suitable for stateful apps that can handle rebalancing of the data

**Cons:**

* Both `Rollout` and `Rollback` processes are time-consuming
* Supporting multiple APIs is hard
* No control over traffic


## Blue-Green Deployment
This strategy utilizes 2 identical environments, "Blue" for "Staging" and "Green" for "Production", running different versions of an application. Initially, the "Blue" environment is not exposed to the users. After all QA and User Acceptance tests are passed within the "Blue" environment, user traffic is shifted to the "Green" environment to the "Blue" one.

![Blue-Green Deployment](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/blue-green-deployment-general.png)

In **Kubernetes,** we use 2 Deployments with different `version` labels, running corresponding versions of the Application. A `Service` object acts as a load balancer, and also it starts sending traffic to the Deployment running the newer version of the Application once the `version` label under the `selector:` is changed:

```yaml
apiVersion: v1
kind: Service
metadata:
 name: my-app
 labels:
   app: my-app
spec:
 type: NodePort
 ports:
 - name: http
   port: 8080
   targetPort: 8080

 # Note here that we match both the app and the version.
 # When switching traffic, we update the label “version” with
 # the appropriate value, ie: v2.0.0
 selector:
   app: my-app
   version: v1.0.0
```

![Blue-Green Deployment](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/blue-green-deployment-kubernetes.png)

**Pros:**

* Switchover to a new version is instantaneous, while the new version's readiness is tested prior to the switchover
* No versioning issues, the entire cluster's state is changed in one go

**Cons:**

* Requires double the infrastructure
* Handling stateful applications is problematic


## Canary Deployment
This strategy releases a new version of an application incrementally to a subset of users. The Application instances are updated in small phases. Inbound traffic from the subset of users is routed to the updated instances.

![Canary Deployment - Phases](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/canary-deployment-general.png)

In **Kubernetes,** a Canary Deployment can be implemented using 2 Deployments with common Pod labels, running the old and new versions respectively. We gradually increase the number of replicas with new version and remove the "old version replicas" correspondingly. If there are no issues, we eventually remove all Pods running the old version and replace them with the ones running the new version. If any unacceptable issue occurs, we can seamlessly roll back to the initial state (all replicas with the old version).

![Canary Deployment - Kubernetes](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/canary-deployment-kubernetes.png)

Using this ReplicaSet technique requires spinning up as many Pods as necessary to route the right percentage of traffic from the subset of users. That said, if we want to send 1% of traffic to V2, we need to have one Pod running V2 and 99 Pods running V1. And also we have to change the numbers manually for both Deployments. This is not convenient. We'll use a better managed traffic distribution, so let's take a look at the examples describing how to leverage `Nginx Ingress` or `Istio` and `Horizontal Pod Autoscaler`:

* https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary

**Pros:**

* A new version is released for a subset of users, which allows to decrease a negative impact should anything go wrong.
* Convenient for error and performance monitoring.
* Cheaper than a "Green-Blue Deployment" since it doesn't require two production-grade environments.
* Early and Fast rollback. If anything goes wrong, we have a good chance to notice it early and roll the changes back fast.

**Cons:**

* Involves testing in Production
* "The First Adopters" can get frustrated by learning that they are the ones to face the worst issues. Consider giving the users a chance to opt-in (or opt-out) the new version, releasing the `canary` for the internal users, etc., so you can mitigate this disadvantage.
* Live verification can be hard to configure and time-consuming
* `Canary Deployment` most likely is not a fit if the changes you need to introduce, require altering the database schemas. Backward compatibility must be maintained.


## A/B Testing
In this strategy, different versions of the application run simultaneously as "experiments" and exposed to the subsets of users. Traffic is routed from a particular set of users to a particular version of application. We can target a given pool of user based on the specific parameters, like `Cookie`, `HTTP headers`, `user agent`, etc. It may look like the `Canary Deployment` to some extent, but here is the biggest difference: while `Canary Deployment's` ultimate goal is to update the application instances with a new version everywhere across the environment, the `A/B Testing` is primarily focused on experimentation and exploration, gathering some insights, based on which the decision is made about the most suitable version to be eventually deployed.

![A/B Testing - General picture](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/ab-test-deployment-general.png)

In **Kubernetes**, like for `Canary Deployment`, we can use two (or more) Deployments with common pod labels for each version respectively. But also we can use, for example, `Istio` service mesh to route the HTTP requests to a Pod running a specific version, based on the information in that request's HTTP Header. And we still can use `weight` values to balance load.

![A/B Testing - Kubernetes](https://storage.googleapis.com/devops-crash-course-str-b1-lab-02/deploymemt-strategies/png/ab-test-deployment-kubernetes.png)

**Pros:**

* Allows testing separate features rather than an entire app's version consisting of multiple features, in production
* Which reduces risk of significant impact on customer's business processes while keeping the infrastructure costs fairly low
* We can shape infrastructure, particularly in terms of its capabilities, capacity and costs, by controlling the traffic distribution

**Cons:**

* A/B Testing is experimental in its nature
* Even though the targeted users may voluntarily opt in to try out a new feature, they could get frustrated, and their businesses could get impacted more than you anticipated
* Gathering the insights and configuring the automation of the A/T Testing can be complicated

## Conclusion
Which Deployment Strategy to use is something that depends on the type of your App and your target environment. Here are some options:

* `Basic` or `Rollout` deployment is usually a good fit for `development` and/or `staging` environments
* `Rollout` and `Blue-Green` deployments might be suitable for Production, but proper testing is necessary
* If you are not confident with the new version's stability even after testing, or your App is `mission-critical`, then you should go with a `Canary Deployment`