# Networking — Complete Notes (Part 4 of 5)

> **Part 4 of 5** — covers Sections 22–24 (Ingress, Ingress Annotations and
> rewrite-target, Practice Test — CKA Ingress Networking 1).
> Previous: [kubernetes-networking-notes_part_03.md](kubernetes-networking-notes_part_03.md)
> (Sections 15–21).
> Next: [kubernetes-networking-notes_part_05.md](kubernetes-networking-notes_part_05.md)
> (Sections 25–26 + Quick Revision Checklist).
>
> Source: `~/tf/ep-data/certified-kubernetes-administrator-course/docs/09-Networking/`

---

## 22. Ingress

- Video reference: *Ingress* — https://kodekloud.com/topic/ingress/
- This section covers two distinct concepts:
  - **Ingress Controller** — the actual component/Pod (e.g. an NGINX-based reverse proxy) that watches Ingress resources and does the real HTTP(S) traffic routing.
  - **Ingress Resources** — the Kubernetes API objects that declare the routing *rules* (which host/path goes to which backend Service) for the controller to act on.
- **Note on source material:** unlike several other files in this course export, file 22 contains **no embedded image references** — all content below is extracted directly from the file's prose/YAML, with only a small amount of clearly-labeled general/background context added where the source is terse.
- Background/exam context (general Kubernetes knowledge, not verbatim in this file but implicit in why Ingress exists): a plain `Service` of type `NodePort` requires a separate high-numbered port (30000–32767) per application, and a `LoadBalancer` Service requires a separate cloud load balancer (and IP) per application — expensive and unwieldy at scale. **Ingress** instead gives you a single entry point (one external IP/load balancer) that performs **Layer-7** (HTTP/HTTPS) routing — by **hostname** and/or **URL path** — fanning out to many different backend Services, and can also handle **SSL/TLS termination**, load balancing algorithms, auth, etc., all in one place. Ingress itself does nothing on its own — it is only a set of rules; an **Ingress Controller** (e.g. GCE controller, NGINX ingress controller, Contour, Traefik, HAProxy, Istio) must be deployed in the cluster to actually implement/enforce those rules. A plain Kubernetes cluster does **not** ship with an Ingress Controller by default (unlike a Service, which is core functionality) — you must deploy one yourself (e.g. `ingress-nginx`).

### Ingress Controller

- Deploying an Ingress Controller means deploying a normal set of Kubernetes objects: a **ConfigMap**, a **Deployment**, a **ServiceAccount** (with RBAC), and a **Service** — there is nothing magic about it; it's just a specially-configured NGINX (or other proxy) running as Pods inside the cluster.

### ConfigMap

- The ConfigMap holds NGINX Ingress Controller configuration options (empty/minimal here — populated with specific option keys as needed):
  ```
  kind: ConfigMap
  apiVersion: v1
  metadata:
    name: nginx-configuration
  ```

### Deployment

