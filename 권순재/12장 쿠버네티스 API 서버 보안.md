# 12장 쿠버네티스 API 서버 보안

<aside>
⚠️ **Docker for Mac 기반 쿠버네티스에서 실습하는 경우 주의사항**

모든 서비스 어카운트가 cluster-admin 권한이 있는 버그(?)가 있다.
관련 이슈: [https://github.com/docker/for-mac/issues/3694](https://github.com/docker/for-mac/issues/3694)
요약

- 사용자의 편의성을 위해 모든 서비스 어카운트에 cluster admin 권한을 부여함. (clusterrolebindings/docker-for-desktop-binding)
- 해당 사항에 대하여 이슈가 생성되었고, kube-system의 서비스 어카운트에만 cluster admin 권한이 적용되도록 고침.
- 근데 잘못 고침. 그러나 시간이 지나면서 해당 이슈는 잊혀짐.

```yaml
# current
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts
  namespace: kube-system
```

```yaml
# fixed like below
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:kube-system
```

</aside>

## 인증

api서버는 요청을 받으면 누가 요청을 보냈는지 밝혀내려고 한다. 여러 인증 플러그인을 거치며 누구인지 밝혀질 때까지 시도하며, 인증 플러그인을 통해 사용자 이름과 사용자 ID 그리고 그룹을 알아낼 수 있다. 쿠버네티스에서는 해당 인증 정보를 별도로 저장하지는 않으며, 해당 인증 정보를 통해 사용자가 작업을 수행할 권한이 있는지를 판단한다.

쿠버네티스에서는 사용자를 실제 사용자와 파드(즉, 파드 내부에서 실행되는 애플리케이션)로 구분한다. 실제 사용자에 대한 인증은 별도의 외부 시스템에 위임하고 파드는 서비스 어카운트라는 메커니즘을 사용한다.

사용자는 하나 이상의 그룹에 속할 수 있다. 그룹에도 권한을 부여할 수 있으며, 해당 그룹에 속하는 모든 사용자는 해당 그룹에 부여된 권한을 사용할 수 있다. 그룹은 임의의 문자열이지만 특별한 의미를 갖는 내장 그룹도 있다. 

서비스 어카운트는 파드나 컨피그맵과 같은 리소스이며 네임스페이스 범위를 갖는다. 기본적으로 각 네임스페이스마다 default 라는 이름의 서비스 어카운트가 있으며, 생성된 파드는 default 서비스 어카운트를 기본값으로 사용한다. 각 파드는 하나의 서비스 어카운트만 사용할 수 있지만, 여러 파드가 같은 서비스 어카운트를 사용할 수 있다.

## 서비스 어카운트

### 생성

```
k create sa my-sa
```

### 상세정보 조회

```
k describe sa my-sa
Name:                my-sa
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   my-sa-token-r8wnc
Tokens:              my-sa-token-r8wnc
Events:              <none>
```

image pull secrets

파드가 이미지를 pull 할 때 갖는 권한이다. 시크릿 이름을 작성하면 된다. 서비스 어카운트에 이미지 풀 시크릿을 추가하면 각 파드에 개별적으로 추가할 필요가 없다.

mountable secrets

해당 서비스 어카운트를 사용하는 파드만 해당 시크릿을 마운트 할 수 있다. 해당 기능을 활성화하기 위해서는 서비스 어카운트가 `kubernetes.io/enforce-montable-secret="true"` 어노테이션을 포함하고 있어야 한다.

tokens

인증 토큰 등이 포함되어 있는 시크릿이다. 첫 번째 토큰이 컨테이너에 마운트된다.

### 파드에서 사용하기

```
apVersion: v1
kind: Pod
metadata:
 ...
spec:
  serviceAccountName: foo-user
  ...
```

`spec.serviceAccountName` 필드에 서비스 어카운트 이름을 설정하면 된다. 나중에 변경할 수 없다.

## RBAC

쿠버네티스 api 서버는 인가 플러그인을 통해 특정 사용자가 특정 액션을 사용할 수 있는지 검사할 수 있다. 인가 플러그인은 대표적으로 RBAC가 있으며 이 외에도 ABAC, Web Hook  등이 있으며 사용자 정의 플러그인도 가능하다.

REST API의 경우 HTTP 메서드를 통해 수행할 행동의 종류를 나타내고 URL을 통해 리소스를 나타내는 특징이 있다. HTTP 메서드와 쿠버네티스에서 수행할 액션을 매핑시킬 수 있다.

| HTTP 메서드 | 단일 리소스 액션 | 컬렉션 액션 |
| --- | --- | --- |
| GET, HEAD | get, watch | list, watch |
| POST | create |  |
| PUT | update |  |
| PATCH | patch |  |
| DELETE | delete | deletecollection |

RBAC는 전체 리소스 유형에 권한을 설정하는 것 뿐만 아니라 특정 리소스 인스턴스 혹은 리소스가 아닌 URL 경로에도 권한을 설정할 수 있다.

RBAC는 총 네 개의 리소스로 구성된다.

롤, 클러스터롤, 롤바인딩, 클러스터롤바인딩

롤과 클러스터롤은 수행할 수 있는 작업을 정의하고, 롤바인딩과 클러스터롤바인딩은 누가 이를 수행할 수 있는지를 정의한다.

롤과 클러스터롤 그리고 롤바인딩과 클러스터롤바인딩의 차이점은 적용 범위가 네임스페이스 수준인가 클러스터 수준인지의 차이가 있다.

### 실습환경 구성

foo namespace를 생성하고 proxy 파드를 생성

```
kubectl run test --image=luksa/kubectl-proxy -n foo
```

bar namespace를 생성하고 proxy 파드를 생성

```
kubectl run test --image=luksa/kubectl-proxy -n bar
```

그리고 리소스 조회 요청을 통해 권한이 없는 것을 확인한다.

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/namespaces/foo/services
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"foo\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}
```

```
kubectl -it exec kubectl-proxy -n bar -- curl localhost:8001/api/v1/namespaces/bar/services
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:bar:default\" cannot list resource \"services\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}
```

### 롤 생성

템플릿을 통해 생성할 수 있다.

```yaml
# service-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: service-reader
  namespace: foo
rules:
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
```

```
kubectl create -f service-reader.yaml
```

명령어로도 생성할 수 있다.

```
kubectl create role service-reader --verb=get --verb=list --resource=services -n bar
```

### 롤바인딩 생성

foo:default에 롤바인딩 (템플릿 사용)

```
# service-reader-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: service-reader:default
  namespace: foo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```

```
kubectl apply -f service-reader-binding.yaml
```

bar:default에 롤바인딩 (명령어 사용)

```
kubectl create rolebinding service-reader:default --role=service-reader --serviceaccount=bar:default -n bar
```

### 테스트

foo → foo service 조회 = 성공

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/namespaces/foo/services
```

```
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "7196"
  },
  "items": []
}
```

bar → bar service 조회 = 성공

```
kubectl -it exec kubectl-proxy -n bar -- curl localhost:8001/api/v1/namespaces/bar/services
```

```
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "7236"
  },
  "items": []
}
```

foo → bar service 조회 = 실패

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/namespaces/bar/services
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}
```

<aside>
💡 kubectl auth can-i 명령어를 통해서도 서비스 어카운트의 권한을 테스트할 수 있다.

```
kubectl auth can-i list services -n foo --as system:serviceaccount:foo:default
```

</aside>

<aside>
💡 kubectl proxy 명령어를 통해 생성된 터널에 직접 API 요청을 보내 권한을 테스트할 수 있다. API 요청을 보낼 때 Header에 서비스 어카운트의 토큰값을 포함시켜야 한다.

```
FOO_SECRET=$(kubectl get sa -n foo foo-user -o jsonpath='{.secrets[0].name}')
FOO_TOKEN=$(kubectl get secret -n foo $FOO_SECRET -o jsonpath='{.data.token}' | base64 -d)
curl localhost:8001/api/v1/namespaces/foo/services -H "Authorization=Bearer $FOO_TOKEN"
```

</aside>

롤바인딩과 롤은 1:1 관계이지만 롤바인딩과 참조 사용자는 1:N 관계임도 같이 알아두자.

### 다른 네임스페이스를 참조하는 롤바인딩

롤바인딩에 다른 네임스페이스의 서비스어카운트를 참조 사용자로 작성할 수 있다. 즉, foo의 default가 bar 네임스페이스이 서비스 조회 권한이 가능하다는 것이다.

롤바인딩 생성

```
# service-reader-binding-for-foo.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: service-reader:default:foo
  namespace: bar
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```

```
kubectl apply -f service-reader-binding-for-foo.yaml
```

또는 명령어 이용

```
kubectl create rolebinding service-reader:default:foo --role=service-reader --serviceaccount=foo:default -n bar
```

foo → bar service 조회 = 성공

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/namespaces/bar/services
```

