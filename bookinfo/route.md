通过`virtualservice` 可以实现根据版本、http头等信息进行路由控制

## 版本控制
执行如下命令，再刷新浏览器访问 `/productpage`，可以看到`reviews`变为v1，即没有星星
```
cat << EOF| kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: productpage.marathon.slave.mesos
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: ratings.marathon.slave.mesos
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: details.marathon.slave.mesos
        subset: v1
---
EOF
```

## 根据登录用户路由
可以根据http头部的登录用户id，进行路由。
```
cat << EOF| kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews.marathon.slave.mesos
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v3
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v1
EOF
```
之后如果登录为jason, 则可以看到红星，logout，则又能看到没有星星
