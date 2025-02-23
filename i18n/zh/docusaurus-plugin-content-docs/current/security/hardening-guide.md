---
title: "CIS Hardening Guide"
weight: 80
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

:::info
请知悉，本文仅提供英文版。
:::

This document provides prescriptive guidance for hardening a production installation of K3s. It outlines the configurations and controls required to address Kubernetes benchmark controls from the Center for Internet Security (CIS).

K3s has a number of security mitigations applied and turned on by default and will pass a number of the Kubernetes CIS controls without modification. There are some notable exceptions to this that require manual intervention to fully comply with the CIS Benchmark:

1. K3s will not modify the host operating system. Any host-level modifications will need to be done manually.
2. Certain CIS policy controls for `NetworkPolicies` and `PodSecurityStandards` (`PodSecurityPolicies` on v1.24 and older) will restrict the functionality of the cluster. You must opt into having K3s configure these by adding the appropriate options (enabling of admission plugins) to your command-line flags or configuration file as well as manually applying appropriate policies. Further details are presented in the sections below.

The first section (1.1) of the CIS Benchmark concerns itself primarily with pod manifest permissions and ownership. K3s doesn't utilize these for the core components since everything is packaged into a single binary.

## Host-level Requirements

There are two areas of host-level requirements: kernel parameters and etcd process/directory configuration. These are outlined in this section.

### Ensure `protect-kernel-defaults` is set

This is a kubelet flag that will cause the kubelet to exit if the required kernel parameters are unset or are set to values that are different from the kubelet's defaults.

> **Note:** `protect-kernel-defaults` is exposed as a top-level flag for K3s.

#### Set kernel parameters

Create a file called `/etc/sysctl.d/90-kubelet.conf` and add the snippet below. Then run `sysctl -p /etc/sysctl.d/90-kubelet.conf`.

```bash
vm.panic_on_oom=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
kernel.keys.root_maxbytes=25000000
```

## Kubernetes Runtime Requirements

The runtime requirements to comply with the CIS Benchmark are centered around pod security (via PSP or PSA), network policies and API Server auditing logs. These are outlined in this section.

By default, K3s does not include any pod security or network policies. However, K3s ships with a controller that will enforce network policies, if any are created. K3s doesn't enable auditing by default, so audit log configuration and audit policy must be created manually. By default, K3s runs with the both the `PodSecurity` and `NodeRestriction` admission controllers enabled, among others.

### Pod Security

<Tabs>
<TabItem value="v1.25 and Newer" default>

