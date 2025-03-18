1. Understanding Ingress Components

Ingress Controller:
Role: The ingress controller is the engine that executes the ingress rules by translating them and routing the traffic accordingly.
Analogy: It functions similarly to a load balancer with forwarding rules, but it requires the ingress object to define those rules.
Deployment: Typically deployed as a pod within the cluster, often through a Deployment or DaemonSet.
Popular Implementations: Nginx and HAProxy are the most commonly used ingress controllers.


Ingress Resource/Object:
Role: The ingress resource is the Kubernetes object that defines the routing rules for the ingress controller.
Interaction: The ingress resource communicates with the ingress controller, instructing it on how to route incoming traffic.

---------------------
2. How Ingress Works in Kubernetes
Step 1: Deploy the Ingress Controller:

Installation: The first step is to install the ingress controller within the cluster. This is done by deploying it as a pod using a Deployment or DaemonSet object.
Inside-Cluster Deployment: The ingress controller is usually deployed inside the cluster, making it easier to manage and configure.
Outside-Cluster Deployment (Optional):
Advanced Setup: Some ingress controllers can be deployed outside the cluster, but this requires additional setup, such as configuring routing through BGP and BIRD.
Use Case: This option is typically used for complex networking environments but is less common for standard deployments.

Step 2: Create the Ingress Resource:

Configuration: After the controller is up and running, the next step is to create the ingress resource. This resource defines the rules that will be applied to incoming traffic.
Manifest File: The ingress resource is defined using a Kubernetes manifest file, similar to other Kubernetes objects.
Updating the Controller: Once created, the ingress resource updates the ingress controller with the rules, enabling it to route traffic according to the specified paths, hosts, and other criteria.

---------------------
3. Practical Example:
Nginx Ingress Controller: The traditional example of deploying an ingress controller is the Nginx ingress controller. It is widely used and well-documented, making it a common choice for Kubernetes clusters.
Manifest File Example: The ingress resource is typically configured in a YAML manifest file, where you can define host-based or path-based routing rules, SSL settings, and more.



apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80

1. Ingress Manifest Example Overview
API Version: apiVersion: networking.k8s.io/v1
This specifies the API version used for the ingress object, ensuring compatibility with Kubernetes.
Kind: kind: Ingress
Defines the type of Kubernetes object, in this case, an ingress resource.
Metadata:
Name: name: ingress-wildcard-host
Specifies the name of the ingress resource for identification within the cluster.

2. Ingress Rules Configuration
Rules Section:

Hosts: The ingress manifest defines different rules for specific hosts.
Host 1: host: "foo.bar.com"
Traffic destined for this host will be routed according to the paths defined under it.
Host 2: host: "*.foo.com"
A wildcard is used to match any subdomain under foo.com, providing flexibility in routing.
Paths:

Path Type: pathType: Prefix
This path type indicates that the path rule should match based on the prefix of the URL path.
Path Matching:
Host 1 Path: path: "/bar"
Requests to foo.bar.com/bar will be routed as defined.
Host 2 Path: path: "/foo"
Requests to any subdomain of foo.com with the path /foo will be routed as defined.
Backend Service and Port:

Host 1:
Service: name: service1
Port: number: 80
Traffic matching foo.bar.com/bar is forwarded to the service1 on port 80.
Host 2:
Service: name: service2
Port: number: 80
Traffic matching *.foo.com/foo is forwarded to the service2 on port 80.
----------

3. How Traffic Is Routed
Traffic Flow:

When a request with the host foo.bar.com and path /bar hits the ingress controller, it is routed to service1 on port 80.
The service then forwards this traffic to the appropriate pods, handling it as usual within Kubernetes.
Flexibility in Ingress Rules:

Content-Based Routing: Ingress rules provide the flexibility to route traffic based on the content of the HTTP headers, not just the IP and port.
Advanced Load Balancing: You can create advanced load-balancing rules, enabling smarter and more precise traffic management within your cluster.
-----------------

4. Ingress Controller Exposure

Challenge:
Ingress controllers are typically deployed inside the cluster, so external traffic needs a way to reach them.
Solution: LoadBalancer Service:
Role: A LoadBalancer service is used to expose the ingress controller to the outside world.
Single IP for Multiple Services: The LoadBalancer service provides a single external-facing IP that directs incoming traffic to the ingress controller.
Routing Mechanism: The ingress controller then decides, based on the defined rules, how to route the traffic to the appropriate backend services within the cluster.
-----------------

5. Traffic Flow Path
Client 🡪 LoadBalancer 🡪 Ingress Controller 🡪 Other Services 🡪 Pods
Client: Sends a request to the external IP provided by the LoadBalancer service.
LoadBalancer: Receives the traffic and forwards it to the ingress controller inside the cluster.
Ingress Controller: Applies the ingress rules to determine which service should handle the traffic.
Service: The selected service forwards the traffic to the appropriate pod(s) for processing.
This breakdown explains the key components of the ingress manifest, how the ingress controller manages traffic, and the role of a LoadBalancer service in exposing the ingress controller to the outside world. It emphasizes the flexibility and advanced routing capabilities that ingress rules provide within a Kubernetes environment.
