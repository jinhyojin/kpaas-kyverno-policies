# CP-Kyverno-install-guide
PSS(Pod Security Standards) and cp-policy(Network isolation between namespaces) implemented as Kyverno policies.

## 1. Kyverno
* kubernetes 버전 : v1.30.4 / Kyverno 설치 버전 : v1.12.5
* Kyverno 정책 목록
```
CP-policies (policies are based on the Kubernetes Pod Security Standards definitions)
├── pod-security
│   ├── baseline
│   │   ├── disallow-capabilities
│   │   ├── disallow-host-namespaces
│   │   ├── disallow-host-path
│   │   ├── disallow-host-ports
│   │   ├── disallow-host-ports-range
│   │   ├── disallow-host-process
│   │   ├── disallow-privileged-containers
│   │   ├── disallow-proc-mount
│   │   ├── disallow-selinux
│   │   ├── restrict-apparmor-profiles
│   │   ├── restrict-seccomp
│   │   └── restrict-sysctls
│   ├── restricted
│   │   ├── disallow-capabilities-strict
│   │   ├── disallow-privilege-escalation
│   │   ├── require-run-as-non-root-user
│   │   ├── require-run-as-nonroot
│   │   ├── restrict-seccomp-strict
│   │   └── restrict-volume-types
```
* [PaaS 개발자 가이드](cp-kyverno-dev-guide-v1.md)

### 1.1 Install Kyverno using Helm
* https://kyverno.io/docs/installation/methods/
* In order to install Kyverno with Helm, first add the Kyverno Helm repository.
```
$ helm repo add kyverno https://kyverno.github.io/kyverno/
```

* Scan the new repository for charts.
```
$ helm repo update
```

* Optionally, show all available chart versions for Kyverno.
```
$ helm search repo kyverno -l
```

* Install Kyverno using Helm
```
# High Availability
$ helm install kyverno kyverno/kyverno -n kyverno --create-namespace --set replicaCount=3
```

```
# Standalone
$ helm install kyverno kyverno/kyverno -n kyverno --create-namespace 
```

* Optionally, Install Kyverno using YAMLs
```
$ kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.12.5/install.yaml
```


### 1.2 PSS(pod-security-standards) 정책 배포
* Install Kyverno-policies using kustomize 
```
# install kustomize 
$ curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

```
# PSS 정책 배포 (kustomize)
$ ./kustomize build https://github.com/kyverno/policies/pod-security | kubectl apply -f-
```

* Optionally, Install Kyverno-policies using YAMLs (아래 URL에서 다운로드 후 배포)
* https://github.com/kyverno/policies/tree/main/pod-security 


### 1.3 CP정책 : 네임스페이스 격리 정책 배포
* 1.3.1 install net-tools 
```
$ sudo apt install net-tools -y
```

* 1.3.2 create deploy file
```
$ vi deploy-cp-kyverno-policies.sh
```
```
#!/bin/bash

############################################################################################
# 0. patch clusterrole : Add 'update' role to namespace (origin : get,list,watch)          #
############################################################################################
kubectl patch clusterrole kyverno:background-controller:core --type='json' -p='[{"op": "replace", "path": "/rules/2", "value":{ "apiGroups": [""], "resources": ["namespaces","configmaps"], "verbs": ["get","list","watch","update"]}}]'


############################################################################################
# 1.네임스페이스 정책 생성 및 배포 (If Namespace is created then NetworkPolicy is create.)  #
############################################################################################

## Add nodes IPv4VXLANTunnelAddr
echo "            # You must find the vxlan.calico IP through the ifconfig command on the Master Node." > IPv4VXLANTunnelAddr.yaml

for IPv4VXLANTunnelAddr in `route -n | grep -e '*' -e 'vxlan.calico' | cut -d ' ' -f 1`
do
echo "            - ipBlock:" >> IPv4VXLANTunnelAddr.yaml
echo "                cidr: ${IPv4VXLANTunnelAddr}/32" >> IPv4VXLANTunnelAddr.yaml
done

IPv4VXLANTunnelAddrYaml=`cat IPv4VXLANTunnelAddr.yaml`