- The Deployment runs the actual `nginx-ingress-controller` image as a Pod:
  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ingress-controller
  spec:
    replicas: 1
    selector:
      matchLabels:
        name: nginx-ingress
    template:
      metadata:
        labels:
          name: nginx-ingress
      spec:
        serviceAccountName: ingress-serviceaccount
        containers:
          - name: nginx-ingress-controller
            image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
            args:
              - /nginx-ingress-controller
              - --configmap=$(POD_NAMESPACE)/nginx-configuration
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            ports:
              - name: http
                containerPort: 80
              - name: https
                containerPort: 443
  ```
  - `args` passes `--configmap=$(POD_NAMESPACE)/nginx-configuration` so the controller binary knows which ConfigMap (in which namespace) to read its config from.
  - `env` populates `POD_NAME`/`POD_NAMESPACE` from the Downward API (`fieldRef` → `metadata.name` / `metadata.namespace`) — these are then referenced inside `args` via `$(POD_NAMESPACE)`.
  - Exposes container ports `80` (http) and `443` (https).
  - Runs under the `ingress-serviceaccount` ServiceAccount (defined below) rather than the `default` one, since the controller needs elevated permissions to watch Ingress/Service/Endpoint objects cluster-wide.

### ServiceAccount

- A ServiceAccount is required for authentication purposes, along with the correct **Roles**, **ClusterRoles**, and **RoleBindings** (the controller needs API permissions to list/watch Ingress objects, Services, Endpoints, Secrets, etc., across namespaces).
- Create the ingress ServiceAccount:
  ```
  $ kubectl create -f ingress-sa.yaml
  serviceaccount/ingress-serviceaccount created
  ```

### Service Type - NodePort

- The Ingress Controller Pods themselves are exposed to the outside world via a normal `NodePort` (or `LoadBalancer`, in cloud environments) Service — this is the single external entry point mentioned above.
  ```
  # service-Nodeport.yaml

  apiVersion: v1
  kind: Service
  metadata:
    name: ingress
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
    selector:
      name: nginx-ingress
  ```
  - `selector: name: nginx-ingress` matches the `labels: name: nginx-ingress` on the Deployment's Pod template, wiring the Service to the controller Pods.
- Create the service:
  ```
  $ kubectl create -f service-Nodeport.yaml
  ```
- Get the service:
  ```
  $ kubectl get service
  ```

### Ingress Resources

- The **Ingress** resource itself is a separate, much simpler object that just declares routing rules. A minimal Ingress with a single default backend and no rules at all:
  ```
  Ingress-wear.yaml

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingress-wear
  spec:
       backend:
          serviceName: wear-service
          servicePort: 80
  ```
  - **Exam-relevant note:** `apiVersion: extensions/v1beta1` is the **old/deprecated** API version used throughout this lecture (Kubernetes 1.18-era course recording). Later files (24/25) use the current stable `networking.k8s.io/v1` API, which has a **different backend schema** (`backend.service.name` / `backend.service.port.number` instead of `backend.serviceName` / `backend.servicePort`, and requires an explicit `pathType`). Know both forms for the exam, but default to `networking.k8s.io/v1` on any current cluster.
  - With just a single `backend` and no `rules`, **all** traffic hitting this Ingress (regardless of host/path) is sent to `wear-service:80` — it acts like a catch-all default backend.
- Create the ingress resource:
  ```
  $ kubectl create -f Ingress-wear.yaml
  ingress.extensions/ingress-wear created
  ```
- Get the ingress:
  ```
  $ kubectl get ingress
  NAME           CLASS    HOSTS   ADDRESS   PORTS   AGE
  ingress-wear   <none>   *                 80      18s
  ```
  - The **`CLASS`** column here shows `<none>` — background/exam context (general Kubernetes knowledge, tied directly to this observed column): this refers to **`IngressClass`**, a separate cluster-scoped resource (`kind: IngressClass`, group `networking.k8s.io/v1`) that identifies *which* Ingress Controller should handle a given Ingress (useful when multiple controllers run in the same cluster). An Ingress picks its controller either via `spec.ingressClassName: <ingressclass-name>` (current/preferred) or the older `kubernetes.io/ingress.class` annotation (deprecated). `kubectl get ingressclass` lists available classes; one can be marked default via the `ingressclass.kubernetes.io/is-default-class: "true"` annotation on the IngressClass object, in which case Ingresses that don't specify a class use it automatically. `<none>` in the output above simply means no `ingressClassName`/annotation was set on `ingress-wear`.
  - **`HOSTS`** shows `*` (all hosts, i.e. no host restriction) and **`PORTS`** shows `80` (the port the controller listens on for this rule set, not a backend port).

### Ingress Resource - Rules

- **1 Rule and 2 Paths** — a single host (implicitly `*`, all hosts) with two different **paths** routed to two different backend Services:
  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingress-wear-watch
  spec:
    rules:
    - http:
        paths:
        - path: /wear
          backend:
            serviceName: wear-service
            servicePort: 80
        - path: /watch
          backend:
            serviceName: watch-service
            servicePort: 80
  ```
  - **Inferred context (general Ingress traffic-routing concept, no explicit image tag in this file but standard for this exact lecture/example):** this manifest is the textbook "fan-out" pattern — a single external hostname/IP splits traffic by URL path prefix (`/wear` → `wear-service`, `/watch` → `watch-service`), letting one Ingress + one Service (the NodePort/LoadBalancer front door) serve multiple independent backend applications, each on its own path.
