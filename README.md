# istio-cni-init-container-demo-01
demo of startup order of a mesh-enabled pod, showing how to enable external access (via UID 1337) for the init container(s)

you need to have a mesh-enabled GKE cluster already set up. in this case, i'm using Cloud Service Mesh (CSM), but this should work with OSS Istio as well.

the TLDR for this is that, when using Istio CNI, init containers are unable to call outside services because the init container execute *before* the sidecar proxy is running, but *after* traffic has been redirected to said sidecar proxy. 

see this for more detail: https://istio.io/latest/docs/setup/additional-setup/cni/#compatibility-with-application-init-containers

### create demo pod namespace

```
kubectl create ns demo
# enable sidecar injection
kubectl label namespace demo istio-injection=enabled
```

### attempt to use an init container without the ability to call outside of the mesh

```
kubectl apply -k workload-no-init-access/variant -n demo
```

see how things aren't working (and of course blocking the pod from coming up):

```
# of course, change the pod details to match your environment
$ kubectl -n demo logs whereami-5cff88f687-gzspd -c init-callgoogle
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (7) Failed to connect to www.google.com port 443 after 41 ms: Could not connect to server
trying to call google
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (7) Failed to connect to www.google.com port 443 after 12 ms: Could not connect to server
trying to call google
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (7) Failed to connect to www.google.com port 443 after 5 ms: Could not connect to server
trying to call google
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
```

### fix by setting the UID for the init container to match the UID for the sidecar proxy

setting the UID to `1337` isn't the only option - you could also do things like disable sidecar injection for the pod (not good because now the pod isn't participating in the mesh) or create port and/or IP (or IP range) exclusions for sidecar, which also might not be great because it could be really difficult to identify the target IP *plus* the exclusions apply to the entire pod.

note the updated init container manifest:

```
initContainers:
- name: init-callgoogle
image: curlimages/curl
securityContext:
    runAsUser: 1337
    allowPrivilegeEscalation: false
resources:
    requests:
    memory: "256Mi"
    cpu: "250m"
    limits:
    memory: "256Mi"
    cpu: "250m"
command: ['sh', '-c', "until curl https://www.google.com; do echo trying to call google; sleep 1; done"]
```

```
kubectl apply -k workload-set-init-container-uid-to-1337/variant -n demo
```

and now, things should work fine:

```
$ kubectl -n demo logs whereami-7d6b88c6cd-ljsgc -c init-callgoogle
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 18823    0 18823    0     0  96482      0 --:--:-- --:--:-- --:--:-- 97025
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="en"><head><meta content="Search the world's information, including webpages, images, videos and more. Google has many special features to help you find exactly what you're looking for." name="description"><meta content="noodp, " name="robots"><meta content="text/html; charset=UTF-8" http-equiv="Content-Type"><meta content="/images/branding/googleg/1x/googleg_standard_color_128dp.png" itemprop="image"><title>Google</title><script nonce="USqqXLmks1gUWH8UEdNYzg">(function(){var _g={kEI:'NWSDZ7Ey7_-nzg_jm8v5Ag',kEXPI:'0,793344,2906969,636,435,538661,2872,2891,8349,34679,30022,360901,45786,9779,38678,60726,3801,2412,58603,18674,20674,1635,29276,21780,5303,5203198,10474,768,5992087,2842796,8,27977910,25224045,4636,14986,1450,106668,884,14280,8182,5933,27322,16174,19011,2657,3437,3319,23878,9140,4599,328,6225,23407,6,10210,688,18357,977,6976,3543,1347,13703,8210,3286,4134,3755,10340,429,3288,3494,33,4669,4372,17667,10666,16224,2281,3,2,5821,4917,437,41,11881,1281,477,1,1518,3396,1827,726,3,2165,1212,352,1887,6105,39,2669,1640,2247,3665,628,908,4962,950,622,1525,4610,7,4726,1047,1413,377,842,1,1544,133,1462,603,307,891,372,2,738,5114,1110,303,2,1438,767,461,2028,4643,845,1158,1058,415,127,2021,3789,1,5684,331,2472,1684,236,1861,3,3,1158,459,1270,369,204,249,2,2,4,429,8,7,1,896,419.............
```

and the pod came up fine:

```
$ kubectl -n demo get pods --watch
NAME                        READY   STATUS    RESTARTS   AGE
whereami-7d6b88c6cd-ljsgc   2/2     Running   0          3m35s
```