cat << EOF > cp-default-namespace-policy.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cp-default-namespace-policy
spec:
  rules:
  - name: cp-default-namespace-rule
    match:
      any:
      - resources:
          kinds:
          - Namespace
    exclude:
      any:
      - resources:
          namespaces:
          - ingress-nginx
          - istio-system
          - metallb-system
          - kubeedge
          - kubeflow
          - kubeflow-user-example-com
          - knative-eventing
          - knative-serving
          - auth
          - cert-manager
          - keycloak
          - harbor
          - vault
          - mariadb
          - cp-portal
          - cp-pipeline
          - cp-source-control
          - chaos-mesh
          - chartmuseum
          - postgres
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: cp-default-namespace-policy
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      data:
        spec:
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: "{{request.object.metadata.name}}"
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: kube-system
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: metallb-system
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: istio-system
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: ingress-nginx
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: cp-portal
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: chaos-mesh
            - ipBlock:
                cidr: 0.0.0.0/0
                except:
                - 10.233.64.0/18
${IPv4VXLANTunnelAddrYaml}
            - podSelector: {}
          podSelector: {}
          policyTypes:
          - Ingress
  - name: cp-default-pod-shared-rule
    match:
      any:
      - resources:
          kinds:
          - Namespace
    exclude:
      any:
      - resources:
          namespaces:
          - ingress-nginx
          - istio-system
          - metallb-system
          - kubeedge
          - kubeflow
          - kubeflow-user-example-com
          - knative-eventing
          - knative-serving
          - auth
          - cert-manager
          - keycloak
          - harbor
          - vault
          - mariadb
          - cp-portal
          - cp-pipeline
          - cp-source-control
          - chaos-mesh
          - chartmuseum
          - postgres
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: cp-default-pod-shared-policy
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      data:
        spec:
          ingress:
          - from:
            - ipBlock:
                cidr: 10.233.64.0/18
          podSelector:
            matchLabels:
              cp-role: shared
          policyTypes:
          - Ingress
EOF

kubectl apply -f cp-default-namespace-policy.yaml

########################################################################################################################
# 2.롤바인딩 정책 생성 및 배포 (If RoleBinding is created then NetworkPolicy is create and Namespace label is update.)   #
########################################################################################################################

cat << EOF > cp-add-rolebinding-policy.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cp-add-rolebinding-policy
spec:
  rules:
  - name: cp-default-namespace-create-rule
    match:
      any:
      - resources:
          kinds:
          - RoleBinding
    exclude:
      any:
      - resources:
          namespaces:
          - ingress-nginx
          - istio-system
          - metallb-system
          - kubeedge
          - kubeflow
          - kubeflow-user-example-com
          - knative-eventing
          - knative-serving
          - auth
          - cert-manager
          - keycloak
          - harbor
          - vault
          - mariadb
          - cp-portal
          - cp-pipeline
          - cp-source-control
          - chaos-mesh
          - chartmuseum
          - postgres
    mutate:
      targets:
        - apiVersion: v1
          kind: Namespace
          name: "{{ request.object.metadata.namespace }}"
      patchStrategicMerge:
        metadata:
          labels:
            +({{ request.object.metadata.name }}): "true"
  - name: cp-default-namespace-update-rule
    match:
      any:
      - resources:
          kinds:
          - RoleBinding
    exclude:
      any:
      - resources:
          namespaces:
          - ingress-nginx
          - istio-system
          - metallb-system
          - kubeedge
          - kubeflow
          - kubeflow-user-example-com
          - knative-eventing
          - knative-serving
          - auth
          - cert-manager
          - keycloak
          - harbor
          - vault
          - mariadb
          - cp-portal
          - cp-pipeline
          - cp-source-control
          - chaos-mesh
          - chartmuseum
          - postgres
    mutate:
      targets:
        - apiVersion: v1
          kind: Namespace
          name: "{{ request.object.metadata.namespace }}"
      patchStrategicMerge:
        metadata:
          labels:
            "{{ request.object.metadata.name }}": "true"
  - name: cp-default-networkpolicy-create-rule
    match:
      any:
      - resources:
          kinds:
          - RoleBinding
    exclude:
      any:
      - resources:
          namespaces:
          - ingress-nginx
          - istio-system
          - metallb-system
          - kubeedge
          - kubeflow
          - kubeflow-user-example-com
          - knative-eventing
          - knative-serving
          - auth
          - cert-manager
          - keycloak
          - harbor
          - vault
          - mariadb
          - cp-portal
          - cp-pipeline
          - cp-source-control
          - chaos-mesh
          - chartmuseum
          - postgres
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: "{{ request.object.metadata.name }}"
      namespace: "{{request.object.metadata.namespace}}"
      synchronize: true
      data:
        spec:
          podSelector: {}
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  "{{ request.object.metadata.name }}": "true"
          policyTypes:
          - Ingress
EOF
kubectl apply -f cp-add-rolebinding-policy.yaml

#####################################################################################################
# 3.롤바인딩 cleanup 정책 생성 및 배포  (If RoleBinding is deleted then Namespace lable is update.)   #
#####################################################################################################