- Describe the earlier created ingress resource:
  ```
  $ kubectl describe ingress ingress-wear-watch
  Name:             ingress-wear-watch
  Namespace:        default
  Address:
  Default backend:  default-http-backend:80 (<none>)
  Rules:
    Host        Path  Backends
    ----        ----  --------
    *
                /wear    wear-service:80 (<none>)
                /watch   watch-service:80 (<none>)
  Annotations:  <none>
  Events:
    Type    Reason  Age   From                      Message
    ----    ------  ----  ----                      -------
    Normal  CREATE  23s   nginx-ingress-controller  Ingress default/ingress-wear-watch
  ```
  - Note the automatically-shown **`Default backend: default-http-backend:80`** — this is the controller-wide fallback Service that handles any request **not** matched by any rule/path (it typically just returns an HTTP 404 page). This exact `default-http-backend` reappears as an answer in the file 24 practice test.
- **2 Rules and 1 Path each** — the alternative, **host-based** routing pattern (as opposed to path-based above): each `host` gets its own rule, and each rule's single path (defaulting to `/`) routes to a different Service:
  ```
  # Ingress-wear-watch.yaml

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingress-wear-watch
  spec:
    rules:
    - host: wear.my-online-store.com
      http:
        paths:
        - backend:
            serviceName: wear-service
            servicePort: 80
    - host: watch.my-online-store.com
      http:
        paths:
        - backend:
            serviceName: watch-service
            servicePort: 80
  ```
  - **Exam-relevant contrast:** path-based routing (previous example) splits one hostname into multiple URL prefixes; host-based routing (this example) uses **DNS** (different subdomains all pointing at the same Ingress Controller IP) to split traffic to different backend Services — both are legitimate, commonly-tested patterns and can be combined (multiple hosts, each with multiple paths).

#### References Docs

- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
- https://thenewstack.io/kubernetes-ingress-for-beginners/

---

## 23. Ingress Annotations and rewrite-target

- Video reference: *Ingress Annotations and rewrite-target* — https://kodekloud.com/topic/ingress-annotations-and-rewrite-target/
- Different **Ingress Controllers** support different customization options via **annotations** on the `Ingress` object. The NGINX Ingress Controller has many such annotations; this lecture focuses on one: **`rewrite-target`**.
- Recorded against **Kubernetes Version 1.18**.
- Example manifest:
  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: test-ingress
    namespace: critical-space
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - http:
        paths:
        - path: /pay
          backend:
            serviceName: pay-service
            servicePort: 8282
  ```
  - The annotation lives under `metadata.annotations`, keyed `nginx.ingress.kubernetes.io/rewrite-target`, with value `/`.
  - This Ingress is in the `critical-space` namespace, named `test-ingress`, and routes path `/pay` to `pay-service:8282`.
- **Concept explanation — how `rewrite-target` actually works (general/background nginx-ingress knowledge; not spelled out as prose in this short source file, but this is the standard, well-established mechanics the annotation is famous for on the CKA exam):**
  - Without any rewrite annotation, a request to `http://<ingress-address>/pay` is forwarded to the backend **exactly as received**, i.e. the backend Service (`pay-service`) itself receives a request for path `/pay`. If the backend application actually only understands requests at its root path `/` (which is extremely common — most simple backend apps are written to serve content at `/`, not at whatever prefix the Ingress happens to route from), this results in a 404 from the backend even though the Ingress "matched" correctly.
  - `nginx.ingress.kubernetes.io/rewrite-target: /` tells the NGINX controller to **rewrite** the incoming path before proxying it to the backend: whatever portion of the URL matched the Ingress rule's `path` is replaced by the `rewrite-target` value.
  - **Worked before/after example (general nginx-ingress rewrite-target mechanics, illustrating the exact YAML above):**
    - Before rewrite (what the client requests): `http://<ingress-controller-address>/pay`
    - Without `rewrite-target`, the backend would receive: `http://pay-service:8282/pay` (path preserved) — likely wrong/404 if `pay-service` only serves content at `/`.
    - With `nginx.ingress.kubernetes.io/rewrite-target: /` set, the matched `/pay` prefix is stripped/replaced, and the backend instead receives: `http://pay-service:8282/` — i.e. the request path is rewritten to just `/` before being proxied.
    - More generally, for a path like `/pay/history` matched by rule `path: /pay`, the `/pay` portion is replaced by the rewrite-target (`/`), so the backend receives `/history` (the `/pay` prefix is stripped and replaced by the target, and anything *after* the matched prefix is preserved/appended).
  - This is why `rewrite-target` is critical whenever a single backend app is mounted under a URL prefix via Ingress but the app itself doesn't know about (or expect) that prefix — it lets the Ingress present a clean external path (`/pay`) while the backend keeps serving from its own natural root (`/`).
  - This exact annotation (`nginx.ingress.kubernetes.io/rewrite-target: /`) is reused verbatim in the practice-test manifests in files 24 and 25 (e.g. `ingress-wear-watch`, `test-ingress`), confirming it's the standard, expected annotation for these labs whenever multiple backend paths are routed through one Ingress.

