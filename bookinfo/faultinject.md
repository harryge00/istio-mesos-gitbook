
## http注入
可以注入http返回码为500，当登录为jason时，看不到reviews：
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
    fault:
      abort:
        percent: 100
        httpStatus: 500
    route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v1
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v1
EOF
```

恢复：
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
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v1
EOF
```

## 延迟注入
可以人为注入7s的延迟，使访问rating的请求超时。首先，令jason用户看到ratings v2：
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
此时再次登录为jason，可以看到红星星
注入错误：
```
cat << EOF| kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings.marathon.slave.mesos
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percent: 100
        fixedDelay: 7s
    route:
    - destination:
        host: ratings.marathon.slave.mesos
        subset: v1
  - route:
    - destination:
        host: ratings.marathon.slave.mesos
        subset: v1
EOF
```
再次刷新，页面会卡住很久，并且异常

以上错误，均可以在jaeger中查看到`error`