cat << EOF > cp-cleanup-network-policy.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cp-cleanup-network-policy
spec:
  rules:
  - name: cp-cleanup-network-policy-rule
    match:
      any:
      - resources:
          kinds:
          - RoleBinding
          operations:
          - DELETE
    exclude:
      any:
      - resources:
          namespaces:
          - ingress-nginx
          - istio-system
          - metallb-system
          - kubeedge
          - kubeflow
          - kubeflow-user-example-com
          - knative-eventing
          - knative-serving
          - auth
          - cert-manager
          - keycloak
          - harbor
          - vault
          - mariadb
          - cp-portal
          - cp-pipeline
          - cp-source-control
          - chaos-mesh
          - chartmuseum
          - postgres
    mutate:
      targets:
        - apiVersion: v1
          kind: Namespace
          name: "{{ request.object.metadata.namespace }}"
      patchStrategicMerge:
        metadata:
          labels:
            "{{ request.object.metadata.name }}": "false"
EOF
kubectl apply -f cp-cleanup-network-policy.yaml


# delete yaml
rm IPv4VXLANTunnelAddr.yaml
rm cp-default-namespace-policy.yaml
rm cp-add-rolebinding-policy.yaml
rm cp-cleanup-network-policy.yaml

echo "cp-kyverno-policies deployment is complete."
```

* 1.3.3 deploy
```
$ source deploy-cp-kyverno-policies.sh

clusterrole.rbac.authorization.k8s.io/kyverno:background-controller:core patched
clusterpolicy.kyverno.io/cp-default-namespace-policy created
clusterpolicy.kyverno.io/cp-add-rolebinding-policy created
clusterpolicy.kyverno.io/cp-cleanup-network-policy created
cp-kyverno-policies deployment is complete.

```

* 1.3.4 (Optionally) remove
```
$ kubectl patch clusterrole kyverno:background-controller:core --type='json' -p='[{"op": "replace", "path": "/rules/2", "value":{ "apiGroups": [""], "resources": ["namespaces","configmaps"], "verbs": ["get","list","watch"]}}]'

$ kubectl delete ClusterPolicy cp-default-namespace-policy
$ kubectl delete ClusterPolicy cp-add-rolebinding-policy
$ kubectl delete ClusterPolicy cp-cleanup-network-policy
```


#### 참고. 네트워크 정책으로 할 수 없는 것(적어도 아직은 할 수 없는)

쿠버네티스 1.30부터 다음의 기능은 네트워크폴리시 API에 존재하지 않지만, 운영 체제 컴포넌트(예: SELinux, OpenVSwitch, IPTables 등) 또는 Layer 7 기술(인그레스 컨트롤러, 서비스 메시 구현) 또는 어드미션 컨트롤러를 사용하여 제2의 해결책을 구현할 수 있다. 쿠버네티스의 네트워크 보안을 처음 사용하는 경우, 네트워크폴리시 API를 사용하여 다음의 사용자 스토리를 (아직) 구현할 수 없다는 점에 유의할 필요가 있다.

- 내부 클러스터 트래픽이 공통 게이트웨이를 통과하도록 강제한다(서비스 메시나 기타 프록시와 함께 제공하는 것이 가장 좋을 수 있음).
- TLS와 관련된 모든 것(이를 위해 서비스 메시나 인그레스 컨트롤러 사용).
- 노드별 정책(이에 대해 CIDR 표기법을 사용할 수 있지만, 특히 쿠버네티스 ID로 노드를 대상으로 지정할 수 없음).
- 이름으로 서비스를 타겟팅한다(그러나, 레이블로 파드나 네임스페이스를 타겟팅할 수 있으며, 이는 종종 실행할 수 있는 해결 방법임).
- 타사 공급사가 이행한 "정책 요청"의 생성 또는 관리.
- 모든 네임스페이스나 파드에 적용되는 기본 정책(이를 수행할 수 있는 타사 공급사의 쿠버네티스 배포본 및 프로젝트가 있음).
- 고급 정책 쿼리 및 도달 가능성 도구.
- 네트워크 보안 이벤트를 기록하는 기능(예: 차단되거나 수락된 연결).
- 명시적으로 정책을 거부하는 기능(현재 네트워크폴리시 모델은 기본적으로 거부하며, 허용 규칙을 추가하는 기능만 있음).
- 루프백 또는 들어오는 호스트 트래픽을 방지하는 기능(파드는 현재 로컬 호스트 접근을 차단할 수 없으며, 상주 노드의 접근을 차단할 수 있는 기능도 없음).
