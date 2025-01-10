# CP-Kyverno-use-guide
PSS(Pod Security Standards) and cp-policy(Network isolation between namespaces) implemented as Kyverno policies.

<br>

## Table of Contents

1. [검증 - policy 설치 여부 확인](#1)<br>
2. [검증 - 네트워크 격리 확인](#2)<br>
3. [활용 - Rolebinding을 통한 네임스페이스간 네트워크 통신 허용](#3)<br>
4. [활용 - 특정 POD 네트워크 통신 허용 (pod labels)](#4)<br>
5. [활용 - 특정 네임스페이스 네트워크 통신 허용](#5)<br>
6. [기타 - policy 수동 설치 및 삭제](#6)<br>

<br>

### <div id='1'>1. 검증 - policy 설치 여부 확인 </div>
* clusterpolicy 확인 (설명란 참고)
```shell
$ kubectl get clusterpolicy
NAME                             ADMISSION   BACKGROUND   VALIDATE ACTION   READY   AGE   MESSAGE  설명
cp-add-rolebinding-policy        true        true         Audit             True    42h   Ready    rolebinding이 생성되었을때 networkpolicy 생성
cp-cleanup-network-policy        true        true         Audit             True    42h   Ready    rolebinding이 삭제되었을때 cleanup
cp-default-namespace-policy      true        true         Audit             True    42h   Ready    namespace가 생성되었을때 networkpolicy 생성
disallow-capabilities            true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-capabilities-strict     true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-host-namespaces         true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-host-path               true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-host-ports              true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-host-process            true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-privilege-escalation    true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-privileged-containers   true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-proc-mount              true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
disallow-selinux                 true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
require-run-as-non-root-user     true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
require-run-as-nonroot           true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Baseline
restrict-apparmor-profiles       true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Restricted
restrict-seccomp                 true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Restricted
restrict-seccomp-strict          true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Restricted
restrict-sysctls                 true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Restricted
restrict-volume-types            true        true         Audit             True    42h   Ready    PSS(Pod Security Standards) Restricted
```

<br>

### <div id='2'> 2. 검증 - 네트워크 격리 확인 </div>
* 네임스페이스 네트워크 격리 검증을 위한 사전 준비
```shell
$ kubectl create namespace test1
namespace/test1 created

$ kubectl create namespace test2
namespace/test2 created

$ kubectl get networkpolicy -A
NAMESPACE   NAME                           POD-SELECTOR     AGE
test1       cp-default-namespace-policy    <none>           8s
test1       cp-default-pod-shared-policy   cp-role=shared   8s
test2       cp-default-namespace-policy    <none>           6s
test2       cp-default-pod-shared-policy   cp-role=shared   6s

$ kubectl run nginx-test1 --image=nginx -n test1
pod/nginx-test1 created

$ kubectl expose pod/nginx-test1 --type=NodePort --port=80 -n test1
service/nginx-test1 exposed

$ kubectl run nginx-test2 --image=nginx -n test2
pod/nginx-test2 created

$ kubectl expose pod/nginx-test2 --type=NodePort --port=80 -n test2
service/nginx-test2 exposed
```

* test1 -> test2 : 다른 네임스페이스 네트워크 접근 (default : deny)
```shell
$ kubectl exec -it -n test1 nginx-test1 -- /usr/bin/curl -m 3 nginx-test2.test2.svc.cluster.local
curl: (28) Connection timed out after 3000 milliseconds
command terminated with exit code 28
```

* test2 -> test1 : 다른 네임스페이스 네트워크 접근 (default : deny)
```shell
$ kubectl exec -it -n test2 nginx-test2 -- /usr/bin/curl -m 3 nginx-test1.test1.svc.cluster.local
curl: (28) Connection timed out after 3000 milliseconds
command terminated with exit code 28
```

* 노드포트 외부 접근이 가능한지 검증 (default : allow)
```shell
$ kubectl get svc -n test1
NAME          TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-test1   NodePort   10.233.6.161   <none>        80:30839/TCP   19h

$ curl http://master-node-public-ip:nginx-test1-nodeport   #( eg. curl http://115.68.100.111:30839 )
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

<br>

### <div id='3'> 3. 활용 - Rolebinding을 통한 네임스페이스간 네트워크 통신 허용 </div>
* kubectl 을 통한 네임스페이스간 네트워크 통신 허용 (eg. namespace : test1,test2,test3)
```
예제와 같이 test1,test2,test3 네임스페이스를 선택한 경우 test1,test2,test3 네임스페이스간 통신은 허용 되며, 
등록하지 않은 네임스페이스는 격리된다. 
- test1 -> test1 (O) | test1 -> test2 (O) | test1 -> test3 (O) | test1 -> test4 (X) 
- test2 -> test1 (O) | test2 -> test2 (O) | test2 -> test3 (O) | test2 -> test4 (X) 
- test3 -> test1 (O) | test3 -> test2 (O) | test3 -> test3 (O) | test3 -> test4 (X) 
```

```shell
# 네임스페이스 생성
$ kubectl create namespace test1
$ kubectl create namespace test2
$ kubectl create namespace test3

# 서비스어카운트 생성
$ kubectl create serviceaccount test-user --namespace test1

# test1 namespace : Role 생성 & RoleBinding 생성
$ kubectl create role test-role --verb=get,list,watch,create,delete,patch,update --resource=pods,services,deployments -n test1
$ kubectl create rolebinding test-role-binding --role=test-role --serviceaccount=test-namespace:test-user -n test1

# test2 namespace : Role 생성 & RoleBinding 생성
$ kubectl create role test-role --verb=get,list,watch,create,delete,patch,update --resource=pods,services,deployments -n test2
$ kubectl create rolebinding test-role-binding --role=test-role --serviceaccount=test-namespace:test-user -n test2

# test3 namespace : Role 생성 & RoleBinding 생성
$ kubectl create role test-role --verb=get,list,watch,create,delete,patch,update --resource=pods,services,deployments -n test3
$ kubectl create rolebinding test-role-binding --role=test-role --serviceaccount=test-namespace:test-user -n test3

# 네임스페이스간 통신 확인 : test1,test2,test3 네임스페이스에 nginx 배포
$ kubectl run nginx-test1 --image=nginx -n test1
$ kubectl expose pod/nginx-test1 --type=NodePort --port=80 -n test1
$ kubectl run nginx-test2 --image=nginx -n test2
$ kubectl expose pod/nginx-test2 --type=NodePort --port=80 -n test2
$ kubectl run nginx-test3 --image=nginx -n test3
$ kubectl expose pod/nginx-test3 --type=NodePort --port=80 -n test3

# 3개의 네임스페이스가 각 통신이 되는지 검증 
$ kubectl exec -it -n test1 nginx-test1 -- /usr/bin/curl -m 3 nginx-test2.test2.svc.cluster.local
$ kubectl exec -it -n test2 nginx-test2 -- /usr/bin/curl -m 3 nginx-test3.test2.svc.cluster.local
$ kubectl exec -it -n test2 nginx-test3 -- /usr/bin/curl -m 3 nginx-test1.test2.svc.cluster.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

* cp-portal UI 에서 네임스페이스간 네트워크 통신 허용
```
1.Clusters > Namespaces > 생성 > 네임스페이스 생성 (eg. test1,test2,test3) 
2.Managements > Users > User(tab) > 원하는 유저 선택 > 네임스페이스 다수 선택 (eg. test1,test2,test3) > 확인 
- 예제(eg.)와 같이 test1,test2,test3 네임스페이스를 선택한 경우 test1,test2,test3 네임스페이스간 통신은 허용 되며, 
  등록하지 않은 네임스페이스는 격리된다. 
- test1 -> test1 (O) | test1 -> test2 (O) | test1 -> test3 (O) | test1 -> test4 (X) 
- test2 -> test1 (O) | test2 -> test2 (O) | test2 -> test3 (O) | test2 -> test4 (X) 
- test3 -> test1 (O) | test3 -> test2 (O) | test3 -> test3 (O) | test3 -> test4 (X) 
```

<br>

### <div id='4'> 4. 활용 - 특정 POD 네트워크 통신 허용 </div>
* 통신을 허용할 POD에 레이블을 추가 : "cp-role=shared"
* "cp-role=shared" label 이 등록된 POD는 모든 네임스페이스의 POD에서 접근할 수 있음
```shell
$ kubectl label pod/nginx-test2 cp-role=shared -n test2
pod/nginx-test2 labeled
```

* "cp-role=shared" label 이 등록된 POD에 접근 검증 ( if "cp-role=shared" then allow else deny )
```shell
$ kubectl exec -it -n test1 nginx-test1 -- /usr/bin/curl -m 3 nginx-test2.test2.svc.cluster.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

<br>

### <div id='5'> 5. 활용 - 특정 네임스페이스 네트워크 통신 허용 </div>
* 특정 네임스페이스 networkpolicy 추가 
```shell
$ cat << EOF > cp-allow-all-namespace-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cp-allow-all-namespace-policy
spec:
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
  podSelector: {}
  policyTypes:
  - Ingress
EOF

$ kubectl apply -f cp-allow-all-namespace-policy.yaml -n <네임스페이스명>  #( eg. kubectl apply -f all-allow-namespace-policy.yaml -n test1 )
```

<br>

### <div id='6'> 6. 기타 - policy 수동 설치 및 삭제 </div>
* 설치파일 다운로드
```shell
$ git clone https://github.com/K-PaaS/cp-deployment.git -b branch_v1.6.x
```

* 수동설치 
```shell
$ cd ~/cp-deployment/applications/kyverno-1.12.5

# kyverno 설치 
$ kubectl create -f kyverno.yaml

# PSS(Pod Security Standards) 정책 적용
$ kubectl apply -f pod-security-standards.yaml

# cp-policy(Network isolation between namespaces) 정책 적용
$ source deploy-cp-kyverno-policies.sh
```

* 수동삭제
```shell
$ cd ~/cp-deployment/applications/kyverno-1.12.5

# cp-policy(Network isolation between namespaces) 정책 삭제
$ source remove-cp-keyverno-policies.sh

# PSS(Pod Security Standards) 정책 삭제
$ kubectl delete -f pod-security-standards.yaml

# kyverno 삭제
$ kubectl delete -f kyverno.yaml

# networkpolicy 확인후 지워지지 않은 항목이 있다면 수동삭제 #( eg. kubectl delete NetworkPolicy cp-allow-all-namespace-policy -n test1 )
$ kubectl get networkpolicy -A | grep cp-
```