#### Reference Docs

- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- https://kubernetes.github.io/ingress-nginx/examples/
- https://kubernetes.github.io/ingress-nginx/examples/rewrite/
- https://github.com/kubernetes/ingress-nginx/blob/master/docs/troubleshooting.md

---

## 24. Practice Test — CKA Ingress Networking 1

- Link: *Practice Test* — https://kodekloud.com/topic/practice-test-cka-ingress-networking-1/
- **Source-format note:** this docs export only captured the **collapsed `<details>` answer** for each numbered question — the original question text itself was not captured in this export (it lives on the interactive lab page, behind the "Check the Solution" toggle). Below, each step shows the **verbatim answer** exactly as given in the source, plus a **reconstructed/inferred likely question** (clearly marked as such) based on the answer content, the lab's known scenario (an nginx Ingress Controller deployed in one namespace, a `wear`/`watch` retail-style app in another, and a `pay` app in a third), and standard CKA Ingress-lab conventions. Treat the reconstructed question wording as inferred context, not a verbatim quote.

1. *(Inferred question: an initial "explore the environment" / connectivity-check step, e.g. "Try accessing the application to confirm it's initially reachable.")*
   - **Answer (verbatim):** `Ok`

2. *(Inferred question: "What is the name of the namespace where the Ingress Controller is deployed?")*
   - **Answer (verbatim):** `INGRESS-SPACE`
   - How to read it: `kubectl get namespaces` / `kubectl get all -n ingress-space` would show the ingress-controller-related objects (Deployment, Service, ConfigMap, ServiceAccount) living in a namespace literally named `ingress-space`.

3. *(Inferred question: "What is the name of the Ingress Controller (e.g. the Deployment)?")*
   - **Answer (verbatim):** `NGINX-INGRESS-CONTROLLER`
   - How to read it: `kubectl get deploy -n ingress-space` would list a Deployment named `nginx-ingress-controller`.

4. *(Inferred question: "What is the name of the namespace where the actual application (wear/watch services) is deployed?")*
   - **Answer (verbatim):** `APP-SPACE`
   - How to read it: `kubectl get pods,svc -n app-space` would show the application's Pods/Services living in a namespace named `app-space`.

5. *(Inferred question: "How many services exist in the app-space namespace?")*
   - **Answer (verbatim):** `3`
   - How to read it: `kubectl get svc -n app-space` — count the rows (e.g. `wear-service`, `watch-service`/`video-service`, and a third such as `apparels-service` or similar).

6. *(Inferred question: "In which namespace should the new Ingress resource for this app be created?")*
   - **Answer (verbatim):** `APP-SPACE`
   - How to read it: since the backend Services live in `app-space`, the Ingress routing to them must also be created there (Ingress→Service references are namespace-scoped; an Ingress can only route to Services in its **own** namespace).

7. *(Inferred question: "What should the name of this new Ingress resource be?")*
   - **Answer (verbatim):** `INGRESS-WEAR-WATCH`
   - How to read it: the object is expected to be named `ingress-wear-watch`, matching the naming convention used throughout files 22/24/25.