K3s v1.25 and newer support [Pod Security Admissions (PSAs)](https://kubernetes.io/docs/concepts/security/pod-security-admission/) for controlling pod security. PSAs are enabled by passing the following flag to the K3s server:
```
--kube-apiserver-arg="admission-control-config-file=/var/lib/rancher/k3s/server/psa.yaml"
```

The policy should be written to a file named `psa.yaml` in `/var/lib/rancher/k3s/server` directory. 

Here is an example of a complaint PSA:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1beta1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "restricted"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: [kube-system, cis-operator-system]
```
</TabItem>
<TabItem value="v1.24 and Older" default>

K3s v1.24 and older support [Pod Security Policies (PSPs)](https://v1-24.docs.kubernetes.io/docs/concepts/security/pod-security-policy/) for controlling pod security. PSPs are enabled by passing the following flag to the K3s server:

```
--kube-apiserver-arg="enable-admission-plugins=NodeRestriction,PodSecurityPolicy"
```
This will have the effect of maintaining the `NodeRestriction` plugin as well as enabling the `PodSecurityPolicy`. 

When PSPs are enabled, a policy can be applied to satisfy the necessary controls described in section 5.2 of the CIS Benchmark.

Here is an example of a compliant PSP:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false                # CIS - 5.2.1
  allowPrivilegeEscalation: false  # CIS - 5.2.5
  requiredDropCapabilities:        # CIS - 5.2.7/8/9
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'csi'
    - 'persistentVolumeClaim'
    - 'ephemeral'
  hostNetwork: false               # CIS - 5.2.4
  hostIPC: false                   # CIS - 5.2.3
  hostPID: false                   # CIS - 5.2.2
  runAsUser:
    rule: 'MustRunAsNonRoot'       # CIS - 5.2.6
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

For the above PSP to be effective, we need to create a ClusterRole and a ClusterRoleBinding. We also need to include a "system unrestricted policy" which is needed for system-level pods that require additional privileges, and an additional policy that allows sysctls necessary for servicelb to function properly.

Combining the configuration above with the [Network Policy](#networkpolicies) described in the next section, a single file can be placed in the `/var/lib/rancher/k3s/server/manifests` directory. Here is an example of a `policy.yaml` file: 

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'csi'
    - 'persistentVolumeClaim'
    - 'ephemeral'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: system-unrestricted-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  fsGroup:
    rule: RunAsAny
  hostIPC: true
  hostNetwork: true
  hostPID: true
  hostPorts:
  - max: 65535
    min: 0
  privileged: true
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - '*'
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: svclb-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  allowPrivilegeEscalation: false
  allowedCapabilities:
  - NET_ADMIN
  allowedUnsafeSysctls:
  - net.ipv4.ip_forward
  - net.ipv6.conf.all.forwarding
  fsGroup:
    rule: RunAsAny
  hostPorts:
  - max: 65535
    min: 0
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:restricted-psp
rules:
- apiGroups:
  - policy
  resources:
  - podsecuritypolicies
  verbs:
  - use
  resourceNames:
  - restricted-psp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:system-unrestricted-psp
rules:
- apiGroups:
  - policy
  resources:
  - podsecuritypolicies
  resourceNames:
  - system-unrestricted-psp
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:svclb-psp
rules:
- apiGroups:
  - policy
  resources:
  - podsecuritypolicies
  resourceNames:
  - svclb-psp
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default:restricted-psp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:restricted-psp
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system-unrestricted-node-psp-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:system-unrestricted-psp
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: system-unrestricted-svc-acct-psp-rolebinding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:system-unrestricted-psp
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: svclb-psp-rolebinding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:svclb-psp
subjects:
- kind: ServiceAccount
  name: svclb
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: intra-namespace
  namespace: kube-system
spec:
  podSelector: {}
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: kube-system
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: intra-namespace
  namespace: default
spec:
  podSelector: {}
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: default
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: intra-namespace
  namespace: kube-public
spec:
  podSelector: {}
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: kube-public
```

</TabItem>
</Tabs>


> **Note:** The Kubernetes critical additions such as CNI, DNS, and Ingress are run as pods in the `kube-system` namespace. Therefore, this namespace will have a policy that is less restrictive so that these components can run properly.

### NetworkPolicies

CIS requires that all namespaces have a network policy applied that reasonably limits traffic into namespaces and pods.

Network policies should be placed the `/var/lib/rancher/k3s/server/manifests` directory, where they will automatically be deployed on startup.

Here is an example of a compliant network policy.

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: intra-namespace
  namespace: kube-system
spec:
  podSelector: {}
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: kube-system
```

With the applied restrictions, DNS will be blocked unless purposely allowed. Below is a network policy that will allow for traffic to exist for DNS.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-network-dns-policy
  namespace: <NAMESPACE>
spec:
  ingress:
  - ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP
  podSelector:
    matchLabels:
      k8s-app: kube-dns
  policyTypes:
  - Ingress
```

The metrics-server and Traefik ingress controller will be blocked by default if network policies are not created to allow access. Traefik v1 as packaged in K3s version 1.20 and below uses different labels than Traefik v2. Ensure that you only use the sample yaml below that is associated with the version of Traefik present on your cluster.

<Tabs>
<TabItem value="v1.21 and Newer" default>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-metrics-server
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      k8s-app: metrics-server
  ingress:
  - {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-svclbtraefik-ingress
  namespace: kube-system
spec:
  podSelector: 
    matchLabels:
      svccontroller.k3s.cattle.io/svcname: traefik
  ingress:
  - {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-traefik-v121-ingress
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: traefik
  ingress:
  - {}
  policyTypes:
  - Ingress
---

```
</TabItem>

<TabItem value="v1.20 and Older" default>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-metrics-server
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      k8s-app: metrics-server
  ingress:
  - {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-svclbtraefik-ingress
  namespace: kube-system
spec:
  podSelector: 
    matchLabels:
      svccontroller.k3s.cattle.io/svcname: traefik
  ingress:
  - {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-traefik-v120-ingress
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      app: traefik
  ingress:
  - {}
  policyTypes:
  - Ingress
---

```
</TabItem>
</Tabs>

:::info
Operators must manage network policies as normal for additional namespaces that are created.
:::


### API Server audit configuration

CIS requirements 1.2.22 to 1.2.25 are related to configuring audit logs for the API Server. K3s doesn't create by default the log directory and audit policy, as auditing requirements are specific to each user's policies and environment.

The log directory, ideally, must be created before starting K3s. A restrictive access permission is recommended to avoid leaking potential sensitive information.

```bash
sudo mkdir -p -m 700 /var/lib/rancher/k3s/server/logs
```

A starter audit policy to log request metadata is provided below. The policy should be written to a file named `audit.yaml` in `/var/lib/rancher/k3s/server` directory. Detailed information about policy configuration for the API server can be found in the Kubernetes [documentation](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/).

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```

Both configurations must be passed as arguments to the API Server as:

```bash
--kube-apiserver-arg='audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log'
--kube-apiserver-arg='audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml'
```

If the configurations are created after K3s is installed, they must be added to K3s' systemd service in `/etc/systemd/system/k3s.service`.

```bash
ExecStart=/usr/local/bin/k3s \
    server \
	'--kube-apiserver-arg=audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log' \
	'--kube-apiserver-arg=audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml' \
```

K3s must be restarted to load the new configuration.

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s.service
```

## Configuration for Kubernetes Components


The configuration below should be placed in the [configuration file](../installation/configuration.md#configuration-file), and contains all the necessary remediations to harden the Kubernetes components.


<Tabs>
<TabItem value="v1.25 and Newer" default>

```yaml
protect-kernel-defaults: true
secrets-encryption: true
kube-apiserver-arg:
  - 'admission-control-config-file=/var/lib/rancher/k3s/server/psa.yaml'
  - 'audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log'
  - 'audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml'
  - 'audit-log-maxage=30'
  - 'audit-log-maxbackup=10'
  - 'audit-log-maxsize=100'
  - 'request-timeout=300s'
  - 'service-account-lookup=true'
kube-controller-manager-arg:
  - 'terminated-pod-gc-threshold=10'
  - 'use-service-account-credentials=true'
kubelet-arg:
  - 'streaming-connection-idle-timeout=5m'
  - 'make-iptables-util-chains=true'
```

</TabItem>

<TabItem value="v1.24 and Older" default>

```yaml
protect-kernel-defaults: true
secrets-encryption: true
kube-apiserver-arg:
  - 'enable-admission-plugins=NodeRestriction,PodSecurityPolicy,NamespaceLifecycle,ServiceAccount'
  - 'audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log'
  - 'audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml'
  - 'audit-log-maxage=30'
  - 'audit-log-maxbackup=10'
  - 'audit-log-maxsize=100'
  - 'request-timeout=300s'
  - 'service-account-lookup=true'
kube-controller-manager-arg:
  - 'terminated-pod-gc-threshold=10'
  - 'use-service-account-credentials=true'
kubelet-arg:
  - 'streaming-connection-idle-timeout=5m'
  - 'make-iptables-util-chains=true'
```

</TabItem>
</Tabs>


## Control Plane Execution and Arguments

Listed below are the K3s control plane components and the arguments they are given at start, by default. Commented to their right is the CIS 1.6 control that they satisfy.

```bash
kube-apiserver 
    --advertise-port=6443 
    --allow-privileged=true 
    --anonymous-auth=false                                                            # 1.2.1
    --api-audiences=unknown 
    --authorization-mode=Node,RBAC 
    --bind-address=127.0.0.1 
    --cert-dir=/var/lib/rancher/k3s/server/tls/temporary-certs
    --client-ca-file=/var/lib/rancher/k3s/server/tls/client-ca.crt                    # 1.2.31
    --enable-admission-plugins=NodeRestriction,PodSecurityPolicy                      # 1.2.17
    --etcd-cafile=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt                  # 1.2.32
    --etcd-certfile=/var/lib/rancher/k3s/server/tls/etcd/client.crt                   # 1.2.29
    --etcd-keyfile=/var/lib/rancher/k3s/server/tls/etcd/client.key                    # 1.2.29
    --etcd-servers=https://127.0.0.1:2379 
    --insecure-port=0                                                                 # 1.2.19
    --kubelet-certificate-authority=/var/lib/rancher/k3s/server/tls/server-ca.crt 
    --kubelet-client-certificate=/var/lib/rancher/k3s/server/tls/client-kube-apiserver.crt 
    --kubelet-client-key=/var/lib/rancher/k3s/server/tls/client-kube-apiserver.key 
    --profiling=false                                                                 # 1.2.21
    --proxy-client-cert-file=/var/lib/rancher/k3s/server/tls/client-auth-proxy.crt 
    --proxy-client-key-file=/var/lib/rancher/k3s/server/tls/client-auth-proxy.key 
    --requestheader-allowed-names=system:auth-proxy 
    --requestheader-client-ca-file=/var/lib/rancher/k3s/server/tls/request-header-ca.crt 
    --requestheader-extra-headers-prefix=X-Remote-Extra- 
    --requestheader-group-headers=X-Remote-Group 
    --requestheader-username-headers=X-Remote-User 
    --secure-port=6444                                                                # 1.2.20
    --service-account-issuer=k3s 
    --service-account-key-file=/var/lib/rancher/k3s/server/tls/service.key            # 1.2.28
    --service-account-signing-key-file=/var/lib/rancher/k3s/server/tls/service.key 
    --service-cluster-ip-range=10.43.0.0/16 
    --storage-backend=etcd3 
    --tls-cert-file=/var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt        # 1.2.30
    --tls-private-key-file=/var/lib/rancher/k3s/server/tls/serving-kube-apiserver.key # 1.2.30
    --tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
```

```bash
kube-controller-manager 
    --address=127.0.0.1 
    --allocate-node-cidrs=true 
    --bind-address=127.0.0.1                                                       # 1.3.7
    --cluster-cidr=10.42.0.0/16 
    --cluster-signing-cert-file=/var/lib/rancher/k3s/server/tls/client-ca.crt 
    --cluster-signing-key-file=/var/lib/rancher/k3s/server/tls/client-ca.key 
    --kubeconfig=/var/lib/rancher/k3s/server/cred/controller.kubeconfig 
    --port=10252 
    --profiling=false                                                              # 1.3.2
    --root-ca-file=/var/lib/rancher/k3s/server/tls/server-ca.crt                   # 1.3.5
    --secure-port=0 
    --service-account-private-key-file=/var/lib/rancher/k3s/server/tls/service.key # 1.3.4 
    --use-service-account-credentials=true                                         # 1.3.3
```

```bash
kube-scheduler 
    --address=127.0.0.1 
    --bind-address=127.0.0.1                                              # 1.4.2
    --kubeconfig=/var/lib/rancher/k3s/server/cred/scheduler.kubeconfig 
    --port=10251 
    --profiling=false                                                     # 1.4.1
    --secure-port=0
```

```bash
kubelet 
    --address=0.0.0.0 
    --anonymous-auth=false                                                # 4.2.1
    --authentication-token-webhook=true 
    --authorization-mode=Webhook                                          # 4.2.2
    --cgroup-driver=cgroupfs 
    --client-ca-file=/var/lib/rancher/k3s/agent/client-ca.crt             # 4.2.3
    --cloud-provider=external 
    --cluster-dns=10.43.0.10 
    --cluster-domain=cluster.local 
    --cni-bin-dir=/var/lib/rancher/k3s/data/223e6420f8db0d8828a8f5ed3c44489bb8eb47aa71485404f8af8c462a29bea3/bin 
    --cni-conf-dir=/var/lib/rancher/k3s/agent/etc/cni/net.d 
    --container-runtime-endpoint=/run/k3s/containerd/containerd.sock 
    --container-runtime=remote 
    --containerd=/run/k3s/containerd/containerd.sock 
    --eviction-hard=imagefs.available<5%,nodefs.available<5% 
    --eviction-minimum-reclaim=imagefs.available=10%,nodefs.available=10% 
    --fail-swap-on=false 
    --healthz-bind-address=127.0.0.1 
    --hostname-override=hostname01 
    --kubeconfig=/var/lib/rancher/k3s/agent/kubelet.kubeconfig 
    --kubelet-cgroups=/systemd/system.slice 
    --node-labels= 
    --pod-manifest-path=/var/lib/rancher/k3s/agent/pod-manifests 
    --protect-kernel-defaults=true                                        # 4.2.6
    --read-only-port=0                                                    # 4.2.4
    --resolv-conf=/run/systemd/resolve/resolv.conf 
    --runtime-cgroups=/systemd/system.slice 
    --serialize-image-pulls=false 
    --tls-cert-file=/var/lib/rancher/k3s/agent/serving-kubelet.crt        # 4.2.10
    --tls-private-key-file=/var/lib/rancher/k3s/agent/serving-kubelet.key # 4.2.10
```
Additional information about CIS requirements 1.2.22 to 1.2.25 is presented below.

## Known Issues
The following are controls that K3s currently does not pass by default. Each gap will be explained, along with a note clarifying whether it can be passed through manual operator intervention, or if it will be addressed in a future release of K3s.

### Control 1.2.15
Ensure that the admission control plugin `NamespaceLifecycle` is set.
<details>
<summary>Rationale</summary>
Setting admission control policy to `NamespaceLifecycle` ensures that objects cannot be created in non-existent namespaces, and that namespaces undergoing termination are not used for creating the new objects. This is recommended to enforce the integrity of the namespace termination process and also for the availability of the newer objects.

This can be remediated by passing this argument as a value to the `enable-admission-plugins=` and pass that to  `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.16
Ensure that the admission control plugin `PodSecurityPolicy` is set.
<details>
<summary>Rationale</summary>
A Pod Security Policy is a cluster-level resource that controls the actions that a pod can perform and what it has the ability to access. The `PodSecurityPolicy` objects define a set of conditions that a pod must run with in order to be accepted into the system. Pod Security Policies are comprised of settings and strategies that control the security features a pod has access to and hence this must be used to control pod access permissions.

This can be remediated by passing this argument as a value to the `enable-admission-plugins=` and pass that to  `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.22
Ensure that the `--audit-log-path` argument is set.
<details>
<summary>Rationale</summary>
Auditing the Kubernetes API Server provides a security-relevant chronological set of records documenting the sequence of activities that have affected system by individual users, administrators or other components of the system. Even though currently, Kubernetes provides only basic audit capabilities, it should be enabled. You can enable it by setting an appropriate audit log path.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.23
Ensure that the `--audit-log-maxage` argument is set to 30 or as appropriate.
<details>
<summary>Rationale</summary>
Retaining logs for at least 30 days ensures that you can go back in time and investigate or correlate any events. Set your audit log retention period to 30 days or as per your business requirements.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.24
Ensure that the `--audit-log-maxbackup` argument is set to 10 or as appropriate.
<details>
<summary>Rationale</summary>
Kubernetes automatically rotates the log files. Retaining old log files ensures that you would have sufficient log data available for carrying out any investigation or correlation. For example, if you have set file size of 100 MB and the number of old log files to keep as 10, you would approximate have 1 GB of log data that you could potentially use for your analysis.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.25
Ensure that the `--audit-log-maxsize` argument is set to 100 or as appropriate.
<details>
<summary>Rationale</summary>
Kubernetes automatically rotates the log files. Retaining old log files ensures that you would have sufficient log data available for carrying out any investigation or correlation. If you have set file size of 100 MB and the number of old log files to keep as 10, you would approximate have 1 GB of log data that you could potentially use for your analysis.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.26
Ensure that the `--request-timeout` argument is set as appropriate.
<details>
<summary>Rationale</summary>
Setting global request timeout allows extending the API server request timeout limit to a duration appropriate to the user's connection speed. By default, it is set to 60 seconds which might be problematic on slower connections making cluster resources inaccessible once the data volume for requests exceeds what can be transmitted in 60 seconds. But, setting this timeout limit to be too large can exhaust the API server resources making it prone to Denial-of-Service attack. Hence, it is recommended to set this limit as appropriate and change the default limit of 60 seconds only if needed.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.27
Ensure that the `--service-account-lookup` argument is set to true.
<details>
<summary>Rationale</summary>
If `--service-account-lookup` is not enabled, the apiserver only verifies that the authentication token is valid, and does not validate that the service account token mentioned in the request is actually present in etcd. This allows using a service account token even after the corresponding service account is deleted. This is an example of time of check to time of use security issue.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 1.2.33
Ensure that the `--encryption-provider-config` argument is set as appropriate.
<details>
<summary>Rationale</summary>
`etcd` is a highly available key-value store used by Kubernetes deployments for persistent storage of all of its REST API objects. These objects are sensitive in nature and should be encrypted at rest to avoid any disclosures.

Detailed steps on how to configure secrets encryption in K3s are available in [Secrets Encryption](secrets-encryption.md).
</details>

### Control 1.2.34
Ensure that encryption providers are appropriately configured.
<details>
<summary>Rationale</summary>
Where `etcd` encryption is used, it is important to ensure that the appropriate set of encryption providers is used. Currently, the `aescbc`, `kms` and `secretbox` are likely to be appropriate options.

This can be remediated by passing a valid configuration to `k3s` as outlined above. Detailed steps on how to configure secrets encryption in K3s are available in [Secrets Encryption](secrets-encryption.md).
</details>

### Control 1.3.1
Ensure that the `--terminated-pod-gc-threshold` argument is set as appropriate.
<details>
<summary>Rationale</summary>
Garbage collection is important to ensure sufficient resource availability and avoiding degraded performance and availability. In the worst case, the system might crash or just be unusable for a long period of time. The current setting for garbage collection is 12,500 terminated pods which might be too high for your system to sustain. Based on your system resources and tests, choose an appropriate threshold value to activate garbage collection.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 3.2.1
Ensure that a minimal audit policy is created.
<details>
<summary>Rationale</summary>
Logging is an important detective control for all systems, to detect potential unauthorized access.

This can be remediated by passing controls 1.2.22 - 1.2.25 and verifying their efficacy.
</details>

### Control 4.2.7
Ensure that the `--make-iptables-util-chains` argument is set to true.
<details>
<summary>Rationale</summary>
Kubelets can automatically manage the required changes to iptables based on how you choose your networking options for the pods. It is recommended to let kubelets manage the changes to iptables. This ensures that the iptables configuration remains in sync with pods networking configuration. Manually configuring iptables with dynamic pod network configuration changes might hamper the communication between pods/containers and to the outside world. You might have iptables rules too restrictive or too open.

This can be remediated by passing this argument as a value to the `--kube-apiserver-arg=` argument to `k3s server`. An example can be found below.
</details>

### Control 5.1.5
Ensure that default service accounts are not actively used
<details>
<summary>Rationale</summary>
Kubernetes provides a `default` service account which is used by cluster workloads where no specific service account is assigned to the pod.

Where access to the Kubernetes API from a pod is required, a specific service account should be created for that pod, and rights granted to that service account.

The default service account should be configured such that it does not provide a service account token and does not have any explicit rights assignments.

This can be remediated by updating the `automountServiceAccountToken` field to `false` for the `default` service account in each namespace.

For `default` service accounts in the built-in namespaces (`kube-system`, `kube-public`, `kube-node-lease`, and `default`), K3s does not automatically do this. You can manually update this field on these service accounts to pass the control.
</details>

## Conclusion

If you have followed this guide, your K3s cluster will be configured to comply with the CIS Kubernetes Benchmark. You can review the [CIS Benchmark Self-Assessment Guide](self-assessment.md) to understand the expectations of each of the benchmark's checks and how you can do the same on your cluster.
