# Example: Deploying PHP Guestbook application with Redis

## Creating the Redis Deployment 
The manifest file, included below, specifies a Deployment controller that runs a single replica Redis Pod.

```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: leader
        tier: backend
        app.kcas/name: redis-leader
    spec:
      schedulerName: kcas-scheduler
      containers:
      - name: leader
        image: "docker.io/redis:6.0.5"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```
## Creating the Redis leader Service 
The guestbook application needs to communicate to the Redis to write its data. You need to apply a Service to proxy the traffic to the Redis Pod. A Service defines a policy to access the Pods

```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: leader
    tier: backend
```

## Set up Redis followers 
Although the Redis leader is a single Pod, you can make it highly available and meet traffic demands by adding a few Redis followers, or replicas.

```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: follower
        tier: backend
        app.kcas/name: redis-follower
    spec:
      schedulerName: kcas-scheduler
      containers:
      - name: follower
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-redis-follower:v2
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

## Creating the Redis follower service 
The guestbook application needs to communicate with the Redis followers to read data. To make the Redis followers discoverable, you must set up another Service.

```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    app: redis
    role: follower
    tier: backend
```

## Set up and Expose the Guestbook Frontend 
Now that you have the Redis storage of your guestbook up and running, start the guestbook web servers. Like the Redis followers, the frontend is deployed using a Kubernetes Deployment.

The guestbook app uses a PHP frontend. It is configured to communicate with either the Redis follower or leader Services, depending on whether the request is a read or a write. The frontend exposes a JSON interface, and serves a jQuery-Ajax-based UX.

### Creating the Guestbook Frontend Deployment 

```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
        app: guestbook
        tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
        app.kcas/name: frontend
    spec:
      schedulerName: kcas-scheduler
      containers:
      - name: php-redis
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80

```

### Creating the Frontend Service 
The Redis Services you applied is only accessible within the Kubernetes cluster because the default type for a Service is ClusterIP. ClusterIP provides a single IP address for the set of Pods the Service is pointing to. This IP address is accessible only within the cluster.

If you want guests to be able to access your guestbook, you must configure the frontend Service to be externally visible, so a client can request the Service from outside the Kubernetes cluster. However a Kubernetes user can use kubectl port-forward to access the service even though it uses a ClusterIP.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: guestbook
    tier: frontend
```

## Create the HPA for Guestbook
Now, create a new HPA spec file for the guestbook.
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: guestbook-frontend
  #namespace: guestbook
  labels:
    app: guestbook
    #env: production
    tier: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 75
```

## Load test with Siege
To force the HPA into action, we’ll use Siege, an HTTP load testing and benchmark utility. Siege is a multi-threaded load testing tool and has a few other capabilities included to make it a good option for putting some force onto a simple web app.

First, put various permutations of the URL in a plaintext file. By doing this, Siege can randomly scan the URLs in he text file and ping them in “Internet mode” by randomly selecting a URL from the list for each request. This could look like the following…
```txt
http://my-guestbook.example.com/
http://my-guestbook.example.com/index.html
http://my-guestbook.example.com/guestbook.php
http://my-guestbook.example.com/guestbook.php?cmd=get&key=messages
```

Once this is done, you can fire up Siege to begin load testing. In this case, to get fast results, we’ll use 255 concurrent users for five minutes, using Internet and benchmark modes.

```bash
siege --verbose --benchmark --internet --concurrent 255 --time 10M --file siege-urls.txt
```