8. *(Inferred question: "What host is configured for this Ingress rule?")*
   - **Answer (verbatim):** `ALL-HOSTS(*)`
   - How to read it: `kubectl describe ingress ingress-wear-watch -n app-space` would show `Host: *` under `Rules`, meaning the rule applies regardless of the requested hostname (path-based routing only, no host-based split — matching the "1 Rule and 2 Paths" pattern from file 22).

9. *(Inferred question: "Which backend Service handles the `/wear` path?")*
   - **Answer (verbatim):** `WEAR-SERVICE`

10. *(Inferred question: "Which path routes to the video/watch backend service?")*
    - **Answer (verbatim):** `/WATCH`
    - How to read it: at this point in the lab, the path is still `/watch` (it gets changed to `/stream` later in step 14).

11. *(Inferred question: "What is the name of the default backend that catches unmatched requests?")*
    - **Answer (verbatim):** `DEFAULT-HTTP-BACKEND`
    - How to read it: matches the `Default backend: default-http-backend:80` line shown in `kubectl describe ingress` output (see file 22).

12. *(Inferred question: "What would a user see if they access a path with no matching Ingress rule?")*
    - **Answer (verbatim):** `404-ERROR-PAGE`
    - How to read it: unmatched requests fall through to the default backend, which serves a generic 404.

13. *(Inferred question: a validation/checkpoint step, e.g. "Confirm the app is reachable at `/wear` and `/watch`.")*
    - **Answer (verbatim):** `OK`

14. *(Task: "The Video service is now called Video, hence let's update Ingress to change the path from `/watch` to `/stream`.")*
    - **Answer (verbatim, command form):**
      ```
      kubectl edit ingress --namespace app-space
      ```
      Change the path from `/watch` to `/stream`.
    - **Answer (verbatim, full manifest form — current `networking.k8s.io`/`extensions` mixed schema as captured, showing the resulting object after the edit):**
      ```yaml
      apiVersion: v1
      items:
      - apiVersion: extensions/v1beta1
        kind: Ingress
        metadata:
          annotations:
            nginx.ingress.kubernetes.io/rewrite-target: /
            nginx.ingress.kubernetes.io/ssl-redirect: "false"
          name: ingress-wear-watch
          namespace: app-space
        spec:
          rules:
          - http:
              paths:
              - backend:
                  serviceName: wear-service
                  servicePort: 8080
                path: /wear
                pathType: ImplementationSpecific
              - backend:
                  serviceName: video-service
                  servicePort: 8080
                path: /stream
                pathType: ImplementationSpecific
        status:
          loadBalancer:
            ingress:
            - {}
      kind: List
      metadata:
        resourceVersion: ""
        selfLink: ""
      ```
    - How to read it: the backend service for this path is actually named `video-service` (not `watch-service`), the path changes from `/watch` to `/stream`, and both paths now carry `pathType: ImplementationSpecific` plus the `rewrite-target: /` and `ssl-redirect: "false"` annotations (needed because the lab environment has no valid TLS cert, so forcing an SSL redirect would break plain-HTTP access).

15. *(Inferred question: validation, e.g. "Confirm `/stream` now serves the video app.")*
    - **Answer (verbatim):** `OK`

16. *(Inferred question: "What happens now if a user tries to access the old `/watch` path?")*
    - **Answer (verbatim):** `404 ERROR PAGE`
    - How to read it: since the rule was **changed** (not added-to) from `/watch` to `/stream`, the old path `/watch` no longer matches any rule and falls through to the default backend's 404 page.

17. *(Inferred question: validation checkpoint.)*
    - **Answer (verbatim):** `OK`

