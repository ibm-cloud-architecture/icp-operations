# Monitoring applications from the console 101

To setup kubectl with an authentication token, you will first need to get an authentication token from the console


1. Get a list of the pods in the namespace `kubectl get pods`
    ```
    NAME                          READY     STATUS    RESTARTS   AGE
    memoryleak-8544b764cb-sv5k7   1/1       Running   79         11h
    ```
1. Get a little more details about the pod `kubectl get pods -o wide` where `-o` adjust the output and `wide` asks for wide output
    ```
    NAME                          READY     STATUS    RESTARTS   AGE       IP             NODE
    memoryleak-8544b764cb-sv5k7   1/1       Running   80         11h       192.168.50.4   169.61.143.185
    ```
1. We notice that the pod has quite a lot of restarts, but we don't know why. We can see the host it runs on, but let's begin by investigating the pod. `kubectl describe pod memoryleak-8544b764cb-sv5k7`
    ```
    Name:           memoryleak-8544b764cb-sv5k7
    Namespace:      default
    Node:           169.61.143.185/169.61.143.185
    Start Time:     Tue, 26 Jun 2018 22:31:11 +0200
    Labels:         k8s-app=memoryleak
                    pod-template-hash=4100632076
    Annotations:    kubernetes.io/psp=default
    Status:         Running
    IP:             192.168.50.4
    Controlled By:  ReplicaSet/memoryleak-8544b764cb
    Containers:
      memoryleak:
        Container ID:   docker://f7d4dbb14e4c231569dd0a792d42ab384a0535232acdc86707cf293d55a1e602
        Image:          patrocinio/memoryleak:latest
        Image ID:       docker-pullable://patrocinio/memoryleak@sha256:fbe336a76c55f99ceac54c1fcafaef2740f7ce08456e7a37b0e8c67aa9a48547
        Port:           <none>
        Host Port:      <none>
        State:          Running
          Started:      Wed, 27 Jun 2018 09:41:44 +0200
        Last State:     Terminated
          Reason:       Error
          Exit Code:    1
          Started:      Wed, 27 Jun 2018 09:32:45 +0200
          Finished:     Wed, 27 Jun 2018 09:40:10 +0200
        Ready:          True
        Restart Count:  80
        Limits:
          cpu:     1
          memory:  1Gi
        Requests:
          cpu:        1
          memory:     1Gi
        Environment:  <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-qfdmg (ro)
    Conditions:
      Type           Status
      Initialized    True
      Ready          True
      PodScheduled   True
    Volumes:
      default-token-qfdmg:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-qfdmg
        Optional:    false
    QoS Class:       Guaranteed
    Node-Selectors:  <none>
    Tolerations:     <none>
    Events:
      Type     Reason   Age                 From                     Message
      ----     ------   ----                ----                     -------
      Normal   Pulling  30m (x78 over 11h)  kubelet, 169.61.143.185  pulling image "patrocinio/memoryleak:latest"
      Warning  BackOff  6m (x482 over 11h)  kubelet, 169.61.143.185  Back-off restarting failed container
    ```
    We observe a couple of interesting bits of information.
    - We can see that the container is terminating because of an error.
    ```
    Last State:     Terminated
    Reason:       Error
    Exit Code:    1
    ```
    We don't know what it is yet
    - We can see that kubernetes is automatically throttling the restarts. Kuberentes does this to protect the system and prevent any application from compromising the stability of the overall system.
    ```
    Warning  BackOff  6m (x482 over 11h)  kubelet, 169.61.143.185  Back-off restarting failed container
    ```
1. Let's see if the application is generating any logs that can give us a clue `kubectl logs memoryleak-8544b764cb-sv5k7`.
  ```
  128. Allocating memory...
  129. Allocating memory...
  130. Allocating memory...
  ```
  Does not really give a good clue, except that it is allocating memory.

1. Let's investigate resource usage. For this we use `kubectl top`
     ```
     $ kubectl top --help
    Display Resource (CPU/Memory/Storage) usage.

    The top command allows you to see the resource consumption for nodes or pods.

    This command requires Heapster to be correctly configured and working on the server.

    Available Commands:
     node        Display Resource (CPU/Memory/Storage) usage of nodes
     pod         Display Resource (CPU/Memory/Storage) usage of pods

    Usage:
     kubectl top [flags] [options]

    Use "kubectl <command> --help" for more information about a given command.
    Use "kubectl options" for a list of global command-line options (applies to all commands).
    ```
1. We will investigate the pod `kubectl top pod memoryleak-8544b764cb-sv5k7`
     ```
     NAME                          CPU(cores)   MEMORY(bytes)
     memoryleak-8544b764cb-sv5k7   15m          168Mi
     ```
1. Wait a few minutes, and do it again to see if it's consuming more resources now.
     ```
     kubectl top pod memoryleak-8544b764cb-sv5k7
     NAME                          CPU(cores)   MEMORY(bytes)
     memoryleak-8544b764cb-sv5k7   13m          302Mi
     ```
     Ok, now it uses a lot more resources, so it looks like there's a problem there possibly. We need to talk to the application owners.
