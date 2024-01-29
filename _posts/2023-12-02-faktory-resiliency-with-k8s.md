---
title: Resilient Faktory K8S Deployment
categories:
  - Infrastructure
tags:
  - K8S
  - Tooling
  - Faktory
---

My current project heavily relies on background Faktory jobs, which are mission-critical. We have multiple Faktory workers (clients) in our K8S cluster that pull the queued jobs, providing ample resiliency. Given the crucial nature of successfully enqueueing the jobs for our application, we've identified a potential issue with having a single Faktory server, and we were hoping to address this concern.

To further illustrate the problem, consider an API endpoint responsible for handling user payments. This endpoint initiates a chain of Faktory jobs to ensure smooth money movement and ledgering, all processed asynchronously. If the Faktory server goes down at any point, the API endpoint will fail because the job cannot be enqueued, leading to service denial for our users.

Our starting point of a single server Faktory deployment on K8S, following [the Faktory docs](https://github.com/contribsys/faktory/wiki/Kubernetes-Deployment-Example), looked like this:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: faktory-server
spec:
  selector:
    app: faktory-server
  ports:
    - name: faktory
      protocol: TCP
      port: 7419
      targetPort: 7419
    - name: dashboard
      protocol: TCP
      port: 7420
      targetPort: 7420
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: faktory-server
  labels:
    app: faktory-server
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: faktory-server
  template:
    metadata:
      labels:
        app: faktory-server
    spec:
      shareProcessNamespace: true
      terminationGracePeriodSeconds: 10
      containers:
        - name: faktory-server
          image: xxx.dkr.ecr.us-west-2.amazonaws.com/faktory-server
          imagePullPolicy: Always
          ports:
            - containerPort: 7419
              name: faktory
            - containerPort: 7420
              name: dashboard
          volumeMounts:
            - name: faktory-server-storage-volume
              mountPath: "/var/lib/faktory"
          env:
            - name: FAKTORY_ENV
              value: production
            - name: FAKTORY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_PASSWORD
            - name: FAKTORY_LICENSE
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_LICENSE
      volumes:
        - name: faktory-server-storage-volume
          persistentVolumeClaim:
            claimName: faktory-server-storage-pv-claim
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: faktory-server-storage-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: "gp2"
  resources:
    requests:
      storage: 5Gi
```

## Migrating to Redis

As you can see, the Faktory server utilized on-disk storage, which is not suitable for a multiserver setup. Therefore, the initial priority was transitioning to using a Redis cluster.

I won't delve into the details of provisioning a Redis cluster, but there is a comprehensive [wiki page](https://github.com/contribsys/faktory/wiki/Ent-Remote-Redis) on this topic in the Faktory repository. In our case, we employed Elasticache within the AWS ecosystem.

Following the provisioning of our Redis cluster and incorporating the Redis URL into our environment variables, our deployment's manifest is now (with the removal of the PersistentVolumeClaim):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: faktory-server
  labels:
    app: faktory-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: faktory-server
  template:
    metadata:
      labels:
        app: faktory-server
    spec:
      shareProcessNamespace: true
      terminationGracePeriodSeconds: 10
      containers:
        - name: faktory-server
          image: xxx.dkr.ecr.us-west-2.amazonaws.com/faktory-server
          imagePullPolicy: Always
          ports:
            - containerPort: 7419
              name: faktory
            - containerPort: 7420
              name: dashboard
          env:
            - name: FAKTORY_ENV
              value: production
            - name: FAKTORY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_PASSWORD
            - name: FAKTORY_LICENSE
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_LICENSE
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: faktory-server-env-vars
                  key: REDIS_URL
```

As you can see above, the image for the deployment is our own image in ECR, based on the Faktory image. That image contains our configs for scheduled jobs etc. in a separate repository. More on that to come.

## Adding a Network Load Balancer

Now that we have storage that can be shared across pods, we need to add a network load balancer to route requests from the workers to the different servers. Luckily, K8S makes it quite easy. We just annotate our service with the internal load balancer annotation and add the `LoadBalancer` type:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: faktory-server
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: true
spec:
  type: LoadBalancer
  selector:
    app: faktory-server
  ports:
    - name: faktory
      protocol: TCP
      port: 7419
      targetPort: 7419
    - name: dashboard
      protocol: TCP
      port: 7420
      targetPort: 7420
```

## Duplicate Scheduled Jobs

After testing this setup for a little while, we encountered an issue with our scheduled jobs. We had a lot of cron scheduled jobs defined in a separate repository and built into the image used in our deployment.
Since we now had multiple servers, they each contained all the configs, which resulted in duplication of the scheduled jobs. That created some serious race condition issues since those jobs ran at exactly the same time.

To mitigate the issue, I moved all those configurations to the K8S manifest and separated between a "primary" Faktory server and "secondaries". The idea is that only the primary will schedule those jobs, but other than that they are all equal in responsibilities.
In case the primary fails, those jobs would not be scheduled until we get alerted and fix the issue, which is not a big issue (since those scheduled jobs are not super time sensitive).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: faktory-server-primary
  labels:
    app: faktory-server
spec:
  # The primary can only have 1 replica, so cron is not duplicated
  replicas: 1
  selector:
    matchLabels:
      app: faktory-server
  template:
    metadata:
      labels:
        app: faktory-server
    spec:
      shareProcessNamespace: true
      terminationGracePeriodSeconds: 10
      containers:
        - name: faktory-server-primary
          image: docker.contribsys.com/contribsys/faktory-ent:1.6.1
          imagePullPolicy: Always
          ports:
            - containerPort: 7419
              name: faktory
            - containerPort: 7420
              name: dashboard
          env:
            - name: FAKTORY_ENV
              value: production
            - name: FAKTORY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_PASSWORD
            - name: FAKTORY_LICENSE
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_LICENSE
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: faktory-server-env-vars
                  key: REDIS_URL
          readinessProbe:
            tcpSocket:
              port: 7420
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 7420
            initialDelaySeconds: 15
            periodSeconds: 10
          volumeMounts:
            - name: faktory-server-cron-volume
              mountPath: "/etc/faktory/conf.d"
            - name: faktory-server-statsd-volume
              mountPath: "/etc/faktory/conf.d"
      imagePullSecrets:
        - name: faktory-server-ent-login
      volumeMounts:
        - name: faktory-server-cron-volume
          mountPath: "/etc/faktory/conf.d"
        - name: faktory-server-statsd-volume
          mountPath: "/etc/faktory/conf.d"
      volumes:
        - name: faktory-server-cron-volume
          configMap:
            name: faktory-server-cron
        - name: faktory-server-statsd-volume
          configMap:
            name: faktory-server-statsd

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: faktory-server-secondary
  labels:
    app: faktory-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: faktory-server
  template:
    metadata:
      labels:
        app: faktory-server
    spec:
      shareProcessNamespace: true
      terminationGracePeriodSeconds: 10
      containers:
        - name: faktory-server-secondary
          image: docker.contribsys.com/contribsys/faktory-ent:1.6.1
          imagePullPolicy: Always
          ports:
            - containerPort: 7419
              name: faktory
            - containerPort: 7420
              name: dashboard
          env:
            - name: FAKTORY_ENV
              value: production
            - name: FAKTORY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_PASSWORD
            - name: FAKTORY_LICENSE
              valueFrom:
                secretKeyRef:
                  name: faktory-secrets
                  key: FAKTORY_LICENSE
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: faktory-server-env-vars
                  key: REDIS_URL
          readinessProbe:
            tcpSocket:
              port: 7420
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 7420
            initialDelaySeconds: 15
            periodSeconds: 10
      imagePullSecrets:
        - name: faktory-server-ent-login
```

As you can see, the only difference between the primary and the secondary deployments is the mounted volumes, which are the configs needed for the primary Faktory server. We also needed to add the enterprise login secret, which is an encoded string.

If you look closely you can also see that we have a config for statsd exporter. We export the logs to be consumed by Prometheus. I might write a separate post about that setup.

Our cron config file looks something like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: faktory-server-cron
data:
  cron.toml: |2

    # This file defines the jobs that should be scheduled via the Faktory Server
    
    # All CRON schedule times are in UTC to produce the times in CDT (UTC-5, local Viva time) indicated in the comment
    
    # Faktory uses an exponential backoff algorithm for retries:  15 + count ^ 4 + (rand(30) * (count + 1))
    # The default is 25 - which is approx 3 weeks.  20 tries is about two days.
    
    [[cron]]
    schedule = "0 12 27 * *"
    [cron.job]
    type = "MyWorker"
    args = ["SomeArgument"]
    retry = 5
```

## Issues with Job Acknowledgement

The changes above fixed the issue of scheduling duplication, and I thought we were grooving at that point. However, after some time we started observing some weird behaviour when jobs were finishing successfully, only to be enqueued again roughly half an hour after they ran initially. This was really bizarre and baffled me for a while.

After consulting with Faktory's author, turns out that half an hour is the timeout between when a job is picked up and when it is freed up again in case the worker doesn't acknowledge the job completed.
Turns out, sometimes our workers would grab a job from server A but acknowledge completion with server B, so server A would free up the job to be enqueued again after 30 minutes. This is happening since the status of the jobs is held in memory, and not represented in the persistence layer (Redis in our case), so there's no shared knowledge of that across our Faktory servers.

This is, unfortunately, where we hit a dead end. After some [discussion with Faktory's author](https://github.com/contribsys/faktory/issues/447), it became clear that changing that implementation detail is not something he's interested in doing.
We decided to keep our current deployment structure with only one master replica, and add a table to our database to dump jobs into in case the server fails, so we don't lose them while we handle fixing the issue.

In conclusion, this was a nice learning process, and I'm happy I went down this rabbit hole. Although I don't agree with the author's arguments against redundancy of the server, I understand his lack of desire to change the system drastically just for that. Our plan B solution should work great, hoping it will never actually be put to the test!