18. *(Task: "A new `food` app needs to be exposed at `/eat` via the same Ingress — add a new path entry.")*
    - **Answer (verbatim, instruction form):**
      Run the command `kubectl edit ingress --namespace app-space` and add a new Path entry for the new service.
    - **Answer (verbatim, full manifest form):**
      ```yaml
      apiVersion: v1
      items:
      - apiVersion: extensions/v1beta1
        kind: Ingress
        metadata:
          annotations:
            nginx.ingress.kubernetes.io/rewrite-target: /
            nginx.ingress.kubernetes.io/ssl-redirect: "false"
          name: ingress-wear-watch
          namespace: app-space
        spec:
          rules:
          - http:
              paths:
              - backend:
                  serviceName: wear-service
                  servicePort: 8080
                path: /wear
                pathType: ImplementationSpecific
              - backend:
                  serviceName: video-service
                  servicePort: 8080
                path: /stream
                pathType: ImplementationSpecific
              - backend:
                  serviceName: food-service
                  servicePort: 8080
                path: /eat
                pathType: ImplementationSpecific
        status:
          loadBalancer:
            ingress:
            - {}
      kind: List
      metadata:
        resourceVersion: ""
        selfLink: ""
      ```
    - How to read it: this time an existing rule's `paths` list simply gets a **third** entry appended (`food-service` at `/eat`) — the previous two paths (`/wear`, `/stream`) are left untouched, demonstrating that adding a new route is purely additive within the same `http.paths` list, no need to touch the other entries.

19. *(Inferred question: validation checkpoint confirming `/eat` now works.)*
    - **Answer (verbatim):** `OK`

20. *(Inferred question: "A new `pay` application needs its own Ingress in a separate namespace — what namespace should it go in?")*
    - **Answer (verbatim):** `CRITICAL-SPACE`
    - How to read it: matches the `critical-space` namespace used in file 23's `test-ingress` example — this practice test appears to build directly on that lecture's manifest.

21. *(Inferred question: "What is the name of the backend Service for the pay application?")*
    - **Answer (verbatim):** `WEBAPP-PAY`
    - How to read it: note this differs from file 23's illustrative example, which used a generic `pay-service` name — in the actual lab environment the real Service backing the `pay` app is called `webapp-pay`; students must check the actual cluster (`kubectl get svc -n critical-space`) rather than assume the lecture's example name.

22. *(Task: "Create an Ingress resource named `test-ingress` in `critical-space` routing `/pay` to the `webapp-pay`/pay Service on its port, using the current stable API.")*
    - **Answer (verbatim, full manifest — current `networking.k8s.io/v1` schema):**
      ```yaml
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: test-ingress
        namespace: critical-space
        annotations:
          nginx.ingress.kubernetes.io/rewrite-target: /
          nginx.ingress.kubernetes.io/ssl-redirect: "false"
      spec:
        rules:
        - http:
            paths:
            - path: /pay
              pathType: Prefix
              backend:
                service:
                  name: pay-service
                  port:
                    number: 8282
      ```
    - How to read it/**exam-relevant contrast**: this is the **current** (`networking.k8s.io/v1`) backend schema — `backend.service.name` + `backend.service.port.number` — versus the **deprecated** `extensions/v1beta1` schema (`backend.serviceName` + `backend.servicePort`) used in file 22's original lecture and in steps 14/18 above. `pathType: Prefix` is **mandatory** on `networking.k8s.io/v1` (no default) — the exam expects one of `Exact`, `Prefix`, or `ImplementationSpecific`.

23. *(Inferred question: final validation, e.g. "Confirm `/pay` now correctly reaches the pay-service application.")*
    - **Answer (verbatim):** `OK`

- **Exam-relevant takeaway (file 24):** always verify the *actual* namespace and Service names in the live cluster (`kubectl get ns`, `kubectl get svc -A`) rather than assuming names from a lecture — the lab's real objects (`ingress-space`, `app-space`, `critical-space`, `video-service`, `webapp-pay`) don't always match a generic teaching example one-for-one. Also know **both** Ingress API schemas (`extensions/v1beta1` legacy vs. `networking.k8s.io/v1` current with mandatory `pathType` and nested `backend.service.name`/`backend.service.port.number`), since exam clusters run the current API even though older course material shows the old one. Editing an Ingress in place (`kubectl edit ingress -n <ns>`) is the standard way to add/change paths without recreating the object.

---

*Continued in [kubernetes-networking-notes_part_05.md](kubernetes-networking-notes_part_05.md)
— Sections 25–26 (Practice Test — CKA Ingress Networking 2, Download Presentation Deck)
plus the Quick Revision Checklist for the entire 09-Networking section.*
