# Create a SELinux Profile for Nginx
There are two way to create SELinux Profile, one is to use the SelinuxProfile CRD, the other is to use the RawSelinuxProfile CRD.

Where the SelinuxProfile CRD helps with the readability of the policy, user doesn't need to know CIL syntax.
The RawSelinuxProfile CRD is for advanced user who wants to write the policy in CIL syntax or some who already have the policy in CIL syntax.

## Preare the Namespace
```bash
[vincent@node spo-demo]$ oc new-project nginx-deploy
```

## Create a SelinuxProfile
```bash
[vincent@node spo-demo]$ oc create -f 01-nginx-secure-selinux-profile.yaml
```

### Make sure the profile is created and installed
```bash
[vincent@node spo-demo]$ oc get selinuxprofile -n nginx-deploy
NAME           USAGE                               STATE
nginx-secure   nginx-secure_nginx-deploy.process   Installed
```

### Find generated CIL policy file on the daemonset pod

```bash
[vincent@node spo-demo]$ oc get pods -n openshift-security-profiles
NAME                                                  READY   STATUS    RESTARTS   AGE
security-profiles-operator-75dfd79b9c-flq2n           1/1     Running   0          33m
security-profiles-operator-75dfd79b9c-ljx78           1/1     Running   0          33m
security-profiles-operator-75dfd79b9c-rrfrf           1/1     Running   0          33m
security-profiles-operator-webhook-6ff97d4598-hdn7f   1/1     Running   0          33m
security-profiles-operator-webhook-6ff97d4598-htsfw   1/1     Running   0          33m
security-profiles-operator-webhook-6ff97d4598-tz95m   1/1     Running   0          33m
spod-7f9hn                                            3/3     Running   0          33m
spod-bvtrs                                            3/3     Running   0          33m
spod-crnqq                                            3/3     Running   0          33m
spod-hbkw7                                            3/3     Running   0          33m
spod-kswm9                                            3/3     Running   0          33m
spod-mq6q9                                            3/3     Running   0          33m

[vincent@node spo-demo]$ oc exec spod-7f9hn -c selinuxd -n openshift-security-profiles -- cat /etc/selinux.d/nginx-secure_nginx-deploy.cil
(block nginx-secure_nginx-deploy
(blockinherit container)
(allow process node_t ( tcp_socket ( node_bind )))
(allow process nginx-secure_nginx-deploy.process ( tcp_socket ( listen )))
(allow process http_cache_port_t ( tcp_socket ( name_bind )))
)
```

## Create a RawSelinuxProfile 
```bash
[vincent@node spo-demo]$ oc apply -f 02-nginx-secure-selinux-raw.yaml 
rawselinuxprofile.security-profiles-operator.x-k8s.io/nginx-secure-raw created
```

### Make sure the profile is created and installed
```bash
[vincent@node spo-demo]$ oc get securityprofilenodestatuses -n nginx-deploy
NAME                                                          STATUS      AGE
nginx-secure-ip-10-0-128-32.us-west-1.compute.internal        Installed   40m
nginx-secure-ip-10-0-152-182.us-west-1.compute.internal       Installed   40m
nginx-secure-ip-10-0-159-166.us-west-1.compute.internal       Installed   40m
nginx-secure-ip-10-0-182-181.us-west-1.compute.internal       Installed   40m
nginx-secure-ip-10-0-213-154.us-west-1.compute.internal       Installed   40m
nginx-secure-ip-10-0-248-144.us-west-1.compute.internal       Installed   40m
nginx-secure-raw-ip-10-0-128-32.us-west-1.compute.internal    Installed   97s
nginx-secure-raw-ip-10-0-152-182.us-west-1.compute.internal   Installed   100s
nginx-secure-raw-ip-10-0-159-166.us-west-1.compute.internal   Installed   101s
nginx-secure-raw-ip-10-0-182-181.us-west-1.compute.internal   Installed   99s
nginx-secure-raw-ip-10-0-213-154.us-west-1.compute.internal   Installed   101s
nginx-secure-raw-ip-10-0-248-144.us-west-1.compute.internal   Installed   97s
```

# Use SELinux Profile in a Pod

For SELinux profiles, the namespace must be labelled to allow privileged workloads.

```bash
[vincent@node spo-demo]$ oc label ns nginx-deploy security.openshift.io/scc.podSecurityLabelSync=false
namespace/nginx-deploy labeled

[vincent@node spo-demo]$ oc label ns nginx-deploy --overwrite=true pod-security.kubernetes.io/enforce=privileged
namespace/nginx-deploy labeled
```

## Create a pod with the profile manually

### Get the profile usage name from the SelinuxProfile CRD
```bash
[vincent@node security-profiles-operator-upstream]$ oc get selinuxprofile.security-profiles-operator.x-k8s.io/nginx-secure -n nginx-deploy -ojsonpath='{.status.usage}'
nginx-secure_nginx-deploy.process
```

### Create a pod with the profile

```bash
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secure
  namespace: nginx-deploy
spec:
  containers:
    - image: nginxinc/nginx-unprivileged:1.21
      name: nginx
      securityContext:
        seLinuxOptions:
          # NOTE: This uses an appropriate SELinux type
          type: nginx-secure_nginx-deploy.process
EOF
```

