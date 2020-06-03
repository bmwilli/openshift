# Hello-node

The kubernetes example for minikube does not run as is with openshift.

This is because the container that it uses needs to execute as the root user.

In these steps we create a service account in openshift and assign it the SCC (Security Context Constraint) of anyuid

## Prerequisites
* Access to an admin account in openshift that can assign the SCC anyuid to a service account

## Steps

1. Create a new project
```bash
oc new-project test
```

2. Create deployment
```bash
oc create deployment hello-node --image=k8s.gcr.io/echoserver:1.4

deployment.apps/hello-node created
```

3. Note that pod will be in error
```bash
oc get pods

NAME                          READY   STATUS             RESTARTS   AGE
hello-node-7dc7987866-f65v2   0/1     CrashLoopBackOff   2          41s
```

4. Examine issue
```bash
oc logs hello-node-7dc7987866-f65v2

2020/06/03 17:35:02 [emerg] 1#1: mkdir() "/var/lib/nginx/proxy" failed (13: Permission denied)
nginx: [emerg] mkdir() "/var/lib/nginx/proxy" failed (13: Permission denied)
```

5. Show list of SCC available
```bash
oc get scc

NAME               AGE
anyuid             4d6h
hostaccess         4d6h
hostmount-anyuid   4d6h
hostnetwork        4d6h
node-exporter      4d5h
nonroot            4d6h
privileged         4d6h
restricted         4d6h
```

6. Let OpenShift determine which SCC could solve problem
```bash
oc get pod hello-node-7dc7987866-f65v2 -o yaml | oc adm policy scc-subject-review -f -

RESOURCE                          ALLOWED BY
Pod/hello-node-7dc7987866-f65v2   anyuid
```

7. Create a service account
```bash
oc create serviceaccount hello-node-sa

serviceaccount/hello-node-sa created
```

8. Assign anyuid SCC to the service account
```bash
oc adm policy add-scc-to-user anyuid -z hello-node-sa

securitycontextconstraints.security.openshift.io/anyuid added to: ["system:serviceaccount:test:hello-node-sa"]
```

9. Set Service Account on Deployment
```bash
oc set serviceaccount deployment hello-node hello-node-sa

deployment.apps/hello-node serviceaccount updated
```

10. Check pod status until only one running
```bash
oc get pods

NAME                          READY   STATUS              RESTARTS   AGE
hello-node-6cc8f6c9d4-kgp7x   0/1     ContainerCreating   0          2s
hello-node-7dc7987866-f65v2   0/1     CrashLoopBackOff    6          10m

oc get pods

NAME                          READY   STATUS        RESTARTS   AGE
hello-node-6cc8f6c9d4-kgp7x   1/1     Running       0          6s
hello-node-7dc7987866-f65v2   0/1     Terminating   6          10m

oc get pods

NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6cc8f6c9d4-kgp7x   1/1     Running   0          15s
```

11. Create service
```bash
oc expose deployment hello-node --port 8080

service/hello-node exposed
```

12. Create route
```bash
oc expose service hello-node

route.route.openshift.io/hello-node exposed
```

13. Show route url
```bash
oc get route hello-node

NAME         HOST/PORT                          PATH   SERVICES     PORT   TERMINATION   WILDCARD
hello-node   hello-node-test.apps-crc.testing          hello-node   8080                 None
```

14. Test route
```bash
curl http://hello-node-test.apps-crc.testing

CLIENT VALUES:
client_address=10.128.0.1
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://hello-node-test.apps-crc.testing:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
forwarded=for=192.168.64.1;host=hello-node-test.apps-crc.testing;proto=http;proto-version=""
host=hello-node-test.apps-crc.testing
user-agent=curl/7.54.0
x-forwarded-for=192.168.64.1
x-forwarded-host=hello-node-test.apps-crc.testing
x-forwarded-port=80
x-forwarded-proto=http
BODY:
-no body in request-
```