1. Looking back at `kubectl describe ` we can see that the application has a memory limit of 1GB. Maybe this is not enough, or maybe the application has a memory leak causing problems.
    ```
    Limits:
      cpu:     1
      memory:  1Gi
    Requests:
      cpu:        1
      memory:     1Gi`
    ```

### Bonus material

If you have finished the previous material, you can attempt the following exercises.

#### Connecting to the container to investigate there

You can connect to a container in a similar fashion as `docker exec` though kuberentes.
1. To connect to the container use ` kubectl exec -it memoryleak-8544b764cb-sv5k7 /bin/bash`
    This starts a shell inside the container and you can perform various tasks to investigate
    ```
    root@memoryleak-8544b764cb-sv5k7:/usr/src/myapp# ls
    Dockerfile  MemoryLeak.class  MemoryLeak.java
    root@memoryleak-8544b764cb-sv5k7:/usr/src/myapp# ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root         1  1.7  2.0 3708056 83520 ?       Ssl  08:15   0:00 java MemoryLeak
    root        26  0.0  0.0  19956  3612 pts/0    Ss   08:16   0:00 /bin/bash
    root        34  0.0  0.0  38384  3140 pts/0    R+   08:16   0:00 ps aux
    root@memoryleak-8544b764cb-sv5k7:/usr/src/myapp# export
    declare -x CA_CERTIFICATES_JAVA_VERSION="20170531+nmu1"
    declare -x HOME="/root"
    declare -x HOSTNAME="memoryleak-8544b764cb-sv5k7"
    declare -x JAVA_DEBIAN_VERSION="8u151-b12-1~deb9u1"
    declare -x JAVA_HOME="/docker-java-home"
    declare -x JAVA_VERSION="8u151"
    declare -x KUBERNETES_PORT="tcp://10.0.0.1:443"
    declare -x KUBERNETES_PORT_443_TCP="tcp://10.0.0.1:443"
    declare -x KUBERNETES_PORT_443_TCP_ADDR="10.0.0.1"
    declare -x KUBERNETES_PORT_443_TCP_PORT="443"
    declare -x KUBERNETES_PORT_443_TCP_PROTO="tcp"
    declare -x KUBERNETES_SERVICE_HOST="10.0.0.1"
    declare -x KUBERNETES_SERVICE_PORT="443"
    declare -x KUBERNETES_SERVICE_PORT_HTTPS="443"
    declare -x LANG="C.UTF-8"
    declare -x OLDPWD
    declare -x PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    declare -x PWD="/usr/src/myapp"
    declare -x SHLVL="1"
    declare -x TERM="xterm"
    root@memoryleak-8544b764cb-sv5k7:/usr/src/myapp# ping 8.8.8.8
    PING 8.8.8.8 (8.8.8.8): 56 data bytes
    64 bytes from 8.8.8.8: icmp_seq=0 ttl=57 time=1.425 ms
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=57 time=1.084 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=57 time=1.103 ms
    ^C--- 8.8.8.8 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max/stddev = 1.084/1.204/1.425/0.156 ms
    root@memoryleak-8544b764cb-sv5k7:/usr/src/myapp# ping memoryleak
    ping: unknown host
    root@memoryleak-8544b764cb-sv5k7:/usr/src/myapp# ping memoryleak-8544b764cb-sv5k7
    PING memoryleak-8544b764cb-sv5k7 (192.168.50.4): 56 data bytes
    64 bytes from 192.168.50.4: icmp_seq=0 ttl=64 time=0.063 ms
    64 bytes from 192.168.50.4: icmp_seq=1 ttl=64 time=0.082 ms

    ```


#### Advanced application log investigation
For those who have completed the exercise. Another very useful CLI tool for investigating logs is **stern**. You can download it [here](https://github.com/wercker/stern)

Stern can stream logs from multiple containers at the same time, based on name or based on labels.

As an experiment let's look at all the pods involved with the authentication mechanism in ICP. As we can see these are named auth-<microservice/function>-<unique-id>

```
$ kubectl -n kube-system get pods | grep auth
auth-apikeys-tqcw4                                            1/1       Running     0          16h
auth-idp-c9mnc                                                3/3       Running     0          16h
auth-idp-platform-auth-cert-gen-jr9s2                         0/1       Completed   0          16h
auth-pap-4427m                                                1/1       Running     0          16h
auth-pdp-b4f4p                                                1/1       Running     0          16h
```

Now run `stern -n kube-system auth*` to stream logs from all containers associated with this function.

```
auth-idp-c9mnc platform-identity-provider [2018-06-26T21:44:29.600Z]  INFO: platform-identity-provider/18 on auth-idp-c9mnc: Calling Userinfo Post
auth-idp-c9mnc platform-identity-provider [2018-06-26T21:44:29.614Z]  INFO: platform-identity-provider/18 on auth-idp-c9mnc: Calling Userinfo Post
auth-idp-c9mnc platform-identity-provider [2018-06-26T21:44:29.630Z]  INFO: platform-identity-provider/18 on auth-idp-c9mnc: Calling Userinfo Post
auth-pap-4427m auth-pap                         "default": "s
```

Note the format of the output.
1. pod name
2. container name
3. log entry


#### Extra challenges

1. Figure out how to query logs based on labels that group multiple pods with stern
2. Investigate the operations you can do with kubectl `kubectl --help`
3. If you operating system allows it, setup autocomplete for kubectl and stern
    - [kubectl autocomplete](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)
    - [stern autocomplete](https://github.com/wercker/stern#completion)
