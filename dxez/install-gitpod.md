# Install GitPod

### GitPod 설치

2020년에 출시되었으며, '즉시 개발 가능한' 환경을 자동화하는 오픈소스 플랫폼

* 설치 문서

```
1. https://www.gitpod.io/docs/configure/self-hosted/helm-deprecated/installation/on-kubernetes
2. https://www.gitpod.io/docs/configure/self-hosted/latest/installing-gitpod
```

* 설치 환경 요구사항

```
- linux kernel > 5.4
- ubuntu 20.04 and 22.04
- kubernetes version 1.23.10
- orchestration Platform : k3s
- containerd 1.5 and 1.6
- cni: calico vxlan
- cert-manager > 1.5
- database : mysql
- image repository : minio aws s3
- At least 4 vCPU and 16GB of RAM
- Load Balancer
- k8s cluster (do not use docker)
```

* 사전 준비 작업 (내부 pod를 dns를 통해 호출 )

> coredns 설정 변경

```bash
$ kubectl -n kube-system edit cm coredns # ready 와 kubernetes cluster.local 사이
```

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        rewrite stop {
           name regex registry.gitpod.handymes.com proxy.gitpod.svc.cluster.local
           answer name prox.gitpod.svc.cluster.local registry.gitpod.handymes.com
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

> node label (모든 worker node)

```bash
$ kubectl label node node1 gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true
$ kubectl label node node2 gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true
$ kubectl label node node3 gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true
```

> gitpod 설치도구 kots 설치 (root 사용자)

```bash
sudo -i
curl https://kots.io/install | bash
kubectl kots install gitpod
```

> dns 서버에 아래sodyd A record로 등록 (gitpod proxy loadbalancer IP)

```
gitlab.handymes.com
@.gitpod.handymes.com
*.gitpod.handymes.com
*.ws.gitpod.handymes.co
```

> gitpod 설정 콘솔

```bash
kubectl kots admin-console --namespace gitpod
```
