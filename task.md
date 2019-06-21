## Authentication Policy

이 작업에서는 Istio 인증 정책을 활성화, 구성 및 사용할 때 수행해야하는 주요 활동에 대해 설명합니다. [인증(authentication) 개요의 기본 개념](https://istio.io/docs/concepts/security/#authentication)에 대해 자세히 알아보십시오.

### Before you begin

Istio [인증 정책(authentication policy)](https://istio.io/docs/concepts/security/#authentication-policies) 및 관련 [상호 TLS 인증(mutual TLS authentication)](https://istio.io/docs/concepts/security/#mutual-tls-authentication) 개념을 이해합니다.

Global mutual TLS를 사용하지 않고 Kubernetes 클러스터를 Istio와 함께 설치합니다.
>[Installation steps](https://istio.io/docs/setup/kubernetes/install/kubernetes/#installation-steps)에서 설명한대로 <span style="color:red">install/kubernetes /istio-demo.yaml</span>을 사용하거나 [Helm](https://istio.io/docs/setup/kubernetes/install/helm/)을 사용하여 <span style="color:red">global.mtls.enabled</span>를 false로 설정

### Setup

우리의 예제는 두 개의 네임 스페이스 <span style="color:red">foo</span>와 <span style="color:red">bar</span>를 사용하는데 <span style="color:red">httpbin</span>과 <span style="color:red">sleep</span>이라는 두 개의 서비스가 Envoy Sidecar Proxy로 실행됩니다. 또한 <span style="color:red">legacy</span> 네임 스페이스에서 Sidecar 없이 실행되는 <span style="color:red">httpbin</span> 및 <span style="color:red">sleep</span>의 두 번째 인스턴스를 사용합니다. 작업을 시도 할 때 동일한 예제를 사용하려면 다음을 실행하십시오.

```javascript

    $ kubectl create ns foo
    $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
    $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
    $ kubectl create ns bar
    $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n bar
    $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n bar
    $ kubectl create ns legacy
    $ kubectl apply -f samples/httpbin/httpbin.yaml -n legacy
    $ kubectl apply -f samples/sleep/sleep.yaml -n legacy
    
```

네임 스페이스 <span style="color:red">foo</span>, <span style="color:red">bar</span> 또는 <span style="color:red">legacy</span>의 모든 <span style="color:red">sleep</span> pod에서 <span style="color:red">curl</span>으로 HTTP 요청을 <span style="color:red">httpbin.foo</span>, <span style="color:red">httpbin.bar</span> 또는 <span style="color:red">httpbin.legacy</span>로 보내 설정을 확인할 수 있습니다. 모든 요청은 HTTP 코드 200에서 성공해야 합니다.

예를 들어 <span style="color:red">sleep.bar</span>를 <span style="color:red">httpbin.foo</span> 접근성으로 확인하는 명령은 다음과 같습니다.

```javascript

    $ kubectl exec $(kubectl get pod -l app=sleep -n bar -o jsonpath={.items..metadata.name}) -c sleep -n bar -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
    200
    
````

아래의 명령은 모든 접근 가능한 조합을 편리하게 반복합니다.

```javascript

    $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.foo to httpbin.foo: 200
    sleep.foo to httpbin.bar: 200
    sleep.foo to httpbin.legacy: 200
    sleep.bar to httpbin.foo: 200
    sleep.bar to httpbin.bar: 200
    sleep.bar to httpbin.legacy: 200
    sleep.legacy to httpbin.foo: 200
    sleep.legacy to httpbin.bar: 200
    sleep.legacy to httpbin.legacy: 200
    
```

또한 다음과 같이 시스템에 기본 메쉬 인증 정책(default mesh authentication policy)이 있는지 확인해야합니다.

```javascript

    $ kubectl get policies.authentication.istio.io --all-namespaces
    No resources found.

```

```javascript

    $ kubectl get meshpolicies.authentication.istio.io
    NAME      AGE
    default   3m

```

마지막으로, 예제 서비스에 적용되는 대상 규칙(destination rules)이 없는지 확인하세요. 기존 대상 규칙(destination rules)의 <span style="color:red">host:</span> value를 검사하고 일치되지 않는 것이 확인되면 작업을 수행 할 수 있습니다.

예를들면 :

```javascript

    $ kubectl get destinationrules.networking.istio.io --all-namespaces -o yaml | grep "host:"
        host: istio-policy.istio-system.svc.cluster.local
        host: istio-telemetry.istio-system.svc.cluster.local

```
    
>Istio의 버전에 따라 표시된 호스트 이외의 호스트에 대한 대상 규칙이 표시 될 수 있습니다. 그러나 <span style="color:red">foo</span>, <span style="color:red">bar</span> 및 <span style="color:red">legacy</span> 네임 스페이스의 호스트에는 아무 것도 없어야하며 모두 일치인 와일드 카드 *도 없어야합니다.

### Globally enabling Istio mutual TLS

상호 TLS(mutual TLS)를 가능하게하는 메쉬 전체 인증 정책(mesh-wide authentication policy)을 설정하려면 다음과 같이 Mesh 인증 정책(mesh authentication policy)을 제출하십시오.

```javascript

$ kubectl apply -f - <<EOF
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
EOF

```

>Mesh 인증 정책은 클러스터 범위(cluster-scoped)의 <span style="color:red">MeshPolicy</span> CRD에 정의 된 일반 인증 정책 API를 사용합니다.

이 정책은 Mesh의 모든 작업 부하가 TLS를 사용하여 암호화 된 요청만 수락하도록 지정합니다. 보시다시피 이 인증 정책에는 kind: <span style="color:red">MeshPolicy</span>가 있습니다. 정책의 이름은 <span style="color:red">default</span> 이며 <span style="color:red">targets</span> 지정이 없습니다 (Mesh의 모든 서비스에 적용하기 위한 것임).

이 시점에서 수신 측만 상호 TLS를 사용하도록 구성됩니다. Istio 서비스 (Sidecar가 있는 서비스)간에 <span style="color:red">curl</span> 명령을 실행하면 클라이언트 측에서 여전히 일반 텍스트를 사용하므로 503 오류 코드로 모든 요청이 실패합니다.

```javascript

    $ for from in "foo" "bar"; do for to in "foo" "bar"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.foo to httpbin.foo: 503
    sleep.foo to httpbin.bar: 503
    sleep.bar to httpbin.foo: 503
    sleep.bar to httpbin.bar: 503

```
클라이언트 측을 구성하려면 대상 TLS를 사용하도록 대상 규칙(destination rules)을 설정해야합니다. 적용 가능한 각 서비스 (또는 네임 스페이스)마다 하나씩 여러 대상 규칙을 사용할 수 있습니다. 그러나 * 와일드 카드가 있는 규칙을 사용하면 모든 서비스를 일치시켜 메쉬 전체 인증 정책(mesh-wide authentication policy)과 동등하게 적용하는 것이 더 편리합니다.

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "default"
      namespace: "istio-system"
    spec:
      host: "*.local"
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF

```

>- Istio 1.1부터 클라이언트 네임 스페이스, 서버 네임 스페이스 및 글로벌 네임 스페이스 (기본적으로 <span style="color:red">istio-system</span>)의 대상 규칙(destination rules) 만 서비스 순서로 고려됩니다.
>- 호스트 값 <span style="color:red">*.local</span>은 외부 서비스가 아닌 클러스터의 서비스에만 일치하도록 제한합니다. 또한 대상 규칙의 이름 또는 네임 스페이스에는 제한이 없습니다.
>- <span style="color:red">ISTIO_MUTUAL</span> TLS 모드를 사용하면 Istio는 내부 구현에 따라 키와 인증서 (예 : 클라이언트 인증서, 개인 키 및 CA 인증서)의 경로를 설정합니다.

Destination rules이 canarying 설정과 같은 비인증 이유로도 사용되지만 동일한 우선 순위가 적용됨을 잊지 마세요. 따라서 서비스가 어떤 이유로 특정 대상 대상 규칙을 필요로하는 경우 (for example, for a configuration load balancer) 규칙에 <span style="color:red">ISTIO_MUTUAL</span> 모드를 사용하는 유사한 TLS 블록이 있어야합니다. 그렇지 않으면 mesh- 또는 namespace-wide TLS settings을 무시하고 TLS 사용하지 않도록 설정합니다.

위와 같이 testing 명령을 다시 실행하면 Istio 서비스 간의 모든 요청이 성공적으로 완료됩니다.

```javascript

    $ for from in "foo" "bar"; do for to in "foo" "bar"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.foo to httpbin.foo: 200
    sleep.foo to httpbin.bar: 200
    sleep.bar to httpbin.foo: 200
    sleep.bar to httpbin.bar: 200

```

### Request from non-Istio services to Istio services

예를 들어 Sidecar가 없는 <span style="color:red">sleep.legacy</span>와 같은 non-Istio 서비스는 Istio 서비스와 연결에 필요한 TLS Connection을 시작할 수 없습니다. 결과적으로 <span style="color:red">sleep.legacy</span>에서 <span style="color:red">httpbin.foo</span> 또는 <span style="color:red">httpbin.bar</span>로 가는 요청은 실패합니다 :

```javascript

    $ for from in "legacy"; do for to in "foo" "bar"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.legacy to httpbin.foo: 000
    command terminated with exit code 56
    sleep.legacy to httpbin.bar: 000
    command terminated with exit code 56

```

>Envoy가 일반 텍스트 요청(plain-text requests)을 거부하는 방식으로 인하여 이 경우에는 curl 종료 코드 56 (네트워크 데이터 수신 실패)이 표시됩니다.

이는 의도 한대로 작동하지만 유감스럽게도 이러한 서비스에 대한 인증 요구 사항(authentication requirements)을 줄이지 않으면 해결할 수 없습니다.

### Request from Istio services to non-Istio services

<span style="color:red">sleep.foo</span> (또는 <span style="color:red">sleep.bar</span>)에서 <span style="color:red">httpbin.legacy</span>로 요청을 보냅니다. Istio가 상호 TLS를 사용하기 위해 목적지 규칙(destination rule)에 지시한 대로 클라이언트를 구성하므로 요청이 실패하는 것을 볼 수 있지만 httpbin.legacy에는 Sidecar가 없으므로 이를 처리 할 수 없습니다.

```javascript

    $ for from in "foo" "bar"; do for to in "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.foo to httpbin.legacy: 503
    sleep.bar to httpbin.legacy: 503
    
```

이 문제를 해결하기 위해 대상 규칙을 추가하여 <span style="color:red">httpbin.legacy</span>의 TLS 설정을 덮어 쓸 수 있습니다. 

예를 들면 :

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
     name: "httpbin-legacy"
     namespace: "legacy"
    spec:
     host: "httpbin.legacy.svc.cluster.local"
     trafficPolicy:
       tls:
         mode: DISABLE
    EOF
    
```

>이 대상 규칙은 서버의 네임 스페이스 (<span style="color:red">httpbin.legacy</span>)에 있으므로 <span style="color:red">istio-system</span>에 정의 된 global destination rule 보다 선호됩니다.

> https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
> https://arisu1000.tistory.com/27859
> ~/istio-1.1.3$ kubectl exec -it sleep-bcc758cff-sj8qx -n foo -- cat /etc/

### Request from Istio services to Kubernetes API server

Kubernetes API 서버에는 Sidecar가 없으므로 Istio가 없는 서비스(on-Istio service)에 요청을 보낼 때와 같은 문제로 인해 sleep.foo와 같은 Istio 서비스 요청이 실패합니다.

```javascript

    $ TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default-token | cut -f1 -d ' ' | head -1) | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
    $ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl https://kubernetes.default/api --header "Authorization: Bearer $TOKEN" --insecure -s -o /dev/null -w "%{http_code}\n"
    000
    command terminated with exit code 35
    
```

다시 말하지만, API 서버 (kubernetes.default)의 대상 규칙을 재정의하여 이를 수정할 수 있습니다.

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
     name: "api-server"
     namespace: istio-system
    spec:
     host: "kubernetes.default.svc.cluster.local"
     trafficPolicy:
       tls:
         mode: DISABLE
    EOF
    
```

이 규칙은 위의 글로벌 인증 정책(global authentication policy) 및 대상 규칙(destination rule)과 함께 상호 TLS가 활성화 된 Istio를 설치할 때 시스템에 자동으로 주입됩니다(injected).
아래와 같이 규칙을 추가 한 후, 위의 테스트 명령을 다시 실행하여 200을 리턴하는지 확인하세요.

```javascript

    $ TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default-token | cut -f1 -d ' ' | head -1) | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
    $ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl https://kubernetes.default/api --header "Authorization: Bearer $TOKEN" --insecure -s -o /dev/null -w "%{http_code}\n"
    200
    
```

Cleanup part 1

세션에 추가 된 전역 인증 정책(global authentication policy) 및 대상 규칙(destination rules)을 제거합니다.

```javascript

    $ kubectl delete meshpolicy default
    $ kubectl delete destinationrules httpbin-legacy -n legacy
    $ kubectl delete destinationrules api-server -n istio-system

```

### Enable mutual TLS per namespace or service

전체 Mesh에 대한 인증 정책을 지정하는 것 외에도 Istio를 사용하여 특정 네임 스페이스 또는 서비스에 대한 정책을 지정할 수 있습니다. 네임 스페이스 차원의 정책(namespace-wide policy)은 메시 전체의 정책(over the mesh-wide policy)보다 우선 순위가 높지만 서비스 별 정책(service-specific policy)은 우선 순위가 더 높습니다.

### Namespace-wide policy

아래 예제는 네임 스페이스 foo의 모든 서비스에 대해 상호 TLS를 사용하도록 설정하는 정책을 보여줍니다. 보시다시피, "MeshPolicy"가 아닌 kind: “Policy”를 사용하고 네임 스페이스를 지정합니다.이 경우에는 foo입니다. 네임 스페이스 값을 지정하지 않으면 정책이 default 네임 스페이스에 적용됩니다.

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "default"
      namespace: "foo"
    spec:
      peers:
      - mtls: {}
    EOF
    
```

> 메쉬 전체 정책(mesh-wide policy)과 마찬가지로 네임 스페이스 차원의 정책(namespace-wide policy)은 <span style="color:red">default</span>라는 이름을 가져야 하며 특정 서비스를 제한하지 않습니다 (<span style="color:red">targets</span> 섹션 없음)

해당 대상 규칙(destination rule) 추가 :

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "default"
      namespace: "foo"
    spec:
      host: "*.foo.svc.cluster.local"
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF
    
```

> Host <span style="color:red">*.foo.svc.cluster.local</span>은 일치를 foo 네임 스페이스의 서비스에만 제한합니다.

```javascript

    $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.foo to httpbin.foo: 200
    sleep.foo to httpbin.bar: 503
    sleep.foo to httpbin.legacy: 503
    sleep.bar to httpbin.foo: 200
    sleep.bar to httpbin.bar: 503
    sleep.bar to httpbin.legacy: 503
    sleep.legacy to httpbin.foo: 000
    command terminated with exit code 56
    sleep.legacy to httpbin.bar: 200
    sleep.legacy to httpbin.legacy: 200
    
```

### Service-specific policy

특정 서비스에 대한 인증 정책 및 대상 규칙을 설정할 수도 있습니다. 이 명령을 실행하여 <span style="color:red">httpbin.bar</span> 서비스에 대해서만 다른 정책을 설정하십시오.

```javascript

    $ cat <<EOF | kubectl apply -n bar -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "httpbin"
    spec:
      targets:
      - name: httpbin
      peers:
      - mtls: {}
    EOF
    
```

And a destination rule:

```javascript

    $ cat <<EOF | kubectl apply -n bar -f -
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "httpbin"
    spec:
      host: "httpbin.bar.svc.cluster.local"
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF
    
```

이 정책 및 대상 규칙은 네임 스페이스 foo의 서비스에만 적용되므로 Sidecar가 없는 Client (<span style="color:red">sleep.legacy</span>)에서 <span style="color:red">httpbin.foo</span>의 요청이 실패하기 시작해야합니다.

```javascript

    $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
    sleep.foo to httpbin.foo: 200
    sleep.foo to httpbin.bar: 200
    sleep.foo to httpbin.legacy: 200
    sleep.bar to httpbin.foo: 200
    sleep.bar to httpbin.bar: 200
    sleep.bar to httpbin.legacy: 200
    sleep.legacy to httpbin.foo: 000
    command terminated with exit code 56
    sleep.legacy to httpbin.bar: 200
    sleep.legacy to httpbin.legacy: 200
    
```

> - 이 예제에서는 Meta Data에 네임 스페이스를 지정하지 않지만 명령 줄 (-n bar)에 넣어두면 동일한 효과가 있습니다.
> - 인증 정책(authentication policy) 및 대상 규칙 이름(destination rule name)에는 제한이 없습니다. 이 예제는 단순화를 위해 서비스 자체의 이름을 사용합니다.

다시 probing 명령을 실행하세요. 예상했든 대로 <span style="color:red">sleep.legacy</span>에서 <span style="color:red">httpbin.bar</span> 로의 요청하면 같은 이유로 실패하기 시작 합니다.


```javascript

    ...
    sleep.legacy to httpbin.bar: 000
    command terminated with exit code 56
    
```


우리가 네임 스페이스 바(namespace bar)에 더 많은 서비스를 가지고 있다면, 그들에 대한 트래픽이 영향을받지 않을 것입니다. 이 동작을 보여주기 위해 더 많은 서비스를 추가하는 대신 정책을 약간 편집하여 특정 포트에 적용합니다.

```javascript

    $ cat <<EOF | kubectl apply -n bar -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "httpbin"
    spec:
      targets:
      - name: httpbin
        ports:
        - number: 1234
      peers:
      - mtls: {}
    EOF
    
```

대상 규칙(destination rule)에 대한 해당 변경 사항은 다음과 같습니다.

```javascript

    $ cat <<EOF | kubectl apply -n bar -f -
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "httpbin"
    spec:
      host: httpbin.bar.svc.cluster.local
      trafficPolicy:
        tls:
          mode: DISABLE
        portLevelSettings:
        - port:
            number: 1234
          tls:
            mode: ISTIO_MUTUAL
    EOF
    
```

이 새로운 정책은 Port <span style="color:red">1234</span>의 <span style="color:red">httpbin</span> 서비스에만 적용됩니다. 따라서 Port <span style="color:red">8000</span>에서 상호 TLS(mutual TLS)가 비활성화되고 <span style="color:red">sleep.legacy</span>의 요청이 다시 시작됩니다.


```javascript

$ kubectl exec $(kubectl get pod -l app=sleep -n legacy -o jsonpath={.items..metadata.name}) -c sleep -n legacy -- curl http://httpbin.bar:8000/ip -s -o /dev/null -w "%{http_code}\n"
200

```

### Policy precedence

서비스 별 정책(service-specific policy)이 네임 스페이스 차원의 정책(namespace-wide policy)보다 우선하는 방식을 설명하기 위해 아래와 같이 <span style="color:red">httpbin.foo</span>의 상호 TLS를 사용하지 않도록 설정하는 정책을 추가 할 수 있습니다. 네임 스페이스 <span style="color:red">foo</span>에있는 모든 서비스에 대해 상호 TLS를 가능하게하고 <span style="color:red">sleep.legacy</span>에서 <span style="color:red">httpbin.foo</span> 로의 요청이 실패하고 있음을 관찰하는 네임 스페이스 차원의 정책(namespace-wide policy)을 이미 만들었습니다 (위 참조).

```javascript

    $ cat <<EOF | kubectl apply -n foo -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "overwrite-example"
    spec:
      targets:
      - name: httpbin
    EOF
    
```

and destination rule:

```javascript

    $ cat <<EOF | kubectl apply -n foo -f -
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "overwrite-example"
    spec:
      host: httpbin.foo.svc.cluster.local
      trafficPolicy:
        tls:
          mode: DISABLE
    EOF
    
```

<span style="color:red">sleep.legacy</span>에서 요청을 다시 실행하면 서비스별 정책(service-specific policy)이 네임 스페이스 전체 정책(namespace-wide policy)보다 우선하는 것을 확인하고 성공 반환 코드를 다시 표시해야합니다 (200).

```javascript

    $ kubectl exec $(kubectl get pod -l app=sleep -n legacy -o jsonpath={.items..metadata.name}) -c sleep -n legacy -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
    200
    
```

### Cleanup part 2

Remove policies and destination rules created in the above steps:

```javascript

    $ kubectl delete policy default overwrite-example -n foo
    $ kubectl delete policy httpbin -n bar
    $ kubectl delete destinationrules default overwrite-example -n foo
    $ kubectl delete destinationrules httpbin -n bar
    
```

### End-user authentication

이 기능을 시험하기 위해서는 유효한 JWT가 필요합니다. JWT는 데모에 사용할 JWKS 엔드 포인트와 일치해야합니다. 이 자습서에서는이 JWT 테스트와 Istio 코드 기반의 JWKS 끝점을 사용합니다.
또한 편의를 위해 <span style="color:red">httpb.foo</span>를 <span style="color:red">ingressgateway</span>를 통해 노출 시키십시오 (자세한 내용은 ingress 작업 참조).

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: httpbin-gateway
      namespace: foo
    spec:
      selector:
        istio: ingressgateway # use Istio default gateway implementation
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "*"
    EOF
    
```

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: httpbin
      namespace: foo
    spec:
      hosts:
      - "*"
      gateways:
      - httpbin-gateway
      http:
      - route:
        - destination:
            port:
              number: 8000
            host: httpbin.foo.svc.cluster.local
    EOF
    
```

#### Get ingress IP

```javascript

    $ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

```

#### And run a test query

```javascript

    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
    200

```

> 시간이 좀 소요됨


이제 <span style="color:red">httpbin.foo</span>에 대한 최종 사용자 JWT가 필요한 정책을 추가하세요. 다음 명령은 <span style="color:red">httpbin.foo</span>에 대한 서비스 특정 정책이 없다고 가정합니다 (위에서 설명한대로 정리를 실행하는 경우에 해당). <span style="color:red">kubectl get policies.authentication.istio.io -n foo</span>을 실행하여 확인할 수 있습니다.

```javascript

    $ cat <<EOF | kubectl apply -n foo -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "jwt-example"
    spec:
      targets:
      - name: httpbin
      origins:
      - jwt:
          issuer: "testing@secure.istio.io"
          jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/jwks.json"
      principalBinding: USE_ORIGIN
    EOF
    
```

이전과 같은 <span style="color:red">curl</span> 명령은 서버가 JWT를 기대하지만 401이 제공되지 않았기 때문에 401 오류 코드와 함께 반환됩니다.

```javascript

    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
    401
    
```

> 시간이 좀 소요 됨
 
위에 생성 된 유효한 토큰을 첨부하면 성공을 반환합니다.

```javascript

    $ TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/demo.jwt -s)
    $ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
    200
    
```

> 시간이 좀 소요됨 

JWT 유효성 검사(JWT validation)의 다른 측면을 관찰하려면 <span style="color:red">gen-jwt.py</span> 스크립트를 사용하여 다른 발급자, 대상, 만료 날짜 등을 테스트 할 새로운 토큰을 생성하세요. 스크립트는 Istio 저장소에서 다운로드 할 수 있습니다.

```javascript

    $ wget https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/gen-jwt.py
    $ chmod +x gen-jwt.py
    
```

You also need the <span style="color:red">key.pem</span> file:

```javascript

    $ wget https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/key.pem

```

예를 들어, 아래 명령은 5 초 후에 만료되는 토큰을 만듭니다. 보시다시피 Istio는 먼저 해당 토큰을 사용하여 요청을 인증하지만 5 초 후에 거부합니다.


```javascript

    $ TOKEN=$(./gen-jwt.py ./key.pem --expire 5)
    $ for i in `seq 1 10`; do curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"; sleep 1; done
    200
    200
    200
    200
    200
    401
    401
    401
    401
    401
    
```

> gen-jwt.py 실행할 때 jwcrypto 모듈이 없어서 Error가 발생하면 
> pip install jwcrypto 명령으로 해당 모듈 설치 후 제실행할 것

또한 ingress gateway에 JWT 정책을 추가 할 수도 있습니다 (예 : service <span style="color:red">istio-ingressgateway.istio-system.svc.cluster.local</span>). 이것은 종종 개별 서비스 대신 게이트웨이에 바인딩 된 모든 서비스에 대한 JWT 정책을 정의하는 데 사용됩니다.

### End-user authentication with per-path requirements

최종 사용자 인증(End-user authentication)은 요청 경로(request path)를 기반으로 활성화 또는 비활성화 할 수 있습니다. 일부 경로 (예 : 상태 확인 또는 상태 보고서에 사용되는 경로)에 대해 인증을 사용하지 않으려는 경우에 유용합니다. 예를 들어 health check 또는 status report, 다른 경로(different paths)에서 다른 JWT 요구 사항(JWT requirements)을 지정할 수도 있습니다.

>경로 별 요구 사항(per-path requirements)을 가진 최종 사용자 인증(end-user authentication)은 Istio 1.1의 실험적 기능이므로 프로덕션 용도로 권장되지 않습니다.

### Disable End-user authentication for specific paths

path <span style="color:red">/user-agent</span>에 대한 최종 사용자 인증(End-user authentication)을 사용하지 않도록 <span style="color:red">jwt-example</span> 정책을 수정합니다.

```javascript

    $ cat <<EOF | kubectl apply -n foo -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "jwt-example"
    spec:
      targets:
      - name: httpbin
      origins:
      - jwt:
          issuer: "testing@secure.istio.io"
          jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/jwks.json"
          trigger_rules:
          - excluded_paths:
            - exact: /user-agent
      principalBinding: USE_ORIGIN
    EOF
    
```

JWT 토큰없이 path <span style="color:red">/user-agent</span>에 액세스 할 수 있는지 확인합니다.

```javascript

    $ curl $INGRESS_HOST/user-agent -s -o /dev/null -w "%{http_code}\n"
    200
    
```

Confirm it’s denied to access paths other than /user-agent without JWT tokens:
JWT 토큰없이 /user-agent 이외의 경로에 액세스하는 것이 거부되었는지 확인합니다.

```javascript

    $ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
    401
    
```

### Enable End-user authentication for specific paths

경로 <span style="color:red">/ip</span>에 대해서만 최종 사용자 인증(End-user authentication)을 사용하도록 <span style="color:red">jwt-example</span> 정책을 수정합니다.

```javascript

    $ cat <<EOF | kubectl apply -n foo -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "jwt-example"
    spec:
      targets:
      - name: httpbin
      origins:
      - jwt:
          issuer: "testing@secure.istio.io"
          jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/jwks.json"
          trigger_rules:
          - included_paths:
            - exact: /ip
      principalBinding: USE_ORIGIN
    EOF
    
```

JWT 토큰없이 <span style="color:red">/ip</span> 이외의 경로에 액세스 할 수 있는지 확인합니다.

```javascript

    $ curl $INGRESS_HOST/user-agent -s -o /dev/null -w "%{http_code}\n"
    200
    
```

JWT 토큰없이 경로 <span style="color:red">/ip</span>에 액세스하는 것이 거부되었는지 확인합니다.

```javascript

    $ curl $INGRESS_HOST/ip -s -o /dev/null -w "%{http_code}\n"
    401
    
```

유효한 JWT 토큰을 사용하여 경로 <span style="color:red">/ip</span>에 액세스 할 수 있는지 확인합니다.

```javascript

    $ TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/demo.jwt -s)
    $ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/ip -s -o /dev/null -w "%{http_code}\n"
    200
    
```

### End-user authentication with mutual TLS

최종 사용자 인증(End-user authentication) 및 상호 TLS(mutual TLS)를 함께 사용할 수 있습니다. 위의 정책을 수정하여 상호 TLS 및 최종 사용자 JWT 인증(end-user JWT authentication)을 모두 정의하세요.

```javascript

    $ cat <<EOF | kubectl apply -n foo -f -
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "jwt-example"
    spec:
      targets:
      - name: httpbin
      peers:
      - mtls: {}
      origins:
      - jwt:
          issuer: "testing@secure.istio.io"
          jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/jwks.json"
      principalBinding: USE_ORIGIN
    EOF
    
```

> <span style="color:red">jwt-example</span> 정책을 제출하지 않은 경우 <span style="color:red">istio</span> create를 사용하세요.

And add a destination rule:

```javascript

    $ kubectl apply -f - <<EOF
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "httpbin"
      namespace: "foo"
    spec:
      host: "httpbin.foo.svc.cluster.local"
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF
    
```

> 이미 상호 TLS(mutual TLS) 메시 전체(mesh-wide) 또는 네임 스페이스 전체(namespace-wide)를 사용하도록 설정 한 경우 호스트 <span style="color:red">httpbin.foo</span>는 다른 대상 규칙(other destination rule)에 이미 적용됩니다. 따라서 이 대상 규칙을 추가 할 필요가 없습니다. 한편, 서비스 특정 정책(service-specific policy)은 mesh-wide (or namespace-wide) 정책을 완전히 무시하므로 <span style="color:red">mtls</span> stanza를 인증 정책에 추가해야 합니다.

이러한 변경 후에는 ingress gateway를 포함한 Istio 서비스에서 <span style="color:red">httpbin.foo</span>로 가는 트래픽이 상호 TLS(mutual TLS)를 사용합니다. 위의 테스트 명령은 계속 작동합니다. 올바른 토큰이 주어지면, <span style="color:red">httpbin.foo</span>로 직접 Istio 서비스로 부터의 요청도 가능합니다. : -> 아래와 같이

```javascript
    
    $ TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.1/security/tools/jwt/samples/demo.jwt -s)
    $ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n" --header "Authorization: Bearer $TOKEN"
    200
    
```

그러나 일반 텍스트(plain-text)를 사용하는 비 Istio 서비스(non-Istio services)의 요청은 실패합니다.

```javascript

    $ kubectl exec $(kubectl get pod -l app=sleep -n legacy -o jsonpath={.items..metadata.name}) -c sleep -n legacy -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n" --header "Authorization: Bearer $TOKEN"
    000
    command terminated with exit code 56
    
```

### Cleanup part 3

1. Remove authentication policy:

```javascript

    $ kubectl -n foo delete policy jwt-example

```

2. Remove destination rule:

```javascript

    $ kubectl -n foo delete destinationrule httpbin

```

3. If you are not planning to explore any follow-on tasks, you can remove all resources simply by deleting test namespaces.

```javascript

    $ kubectl delete ns foo bar legacy

```

<br><br>

## [Authorization for HTTP Services](https://istio.io/docs/tasks/security/authz-http/)

<br><br>

## [Authorization for TCP Services](https://istio.io/docs/tasks/security/authz-tcp/)

<br><br>

## Authorization for groups and list claims

<br><br>

## Authorization permissive mode

[권한 허용 모드(authorization permissive mode)](https://istio.io/docs/concepts/security/#authorization-permissive-mode)에서는 프로덕션 환경에서 적용하기 전에 권한 policy(authorization policies)를 검증 할 수 있습니다.

허가 허용 모드(authorization permissive mode)는 버전 1.1의 시험적인 기능입니다. 인터페이스는 향후 릴리스에서 변경 될 수 있습니다. 허용 모드(permissive mode) 기능을 시험하고 싶지 않은 경우 Istio 인증(Istio authorization)을 직접 [허용 모드(permissive mode)로 설정](https://istio.io/docs/tasks/security/authz-http/#enabling-istio-authorization)하여 건너 뛰게 할 수 있습니다.

이 작업에서는 권한 부여(authorization)를 위해 허용 모드(permissive mode) 사용과 관련된 두 가지 시나리오를 다룹니다.

- 권한 부여(authorization)가 사용 불가능한 환경의 경우,이 태스크는 권한 부여(authorization)를 사용하는 것이 안전한지 여부를 테스트하는데 도움이됩니다.
- 권한 부여(authorization)가 사용 가능한 환경의 경우,이 태스크는 새로운 권한(authorization) policy를 추가하는 것이 안전한지 여부를 테스트하는데 도움이됩니다.

### Before you begin

이 작업을 완료하려면 먼저 다음 작업을 수행해야합니다.

- Read the authorization concept.
- Follow the instructions in the Kubernetes quick start to install Istio with mutual TLS enabled.
- Deploy the Bookinfo sample application.

- Bookinfo 응용 프로그램에 대한 서비스 계정(service accounts)을 만듭니다. 다음 명령을 실행하여 제품 페이지에 대한 서비스 계정 bookinfo-productpage 및 검토를 위한 서비스 계정 bookinfo-reviews를 만듭니다.
    ```javascript
    
        $ kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo-add-serviceaccount.yaml)
    
    ```

### Test enabling authorization globally

다음 단계에서는 승인 허용 모드(authorization permissive mode)를 사용하여 globally 인증(authorization)을 사용하는 것이 안전한지 테스트하는 방법을 보여줍니다.

1. To enable the permissive mode in the global authorization configuration, run the following command:

    ```javascript
    
    $ kubectl apply -f - <<EOF
    apiVersion: "rbac.istio.io/v1alpha1"
    kind: ClusterRbacConfig
    metadata:
      name: default
    spec:
      mode: 'ON_WITH_INCLUSION'
      inclusion:
        namespaces: ["default"]
      enforcement_mode: PERMISSIVE
    EOF
    
    ```
    
2. Go to the productpage at http://$GATEWAY_URL/productpage and verify that everything works fine.

3. Apply the rbac-permissive-telemetry.yaml YAML file to enable the metric collection for the permissive mode:

    ```javascript
    
    $ kubectl apply -f samples/bookinfo/platform/kube/rbac/rbac-permissive-telemetry.yaml
    logentry.config.istio.io/rbacsamplelog created
    stdio.config.istio.io/rbacsamplehandler created
    rule.config.istio.io/rabcsamplestdio created
    
    ```
4. Send traffic to the sample application with the following command:

    ```javascript
    
    $ curl http://$GATEWAY_URL/productpage
    
    ```

5. Go to the productpage at http://$GATEWAY_URL/productpage and verify that everything works fine.

6. Get the log for telemetry and search for the permissiveResponseCode with the following command:

    ```javascript
    
    $ kubectl -n istio-system logs -l istio-mixer-type=telemetry -c mixer | grep \"instance\":\"rbacsamplelog.logentry.istio-system\"
    {"level":"warn","time":"2018-08-30T21:53:42.059444Z","instance":"rbacsamplelog.logentry.istio-system","destination":"ratings","latency":"9.158879ms","permissiveResponseCode":"denied","permissiveResponsePolicyID":"","responseCode":200,"responseSize":48,"source":"reviews","user":"cluster.local/ns/default/sa/bookinfo-reviews"}
    {"level":"warn","time":"2018-08-30T21:53:41.037824Z","instance":"rbacsamplelog.logentry.istio-system","destination":"reviews","latency":"1.091670916s","permissiveResponseCode":"denied","permissiveResponsePolicyID":"","responseCode":200,"responseSize":379,"source":"productpage","user":"cluster.local/ns/default/sa/bookinfo-productpage"}
    {"level":"warn","time":"2018-08-30T21:53:41.019851Z","instance":"rbacsamplelog.logentry.istio-system","destination":"productpage","latency":"1.112521495s","permissiveResponseCode":"denied","permissiveResponsePolicyID":"","responseCode":200,"responseSize":5723,"source":"istio-ingressgateway","user":"cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"}
    
    ```
    
7. Verify that the the log shows a responseCode of 200 and a permissiveResponseCode of denied.

8. Apply the productpage-policy.yaml authorization policy in permissive mode with the following command:

    ```javascript
    
    $ kubectl apply -f samples/bookinfo/platform/kube/rbac/productpage-policy.yaml
    
    ```

9. Send traffic to the sample application with the following command:

    ```javascript
    
    $ curl http://$GATEWAY_URL/productpage
    
    ```

10. Get the log for telemetry and search for the permissiveResponseCode with the following command:

```javascript

$ kubectl -n istio-system logs -l istio-mixer-type=telemetry -c mixer | grep \"instance\":\"rbacsamplelog.logentry.istio-system\"
{"level":"warn","time":"2018-08-30T21:55:53.590430Z","instance":"rbacsamplelog.logentry.istio-system","destination":"ratings","latency":"4.415633ms","permissiveResponseCode":"denied","permissiveResponsePolicyID":"","responseCode":200,"responseSize":48,"source":"reviews","user":"cluster.local/ns/default/sa/bookinfo-reviews"}
{"level":"warn","time":"2018-08-30T21:55:53.565914Z","instance":"rbacsamplelog.logentry.istio-system","destination":"reviews","latency":"32.97524ms","permissiveResponseCode":"denied","permissiveResponsePolicyID":"","responseCode":200,"responseSize":379,"source":"productpage","user":"cluster.local/ns/default/sa/bookinfo-productpage"}
{"level":"warn","time":"2018-08-30T21:55:53.544441Z","instance":"rbacsamplelog.logentry.istio-system","destination":"productpage","latency":"57.800056ms","permissiveResponseCode":"allowed","permissiveResponsePolicyID":"productpage-viewer","responseCode":200,"responseSize":5723,"source":"istio-ingressgateway","user":"cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"}

```

11. Verify that the the log shows a responseCode of 200 and a permissiveResponseCode of allowed for productpage service.

12. Remove the YAML files related to enabling the permissive mode:

    ```javascript
    $ kubectl delete -f samples/bookinfo/platform/kube/rbac/productpage-policy.yaml
    $ kubectl delete -f samples/bookinfo/platform/kube/rbac/rbac-config-on-permissive.yaml
    $ kubectl delete -f samples/bookinfo/platform/kube/rbac/rbac-permissive-telemetry.yaml
    ```
    
13. Congratulations! You tested an authorization policy with permissive mode and verified it works as expected. To enable the authorization policy, follow the steps described in the Enabling Istio authorization task.

### Test adding authorization policy

The following steps show how to test a new authorization policy with permissive mode when authorization has already been enabled.

1. Allow access to the producepage service by following the instructions in Enabling authorization for HTTP services step 1.

2. Allow access to the details and reviews service in permissive mode with the following command:

    ```javascript
        $ kubectl apply -f samples/bookinfo/platform/kube/rbac/details-reviews-policy-permissive.yaml
    ```
    
3. Verify there are errors Error fetching product details and Error fetching product reviews on the Bookinfo productpage by pointing your browser at the productpage (http://$GATEWAY_URL/productpage), These errors are expected because the policy is in PERMISSIVE mode.

4. Apply the rbac-permissive-telemetry.yaml YAML file to enable the permissive mode metric collection.

    ```javascript
    
    $ kubectl apply -f samples/bookinfo/platform/kube/rbac/rbac-permissive-telemetry.yaml
    
    ```

5. Send traffic to the sample application:

    ```javascript
        $ curl http://$GATEWAY_URL/productpage
    
    ```

6. Get the log for telemetry and search for the permissiveResponseCode with the following command:

    ```javascript
        $ kubectl -n istio-system logs -l istio-mixer-type=telemetry -c mixer | grep \"instance\":\"rbacsamplelog.logentry.istio-system\"
        {"level":"warn","time":"2018-08-30T22:59:42.707093Z","instance":"rbacsamplelog.logentry.istio-system","destination":"details","latency":"423.381µs","permissiveResponseCode":"allowed","permissiveResponsePolicyID":"details-reviews-viewer","responseCode":403,"responseSize":19,"source":"productpage","user":"cluster.local/ns/default/sa/bookinfo-productpage"}
        {"level":"warn","time":"2018-08-30T22:59:42.763423Z","instance":"rbacsamplelog.logentry.istio-system","destination":"reviews","latency":"237.333µs","permissiveResponseCode":"allowed","permissiveResponsePolicyID":"details-reviews-viewer","responseCode":403,"responseSize":19,"source":"productpage","user":"cluster.local/ns/default/sa/bookinfo-productpage"}
    ```
    
7. Verify that the the log shows a responseCode of 403 and a permissiveResponseCode of allowed for ratings and reviews services.

8. Remove the YAML files related to enabling the permissive mode:

    ```javascript
        $ kubectl delete -f samples/bookinfo/platform/kube/rbac/details-reviews-policy-permissive.yaml
        $ kubectl delete -f samples/bookinfo/platform/kube/rbac/rbac-permissive-telemetry.yaml
    ```
    
9. Congratulations! You tested adding an authorization policy with permissive mode and verified it will work as expected. To add the authorization policy, follow the steps described in the Enabling Istio authorization task.


<br><br>
## Istio Vault CA Integration


<br><br>

## Mutual TLS Deep-Dive

이 작업을 통해 상호 TLS(mutual TLS)를 면밀히 관찰하고 설정을 학습 할 수 있습니다. 이 작업에서는 다음을 전제로합니다.

- 인증 정책(authentication policy) 작업을 완료했습니다.
- 인증 정책(authentication policy)을 사용하여 상호 TLS(mutual TLS)를 사용하는 것에 익숙합니다.
- **Istio는 글로벌 상호 TLS(global mutual TLS)가 활성화 된 Kubernetes에서 실행됩니다. 지침에 따라 Istio를 설치할 수 있습니다. 이미 Istio가 설치되어있는 경우 인증 정책(authentication policies) 및 대상 규칙(destination rules)을 추가 또는 수정하여 이 [작업(Task)](https://istio.io/docs/tasks/security/authn-policy/#globally-enabling-istio-mutual-tls)에서 설명한대로 상호 TLS(mutual TLS)를 사용할 수 있습니다.**
- Default namespace에 httpbbin과 Envoy Sidecar 설치된 sleep를 배포 합니다. 예를 들어, [manual sidecar injection](https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/#manual-sidecar-injection)으로 이러한 서비스를 배치하는 명령은 다음과 같습니다.

```javascript

    $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
    $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
    
```

### Verify Citadel runs properly

Citadel은 Istio의 key management 서비스입니다. Citadel는 올바르게 작동하려면 상호 TLS(mutual TLS)를 제대로 실행해야 합니다. 다음 명령을 사용하여 클러스터 수준의 Citadel(cluster-level Citadel)이 올바르게 실행되는지 확인하세요.

```javascript

    $ kubectl get deploy -l istio=citadel -n istio-system
    NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    istio-citadel   1         1         1            1           1m

```

"AVAILABLE" Column이 1 인 경우 Citadel이 작동 중인 상태 입니다.

### Verify keys and certificates installation

Istio는 모든 sidecar containers에 상호 TLS 인증(mutual TLS authentication)을 위해 필요한 키와 인증서를 자동으로 설치합니다. /etc/certs에 키와 인증서 파일이 있는지 확인하려면 아래 명령을 실행하세요.

```javascript

    $ kubectl exec $(kubectl get pod -l app=httpbin -o jsonpath={.items..metadata.name}) -c istio-proxy -- ls /etc/certs
    cert-chain.pem
    key.pem
    root-cert.pem
    
```

> cert-chain.pem은 Envoy의 인증서(cert)로 상대방(the other side)에게 제공해야 합니다. key.pem은 Envoy의 개인 키(private key)이며 Envoy의 cert와 cert-chain.pem이 쌍을 이룹니다. root-cert.pem은 Peer의 인증서(peer’s cert)를 확인하기 위한 Root 인증서입니다. 이 예에서는 클러스터에 Citadel 하나만 있으므로 모든 Envoys에는 동일한 root-cert.pem이 있습니다.

Use the openssl tool to check if certificate is valid (current time should be in between Not Before and Not After)

openssl 도구를 사용하여 인증서가 유효한지 확인하세요 (현재 시간은 Not Before와 Not After 사이에 있어야 함.)

```javascript

    $ kubectl exec $(kubectl get pod -l app=httpbin -o jsonpath={.items..metadata.name}) -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep Validity -A 2
    Validity
            Not Before: May 17 23:02:11 2018 GMT
            Not After : Aug 15 23:02:11 2018 GMT
    
```

> PEM 인코딩된 인증서를 파싱해서 정보를  출력
> $ openssl x509 -text -noout -in localhost.crt


클라이언트 인증서(client certificate)의 ID(identity)를 확인할 수도 있습니다.

```javascript

    $ kubectl exec $(kubectl get pod -l app=httpbin -o jsonpath={.items..metadata.name}) -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep 'Subject Alternative Name' -A 1
            X509v3 Subject Alternative Name:
                URI:spiffe://cluster.local/ns/default/sa/default
    
```

Istio에서 서비스 신원(service identity)에 대한 자세한 정보는 [Istio 신분(Istio identity)](https://istio.io/docs/concepts/security/#istio-identity)을 확인하세요.

### Verify mutual TLS configuration

istioctl 도구를 사용하여 상호 TLS(mutual TLS) 설정이 유효한지 확인하세요. 대상 규칙(destination rule)은 클라이언트 네임 스페이스에 따라 다르므로 istioctl 명령에는 클라이언트의 포드가 필요합니다. 대상 서비스를 제공하여 해당 서비스로만 상태를 필터링 할 수도 있습니다.

다음 명령은 httpbin.default.svc.cluster.local 서비스의 인증 정책(authentication policy)을 식별하고 sleep App의 동일한 Pod에서 본 서비스의 대상 규칙(destination rules)을 식별합니다.

```javascript

    $ SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
    $ istioctl authn tls-check ${SLEEP_POD} httpbin.default.svc.cluster.local
    
```

다음 예제 출력에서 볼 수 있습니다 :

- 상호 TLS(Mutual TLS)는 포트 8000의 httpbin.default.svc.cluster.local에 대해 일관되게 설정됩니다.
- Istio는 메쉬 전체 기본 인증 정책(mesh-wide default authentication policy)을 사용합니다.
- Istio는 istio-system 네임 스페이스에 기본 대상 규칙(default destination rule)을가집니다.
 
```javascript

    HOST:PORT                                  STATUS     SERVER     CLIENT     AUTHN POLICY        DESTINATION RULE
    httpbin.default.svc.cluster.local:8000     OK         mTLS       mTLS       default/            default/istio-system
    
```

The output shows:

- STATUS: 이경우에는 httpbin service 인 Server와 Client 또는 httpbin를 호출하는 client간의 TLS Setting 일관되는지 여부 
- SERVER: 서버에서 사용되는 모드
- CLIENT: 클라이언트에서 사용되는 모드.
- AUTHN POLICY: the name and namespace of the authentication policy. If the policy is the mesh-wide policy, namespace is blank, as in this case: default/
- DESTINATION RULE: the name and namespace of the destination rule used.

충돌(conflicts)이 있는 경우를 설명하기 위해 잘못된 TLS 모드로 httpbin에 대한 서비스 별 대상 규칙(service-specific destination)을 추가하세요.

```javascript

    $ cat <<EOF | kubectl apply -f -
    apiVersion: "networking.istio.io/v1alpha3"
    kind: "DestinationRule"
    metadata:
      name: "bad-rule"
      namespace: "default"
    spec:
      host: "httpbin.default.svc.cluster.local"
      trafficPolicy:
        tls:
          mode: DISABLE
    EOF
    
```

위와 같은 istioctl 명령을 실행하면 서버가 mTLS에 있는 동안 클라이언트가 HTTP 모드에 있으므로 CONFLICT 상태가 표시됩니다.

```javascript

    $ i authn tls-check ${SLEEP_POD} httpbin.default.svc.cluster.local
    HOST:PORT                                  STATUS       SERVER     CLIENT     AUTHN POLICY        DESTINATION RULE
    httpbin.default.svc.cluster.local:8000     CONFLICT     mTLS       HTTP       default/            bad-rule/default
    
```

You can also confirm that requests from sleep to httpbin are now failing:
sleep에서 httpbin으로의 요청이 실패했음을 확인할 수도 있습니다 :

```javascript

    $ kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl httpbin:8000/headers -o /dev/null -s -w '%{http_code}\n'
    503
    
```

계속하기 전에 잘못된 대상 규칙(destination)을 제거하여 다음 명령을 사용하여 상호 TLS(mutual TLS)를 다시 작동 시키세요.

```javascript

    $ kubectl delete destinationrule --ignore-not-found=true bad-rule

```

### Verify requests

이 작업은 상호 TLS(mutual TLS)를 사용하는 서버가 다음과 같은 요청에 대한 응답을 활성화하는 방법을 보여줍니다.

- In plain-text
- With TLS but without client certificate
- With TLS with a client certificate

이 작업을 수행하려면 client proxy를 by-pass해야합니다. 가장 간단한 방법은 istio-proxy 컨테이너에서 요청을 보내는 것입니다.


1. 다음 명령을 사용하여 httpbin과 통신하려면 TLS가 필요하므로 일반 텍스트(plain-text) 요청이 실패했는지 확인하세요.

    ```javascript
    
        $ kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c istio-proxy -- curl http://httpbin:8000/headers -o /dev/null -s -w '%{http_code}\n'
        000
        command terminated with exit code 56
        
    ```
    
    > 종료 코드는 56입니다.이 코드는 네트워크 데이터를 수신하지 못하는 것으로 해석됩니다.
    
    > [curl 응답코드](https://ec.haxx.se/usingcurl-returns.html)
    > 네트워크 데이터 수신 실패. 네트워크를 통해 데이터를 수신하는 것은 대부분의 curl 조작에서 중요한 부분이며 curl이 가장 낮은 네트워킹 계층에서 데이터 수신에 실패한 오류가 발생하면이 종료 상태가 리턴됩니다. 왜 이런 일이 발생했는지 정확하게 알기 위해 보통 심각한 파기가 필요합니다. 자세한 모드를 활성화하고, 추적하고 가능하다면 Wireshark 또는 이와 유사한 도구를 사용하여 네트워크 트래픽을 확인하세요.

2. 클라이언트 인증서가없는 TLS 요청도 실패하는지 확인합니다.

    ```javascript

        $ kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c istio-proxy -- curl https://httpbin:8000/headers -o /dev/null -s -w '%{http_code}\n' -k
        000
        command terminated with exit code 35
    
    ```

    > 이번에는 종료 코드가 35이며 SSL/TLS handshake 어딘가에서 발생하는 문제에 해당합니다.

    > [curl 응답코드](https://ec.haxx.se/usingcurl-returns.html)
    > TLS / SSL 연결 오류입니다. SSL 핸드 셰이크가 실패했습니다. 여러 가지 이유로 SSL 핸드 셰이크가 실패 할 수 있으므로 오류 메시지가 몇 가지 추가적인 단서를 제공 할 수 있습니다. 당사자들이 SSL / TLS 버전, 쾌적한 암호 모음 또는 이와 유사한 것에 동의하지 않았을 수 있습니다.

3. Confirm TLS request with client certificate succeed:

    ```javascript
    
        $ kubectl exec $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c istio-proxy -- curl https://httpbin:8000/headers -o /dev/null -s -w '%{http_code}\n' --key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k
        200
    
    ```

   > Istio는 서비스 이름보다 Kubernetes 서비스 계정(service accounts)을 사용하여 서비스 이름(service name)보다 강력한 보안을 제공합니다 (자세한 내용은 Istio ID 참조). 따라서 Istio에서 사용하는 인증서에는 서비스 이름이 없습니다.이 서비스 이름은 curl이 서버 ID(server identity)를 확인하는 데 필요한 정보입니다. curl 클라이언트가 중단되지 않도록 curl을 -k 옵션과 함께 사용합니다. 이 옵션을 사용하면 클라이언트가 서버에서 제공하는 인증서의 서버 이름 (예 : httpbin.default.svc.cluster.local)을 확인 및 찾지 못합니다.

### Cleanup

```javascript

$ kubectl delete --ignore-not-found=true -f samples/httpbin/httpbin.yaml
$ kubectl delete --ignore-not-found=true -f samples/sleep/sleep.yaml
```

<br><br>

## Plugging in External CA Key and Certificate

<br><br>

## [Citadel Health Checking](https://istio.io/docs/tasks/security/health-check/)

<br><br>

## Provisioning Identity through SDS

<br><br>

## Mutual TLS Migration

<br><br>

## Mutual TLS over HTTPS