### Check the pod status and logs
```bash
[vincent@node spo-demo]$ oc get pod -n nginx-deploy
NAME           READY   STATUS    RESTARTS   AGE
nginx-secure   1/1     Running   0          26s

[vincent@node spo-demo]$ oc logs nginx-secure -n nginx-deploy
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: /etc/nginx/conf.d/default.conf differs from the packaged version
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/09/21 11:39:50 [notice] 1#1: using the "epoll" event method
2023/09/21 11:39:50 [notice] 1#1: nginx/1.21.6
2023/09/21 11:39:50 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2023/09/21 11:39:50 [notice] 1#1: OS: Linux 5.14.0-284.16.1.el9_2.x86_64
2023/09/21 11:39:50 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2023/09/21 11:39:50 [notice] 1#1: start worker processes
2023/09/21 11:39:50 [notice] 1#1: start worker process 30
2023/09/21 11:39:50 [notice] 1#1: start worker process 31
2023/09/21 11:39:50 [notice] 1#1: start worker process 32
2023/09/21 11:39:50 [notice] 1#1: start worker process 33
```

### Check the pod if it has SELinux profile
```bash
[vincent@node spo-demo]$ oc get pod nginx-secure -n nginx-deploy -o json | jq '.spec.containers[0].securityContext.seLinuxOptions'
{
  "type": "nginx-secure_nginx-deploy.process"
}
```

### Remove the pod
```bash
[vincent@node spo-demo]$ oc delete pod nginx-secure -n nginx-deploy
```

## Use ProfileBinding to bind the profile automatically

### Create a ProfileBinding
```bash
[vincent@node spo-demo]$ oc create -f 03-nginx-profile-binding.yaml
profilebinding.security-profiles-operator.x-k8s.io/nginx-binding created
```

### Enable profile binding on the namespace
```bash
[vincent@node spo-demo]$ oc label ns nginx-deploy spo.x-k8s.io/enable-binding=true
namespace/nginx-deploy labeled
```

### Create a pod without specifying the profile in SecurityContext
```bash
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secure
  namespace: nginx-deploy
spec:
  containers:
    - image: nginxinc/nginx-unprivileged:1.21
      name: nginx
EOF
```

### Check the pod if it has SELinux profile
```bash
[vincent@node spo-demo]$ oc get pod nginx-secure -n nginx-deploy -o json | jq '.spec.containers[0].securityContext.seLinuxOptions'
{
  "type": "nginx-secure_nginx-deploy.process"
}
```

### Remove the pod and profile binding
```bash
[vincent@node spo-demo]$ oc delete pod nginx-secure -n nginx-deploy
pod "nginx-secure" deleted

[vincent@node spo-demo]$ oc delete -f 03-nginx-profile-binding.yaml 
profilebinding.security-profiles-operator.x-k8s.io "nginx-binding" deleted
```

# Create a SELinux Profile using ProfileRecording

We support create SELinux profile from scratchby recording and inspecting a selected workload as it is running.
 

## Enable profile recording on the namespace
```bash
[vincent@node spo-demo]$ oc label ns nginx-deploy spo.x-k8s.io/enable-recording=true
namespace/nginx-deploy labeled
```

## Enable log enricher to read audit log
```bash
[vincent@node spo-demo]$ oc label ns nginx-deploy spo.x-k8s.io/enable-log-enricher=true
namespace/nginx-deploy labeled
```

## Wait for the spod to be ready
```bash
[vincent@node spo-demo]$ watch oc get pod -n openshift-security-profiles
```

## Create a pod to record the profile
```bash
oc apply -f 04-nginx-profile-recording.yaml
```

### Create a pod to record the profile
```bash
oc create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: nginx-deploy
  labels:
    app: my-app
spec:
  containers:
    - name: nginx
      image: quay.io/security-profiles-operator/test-nginx:1.19.1
    - name: redis
      image: quay.io/security-profiles-operator/redis:6.2.1
EOF
```

### Check the pod status and logs
```bash
[vincent@node spo-demo]$ oc get pod -n nginx-deploy
NAME     READY   STATUS    RESTARTS   AGE
my-pod   2/2     Running   0          61s

[vincent@node spo-demo]$  oc logs my-pod -n nginx-deploy -c nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
```

### Check logs from the log enricher

```
[vincent@node spo-demo]$ oc -n openshift-security-profiles logs -f ds/spod log-enricher
Found 6 pods, using pod/spod-7sgmm
...
I0921 12:27:17.796283  131982 enricher.go:124] log-enricher "msg"="Connecting to local GRPC server" 
I0921 12:27:20.174639  131982 enricher.go:244] log-enricher "msg"="Starting GRPC server API" 
I0921 12:27:20.175002  131982 enricher.go:175] log-enricher "msg"="Reading from file /var/log/audit/audit.log" 
```

### Remove the pod
```bash
[vincent@node spo-demo]$ oc delete pod my-pod -n nginx-deploy
```

### Check the generated profile

```
[vincent@node spo-demo]$ oc get selinuxprofile -n nginx-deploy
NAME                   USAGE                                       STATE
test-recording-nginx   test-recording-nginx_nginx-deploy.process   Installed
test-recording-redis   test-recording-redis_nginx-deploy.process   Installed
```

# Remove the namespace
```bash
[vincent@node spo-demo]$ oc delete ns nginx-deploy
```