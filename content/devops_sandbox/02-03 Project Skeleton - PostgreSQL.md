---
title: 02-03 Project Skeleton - PostgreSQL
fileName: 02-03 Project Skeleton - PostgreSQL
tags:
- postgres
- statefulset
- k8s
categories:
- 
date: 2023-02-14
lastMod: 2023-02-16
ShowPostNavLinks: true
ShowBreadCrumbs: true
ShowReadingTime: true
---
# PostgreSQL

I've chosen [PostgreSQL](https://www.postgresql.org/) for the same reason as Kafka: I'm used to it from my day-to-day work. I will also use a Statefulset for this deployment, but I'll leave the redundancy for later. As with KRaft, I learned the hard way that redundancy is not magical, but that I need to configure it myself.

## Service

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  labels:
    app: postgres-app
spec:
  clusterIP: None
  ports:
  	- name: '5432'
      port: 5432
      protocol: TCP
      targetPort: 5432
  selector:
    app: postgres-app
```

Again, a Headless service (as with Kafka). I should be able to access it through `postgres-svc:5432`, and I'll check that later.

## Statefulset

`statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  labels:
    app: postgres-app
spec:
  serviceName: postgres-svc
  replicas: 1
  selector:
    matchLabels:
      app: postgres-app
  template:
    metadata:
      labels:
        app: postgres-app
    spec:
      containers:
        - name: postgres-container
          image: postgres:15
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - name: postgresdata
              mountPath: /var/lib/postgresql/data
              subPath: data
  volumeClaimTemplates:
    - metadata:
        name: postgresdata
      spec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: "10Gi"
```

One of the issue I had when following an online guide was that the volume was mounted on `/data/volume`. After destroying my pod, I looked for the table I just created but it was missing. The answer was the mount path, which should have been `/var/lib/postgresql/data`.

While I was at it, I decided to check the different values accepted by the volume's `accessModes`. This parameter is used to restrict who can read and write, and also how many "who"s. The possible values are:

  + `ReadWriteMany`: "`rw` for everyone", Unix-style. This is used when multiple pods need to write to the same volume.

  + `ReadWriteOnce` is `rw` for one pod, which is what I need since the volume will always be tied to the same pod.

  + `ReadOnlyMany` and `ReadOnlyOnce` are the read-only variants. I wonder if it is possible to use them to create read-only PostgreSQL instances with a single write instance...

## Secret

Finally the secret, already seen in `statefulset.yaml` as a `secretRef`. It could have been `postgres:postgres`, but I'll try to do better. Also, I will use `kustomize` for this, as it is integrated with kubernetes.

`kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - service.yaml
  - statefulset.yaml

secretGenerator:
  - name: postgres-secret
    literals:
      - POSTGRES_DB=sand_db
      - POSTGRES_USER=sandman
      - POSTGRES_PASSWORD=<super-duper password>
generatorOptions:
  disableNameSuffixHash: true
```

You can see the `secretGenerator` section, with all the different fields I'm using in the PostgreSQL pod. The password was generated using:

```bash
$ openssl rand -base64 32
```

The `disableNameSuffixHash: true` is here because by default, kustomize will generate a secret with a random suffix. Since I want to directly reference that secret in my `statefulset.yaml`, I disabled it.

## Applying

Once all the files are created, deploying PostgreSQL is simple:

```bash
$ k apply -k .
$ k get pods
NAME           READY   STATUS    RESTARTS         AGE
postgres-0     1/1     Running   3 (3h20m ago)    3d
```

(The number of restarts corresponds to the number of times I've rebooted my computer)

## Checks

### Check the secret

Is the secret properly created?

```bash
$ k get secret postgres-secret -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d
GetYourOwnPassword=
```

Of course the password is not `GetYourOwnPassword=`...

### Check the network

```bash
$ k run dig --rm -ti --image arunvelsriram/utils -- bash
utils@dig:~$ nslookup postgres-svc
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	postgres-svc.default.svc.cluster.local
Address: 10.244.0.4

utils@dig:~$ dig postgres-0.postgres-svc.default.svc.cluster.local
[...]
;; ANSWER SECTION:
postgres-0.postgres-svc.default.svc.cluster.local. 30 IN A 10.244.0.4
```

As with Kafka, I expect to have more IPs if I increase the replicas in the `statefulset.yaml`.

### Check PostgreSQL

Is the thing I deployed really PostgreSQL?

```bash
$ export POSTGRES_PASSWORD=$(k get secret postgres-secret -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d)

$ k run postgresql-dev-client --rm --tty -i --restart='Never' --image docker.io/bitnami/postgresql:15 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
--command -- psql --host postgres-svc -U sandman -d sand_db -p 5432

sand_db=# \x
Expanded display is on.

sand_db=# CREATE TABLE dream (id uuid DEFAULT gen_random_uuid() PRIMARY KEY, name text);
CREATE TABLE

sand_db=# \dt
List of relations
-[ RECORD 1 ]---
Schema | public
Name   | dream
Type   | table
Owner  | sandman

sand_db=# INSERT INTO dream (name) VALUES ('Mr Sandman');
INSERT 0 1

sand_db=# SELECT * FROM dream;
-[ RECORD 1 ]------------------------------
id   | 9fe0ce9a-075e-48ec-92eb-59c42a794d6d
name | Mr Sandman
```

### Check persistence

Now is my data really persisted? This test was quite useful, as it helped my identify the wrong mount path of my volume.

```bash
$ k delete pod postgres-0
pod "postgres-0" deleted

# check that the pod is automatically recreated
$ k get pods
NAME                    READY   STATUS    RESTARTS         AGE
postgres-0              1/1     Running   0                17s

$ export POSTGRES_PASSWORD=$(k get secret postgres-secret -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d)

$ k run postgresql-dev-client --rm --tty -i --restart='Never' --image docker.io/bitnami/postgresql:15 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
--command -- psql --host postgres-svc -U sandman -d sand_db -p 5432

sand_db=# \dt
List of relations
-[ RECORD 1 ]---
Schema | public
Name   | dream
Type   | table
Owner  | sandman

sand_db=# SELECT * FROM dream;
-[ RECORD 1 ]------------------------------
id   | 9fe0ce9a-075e-48ec-92eb-59c42a794d6d
name | Mr Sandman
```

Mr Sandman is still here!

## Takeaways

Statefulsets are neat, and I want to see how to implement redundancy. But that would be for later...

Checking early was useful: it helped me weed out the volume configuration bug. Imagine having applications connected to the DB and having no data... That would have been harder to debug!
