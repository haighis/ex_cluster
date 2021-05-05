# ExCluster

ExCluster is a demo application built with Elixir and OTP to show how to integrate
Distributed Elixir on Kubernetes with Horde, LibCluster, and Distillery. The application
mimics a service in charge of processing orders, a GenServer models a order and
stores in it's state a list of integers as order contents. All state is kept in-memory,
and using Distributed Elixir and Horde when a Node goes down gracefully, processes
are restarted throughout the remaining Cluster Nodes (with process state transferred).
This is integrated on Kubernetes via a HeadlessService setup with LibCluster.

## Running Locally - Single Node

You can run the application locally simply via `iex -S mix`, this will run a single Node
without any clustering.

## Running Locally - MultiNode

To run a Cluster locally, run the application multiple times with unique Node names and a
consistent cookie specified via `ERL_FLAGS`, for example this will run a 3 Node cluster:

```
# Terminal 1
$ ERL_FLAGS="-name count1@127.0.0.1 -setcookie cookie" NODES="count2@127.0.0.1,count3@127.0.0.1" iex -S mix

# Terminal 2
$ ERL_FLAGS="-name count2@127.0.0.1 -setcookie cookie" NODES="count1@127.0.0.1,count3@127.0.0.1" iex -S mix

# Terminal 3
$ ERL_FLAGS="-name count3@127.0.0.1 -setcookie cookie" NODES="count1@127.0.0.1,count2@127.0.0.1" iex -S mix
```

## Running on `minikube`

To run on `minikube`, start it up and then deploy the charts:

```
$ docker build -t ex_cluster:local .
$ eval $(minikube docker-env)
$ kubectl create -f k8s/service-headless.yaml
$ kubectl create -f k8s/deployment.yaml
$ kubectl get pods

Exec onto one, attach to the console, and create a few orders:
$ kubectl exec -it ex-cluster-5b45c8577b-jltm9 /bin/bash
[in the pod shell]$ _build/prod/rel/ex_cluster/bin/ex_cluster remote_console
[should now be attached to iex on the Pod]
iex(ex_cluster@172.17.0.6)1> Horde.Supervisor.start_child(ExCluster.OrderSupervisor, { ExCluster.Order, "John" })
iex(ex_cluster@172.17.0.6)3> ExCluster.Order.add("John", [1,2,3])
:ok
iex(ex_cluster@172.17.0.6)4> ExCluster.Order.contents("John")
[1, 2, 3]

Terminate the Pod where the order is running:
$ kubectl delete pod ex-cluster-5b45c8577b-jltm9

A new Pod should have been automatically spun up, with the terminated Pod now going down. Exec onto one of the two running Pods, and check on the order:
$ kubectl get pods
NAME                          READY     STATUS        RESTARTS   AGE
ex-cluster-5b45c8577b-8cg9c   1/1       Running       0          3m
ex-cluster-5b45c8577b-d4cm8   1/1       Running       0          5s
ex-cluster-5b45c8577b-fkfj2   1/1       Terminating   0          3m
$ kubectl exec -it ex-cluster-5b45c8577b-d4cm8 /bin/bash
[in the pod shell]$ _build/prod/rel/ex_cluster/bin/ex_cluster remote_console
[should now be attached to iex on the Pod]
iex(ex_cluster@172.17.0.7)1> ExCluster.Order.contents("John")
[1, 2, 3]
There you go, the order was preserved through Pod tear down!

```

You can now get Pods, exec onto one and create orders, kill the Pod, and view the restarted
order process with state still alive in the Cluster.