```
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "9404"
  },
  "items": []
}
```

### 클러스터롤과 클러스터롤바인딩

일반적인 롤은 롤이 위치하고 있는 네임스페이스의 리소스에만 액세스할 수 있는 권한을 부여한다. 다른 네임스페이스에 권한을 부여하려면 각 네임스페이스마다 롤과 롤바인딩을 생성해야 한다.

그리고 pv처럼 네임스페이스 범위에 속하지 않는 리소스도 있다. 이러한 경우 클러스터롤을 사용한다. 또한 `/healthz` 와 같이 리소스에 대한 액세스 권한 부여도 클러스터롤을 통해 가능하다.

### 모든 네임스페이스의 리소스 조회

foo → bar pod 조회 = 실패

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/pods
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

pod-reader 클러스터과 클러스터롤바인딩 생성

```
kubectl create clusterrole pod-reader --verb=get,list --resource=pods
```

```
kubectl create clusterrolebinding pod-reader:default:foo --clusterrole=pod-reader --serviceaccount=foo:default
```

테스트

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/pods | head -n 10
```

```
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "10302"
  },
  "items": [
    {
      "metadata": {
        "name": "kubectl-proxy",
```

### 클러스터 리소스 조회

foo → pv 조회 = 실패

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/persistentvolumes
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "persistentvolumes is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"persistentvolumes\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "persistentvolumes"
  },
  "code": 403
}
```

pv-reader 클러스터롤과 클러스터롤바인딩 생성

```
kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
```

```
kubectl create clusterrolebinding pv-reader:default:foo --clusterrole=pv-reader --serviceaccount=foo:default
```

테스트

```
kubectl -it exec kubectl-proxy -n foo -- curl localhost:8001/api/v1/persistentvolumes
```

```
{
  "kind": "PersistentVolumeList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "10473"
  },
  "items": []
}
```

URL 액세스 허용하기

기본적으로 `system:discovery` 이름으로 클러스터롤과 클러스터롤바인딩이 있음

<aside>
💡 URL의 경우 verbs에 create, update 대신 post, put, patch 와 같이 HTTP 메서드를 입력한다.

</aside>

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
rules:
- nonResourceURLs:
  - /api
  - /api/*
  - /apis
  - /apis/*
  - /healthz
  - /livez
  - /openapi
  - /openapi/*
  - /readyz
  - /version
  - /version/
  verbs:
  - get
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:discovery
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

### 클러스터롤과 롤바인딩을 사용하기

롤바인딩에서 클러스터롤을 사용할 수도 있다.

bar → bar pod 조회 = 실패

```
kubectl -it exec kubectl-proxy -n bar -- curl localhost:8001/api/v1/namespaces/bar/pods
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:bar:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

롤바인딩 생성

기존에 생성했던 pod-reader 클러스터롤을 사용해보자

```
kubectl create rolebinding pod-reader:default:bar --clusterrole=pod-reader --serviceaccount=bar:default -n bar
```

```

bar → bar pod 조회 = 성공

```
kubectl -it exec kubectl-proxy -n bar -- curl localhost:8001/api/v1/namespaces/bar/pods | head -n 10
```

```
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "10943"
  },
  "items": [
    {
      "metadata": {
        "name": "kubectl-proxy",
```

그럼 bar는 foo의 pod도 조회할 수 있을까?

bar → foo pod 조회 = 실패

```
kubectl -it exec kubectl-proxy -n bar -- curl localhost:8001/api/v1/namespaces/foo/pods | head -n 10
```

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:bar:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"foo\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

클러스터롤과 롤바인딩을 사용하는 경우 결과 자체는 롤과 롤바인딩을 사용하는 것과 차이가 없다. 그러나 롤은 네임스페이서 범위의 리소스이므로 특정 네임스페이스만 사용이 가능하다. 보편적인 권한을 클러스터롤로 정의하고 모든 네임스페이스에서 재사용할 수 있다는 장점이 있다.

**롤, 클러스터롤, 롤바인딩, 클러스터롤바인딩 조합 요약 표**

| 접근권한 | 롤 타입 | 바인딩 타입 |
| --- | --- | --- |
| 클러스터 범위 리소스 | 클러스터롤 | 클러스터롤바인딩 |
| URL 액세스 | 클러스터롤 | 클러스터롤바인딩 |
| 모든 네임스페이스의 네임스페이스 범위 리소스 | 클러스터롤 | 클러스터롤바인딩 |
| 특정 네임스페이스의 네임스페이스 범위 리소스 | 클러스터롤 | 롤바인딩 |
| 특정 네임스페이스의 네임스페이스 범위 리소스 | 롤 | 롤바인딩 |

## 기본적으로 생성되는 클러스터롤과 클러스터롤바인딩

쿠버네티스 클러스터에서 기본적으로 생성되는 클러스터롤과 클러스터롤바인딩이 있다. 대부분 시스템을 위해 사용되는 것이므로 크게 신경 쓸 필요는 없지만 다음 4가지는 쿠버네티스 사용자(관리자)가 사용할 수 있도록 편의를 위해 제공된 것이므로 알아두면 좋다.

## view

롤과 롤바인딩 그리고 시크릿을 제외한 네임스페이스 범위의 리소스를 읽을 수 있다.

## edit

네임스페이스 범위의 리소스를 수정할 수 있을뿐만 아니라 시크릿을 읽고 수정할 수 있다. 그러나 롤과 롤바인딩을 읽거나 수정하는 것은 불가능하다.

## admin

리소스쿼터와 네임스페이스 리소스 자체를 제외하고 네임스페이스 내의 모든 리소스를 읽고 수정할 수 있다. 롤과 롤바인딩도 읽고 수정할 수 있다는 점이 큰 차이점이다.

<aside>
💡 admin이 있으면 결국 리소스쿼터와 네임스페이스 리소스에 대한 권한이 있는 롤과 롤바인딩을 생성할 수 있지 않냐고 의심할 수 있다. API 서버에서는 권한 상승 방지를 위해 사용자가 가지고 있는 권한에 대해서만 롤을 만들고 업데이트할 수 있다.

</aside>

## cluster-admin

리소스쿼터 및 네임스페이스 리소스 수정 권한까지 포함하여 모든 권한을 가지고 있다.

위 4가지 클러스터롤과 롤바인딩을 적절하게 사용하여 사용자(실제 사용자, 서비스 어카운트, 그룹)에게 권한을 분배할 수 있다.