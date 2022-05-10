# Ghi chú những kiến thức quan trọng & Lab

## I. Kiến thức cần chú ý

- Kiến thức về phần `controller` của các resources ở khóa `The Kubernetes Machine` (các controller sẽ interval gọi api đến `api server` rồi cập nhật trạng thái chứ không tương tác trực tiếp giữa các container với nhau).

- Kiến thức về module `scheduled` trong khóa `The Kubernetes Machine` giúp hiểu được cách k8s gán pod cho node và hiểu được tại sao mặc định node master ko thể chạy được pod.

- Module `etcd` là rất quan trọng, nó là database của hệ thống, chỉ cần backup và đảm bảo sự toàn vẹn của `etcd` là hệ thống sẽ ổn định (etcd lưu thông tin namespace, pod, network, crds, sa, ...)

- `kubelet` chạy trên tất cả các node và không chạy bằng `container`. Nó là service interval gọi `api server` để tìm pod, pv gán cho node của nó.

- `pod network` gồm một số kiến thức cần chú ý:

  - Mỗi một cluster cần một CNI để có trách nhiệm tạo network cho pod, connect các pod được với nhau(CNI được phát triển chung cho các trình container)

  - Thiết lập các rule `Network Policy` để tăng tính bảo mật cho pod. Gồm 2 rule là `ingress` đi vào và `engress`đi ra.

- `service` Khi một request gửi đến service thì đường đi sẽ như sau LoadBalancer > NodePort > ClusterIP. Đến clusterip nó sẽ đọc `endpoints` resources và trả về podIP.

- `ingress` nó giống như một service bình thường, có tác dụng forward request theo ý muốn.

- `readiness` & `liveness` khá hữu ích trong việc kiểm tra tính sẵn sàng và tính "sống chết" của pod.

- `heml` thực chất là các file yaml có tác dụng triển khai một dịch vụ thôi, còn các logic mang tính "magic" đều do images build trong nó thực hiện.

- trong khi sử dụng k8s, sẽ dùng rất nhiều `CRDs`, thực chất nó gồm 2 thành phần: `custom resources` (tương tự như deployments, service, replicaset,..) và `custom controller` (tương tự như deployment controller, replicaset controller, ...). Các `CRDs` này sẽ được cung cấp `service account` với các `role` cần thiết để duy trì tính đúng đắn của các resources nó cần tác động. Việc cung cấp service account sẽ qua một tham số `serviceAccountName` trong template của pod.

## II. Một số bài Lab

> Các bài lab đều được thực hiện trên cluster gồm: 1 master node, 2 worker node

```console
NAME         STATUS   ROLES                  AGE   VERSION
controller   Ready    control-plane,master   20d   v1.22.0
worker-1     Ready    <none>                 20d   v1.22.0
worker-2     Ready    <none>                 20d   v1.22.0

```

### 1. PV/PVC

- sử dụng `hostpath`

  ```console
  apiVersion: v1
  kind: PersistentVolume
  metadata:
  name: pv1
  labels:
      name: pv1
  spec:
  storageClassName: mystorageclass
  capacity:
      storage: 5Gi
  accessModes:
      - ReadWriteOnce
  hostPath:
      path: "/v1"
  ```

  ```console
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc1
  labels:
      name: pvc1
  spec:
  storageClassName: mystorageclass
  accessModes:
      - ReadWriteOnce
  resources:
      requests:
      storage: 150Mi
  ```

  ```console
  apiVersion: apps/v1
  kind: Deployment
  metadata:
  name: myapp
  spec:
  replicas: 3
  selector:
      matchLabels:
      name: myapp
  template:
      metadata:
      name: myapp
      labels:
          name: myapp
      spec:
      volumes:
      # Khai báo VL sử dụng PVC
      - name: myvolume
          persistentVolumeClaim:
            claimName: pvc1
      containers:
      - name: myapp
          image: nginx
          resources:
          limits:
              memory: "300Mi"
              cpu: "300m"
          volumeMounts:
          - mountPath: "/data"
          name: myvolume
  ```

  ==> pod chạy ở node nào thì nó sẽ sử dụng pv ở node đó
  ==> multi node sử dụng được pv dạng hostpath

- sử dụng `local`

  ```console
  apiVersion: v1
  kind: PersistentVolume
  metadata:
  name: example-pv-local
  spec:
  capacity:
      storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
      path: /mnt/disks/ssd1
  nodeAffinity:
      required:
      nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-1
  ```

  ```console
  apiVersion: v1
  kind: PersistentVolume
  metadata:
  name: example-pv-local
  spec:
  capacity:
      storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
      path: /mnt/disks/ssd1
  nodeAffinity:
      required:
      nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-1
  root@controller:/pv-pvc# cat pvc-local.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
  name: pvc-local
  labels:
      name: pvc1
  spec:
  storageClassName: local-storage
  accessModes:
      - ReadWriteOnce
  resources:
      requests:
              storage: 8Gi

  ```

  ```console
  apiVersion: apps/v1
  kind: Deployment
  metadata:
  name: myapp
  spec:
  replicas: 3
  selector:
      matchLabels:
      name: myapp
  template:
      metadata:
      name: myapp
      labels:
          name: myapp
      spec:
      volumes:
      # Khai báo VL sử dụng PVC
      - name: myvolume
          persistentVolumeClaim:
          claimName: pvc-local
      containers:
      - name: myapp
          image: nginx
          resources:
          limits:
              memory: "200Mi"
              cpu: "200m"
          volumeMounts:
          - mountPath: "/data"
          name: myvolume

  ```

===> Khởi tạo pv dạng local ở node worker1, sau đó tạo pvc yêu cầu đến pv đó. => Các pod sử dụng pvc trên sẽ chỉ được chạy trên worker1

### 2. Service type

- triển khai các loại service

- phân biệt clusterip, nodeport, loadbalancer

  - một cái ví dụ đơn giản nhất khi tìm hiểu 3 loại service này đó là: khi một request đến `loadbalancer` nó sẽ load balance rồi chọn ra một node từ đó gọi `nodeip:nodeport`, tại node đó dựa vào `kube-proxy` nó sẽ tìm ra `cluster-ip`. Tại cluster-ip một lần nữa nó sẽ load balance để chọn ra một pod đáp ứng request đó.

  ```console
  request ==> loadbalancer ==> nodeport ==> clusterip ==> podip
  ```

### 3. Ingress

- Cài đặt một ingress controller

  ```console
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/baremetal/deploy.yaml
  ```

- Triển khai một service

  ```console
  apiVersion: v1
  kind: Service
  metadata:
  name: http-test-svc
  namespace: ingress-nginx
  spec:
  ports:
  - port: 80
      protocol: TCP
      targetPort: 80
  selector:
      run: http-test-app
  sessionAffinity: None
  type: ClusterIP
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
  labels:
      run: http-test-svc
  name: http-test-svc
  namespace: ingress-nginx
  spec:
  replicas: 2
  selector:
      matchLabels:
      run: http-test-app
  template:
      metadata:
      labels:
          run: http-test-app
      spec:
      containers:
      - image: httpd
          imagePullPolicy: IfNotPresent
          name: http
          ports:
          - containerPort: 80
          protocol: TCP
          resources: {}
  ```

- Triển khai một ingress

  ```console
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
  name: app
  namespace: ingress-nginx
  spec:
  rules:
      # Tên miền truy cập
  - host: abc.test
      http:
      paths:
      - path: "/"
          pathType: Prefix
          backend:
          service:
              name: http-test-svc
              port:
              number: 80
  ```

  ===> Kết quả: khi gọi `http://abc.test` ta sẽ được kết quả trả về từ service k8s có tên `http-test-svc` (để gọi được domain `abc.test` ta định nghĩa thêm trong file hosts tùy os)

### 4. tail node

- Muốn pod chạy trên một node đã bị taint

  ```console
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
  name: dsapp
  spec:
  selector:
      matchLabels:
      app: ds-nginx
  template:
      metadata:
      labels:
          app: ds-nginx
      spec:
      containers:
      - name: nginx
          image: nginx
          resources:
          limits:
              memory: "128Mi"
              cpu: "100m"
          ports:
          - containerPort: 80
      tolerations:
      - effect: "NoSchedule"
          operator: "Exists"
          key: "node-role.kubernetes.io/master"
  ```

==> Deamonset này sẽ giúp pod chạy trên cả node master

- Muốn taint một node:

  ```console
  kubectl taint nodes node1 key1=value1:NoSchedule
  ```

- Bỏ taint:

  ```console
  kubectl taint nodes node1 key1=value1:NoSchedule-
  ```

### 5. networkpolicy

- Tạo một manifest network policy

  ```console
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
  name: test-np-1
  namespace: default
  spec:
  podSelector:
      matchLabels:
        app: box
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - to:
      - ipBlock:
          cidr: 34.126.115.145/32
      ports:
      - protocol: TCP
      port: 31902

  ```

  > ở đây sẽ áp dụng network policy cho các pod có label `app: box`, tất cả các pod này chỉ được phép gọi ra ngoài địa chỉ `34.126.115.145` ở port `31902`. (kiểm chứng bằng cách `exec` vào pod rồi kiểm tra bằng lệnh `curl` và `ping`)

### 6. etcd

- danh sách etcd

  ```console
  root@controller:~# k get po -n kube-system -o wide | grep etcd
  etcd-controller                            1/1     Running   1          20d   10.240.0.11      controller   <none>           <none>

  ```

  > chỉ có 1 pod etcd đang chạy tại ip `10.240.0.11`

- truy cập vào etcd

  ```console
  k exec -it -n kube-system etcd-controller /bin/sh
  ```

  ```console
  ENDPOINTS="https://10.240.0.11:2379"
  TLS="--cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt"
  alias etcdctl="etcdctl $TLS --endpoints $ENDPOINTS"
  ```

- truy vấn etcd

  - lấy danh sách namespaces

  ```console
  etcdctl get /registry/namespaces --prefix --keys-only
  ```

  - thêm dữ liệu vào etcd

  ```console
  etcdctl put key value
  ```

  - xóa dữ liệu

  ```console
  etcdctl del key
  ```

  > Mọi cấu hình resources của cluster đều lưu ở etcd ==> tầm quan trọng của việc lưu trữ, backup etcd.

### 7. liveness, readiness

- triển khai một replica

  ```code
  kind: List
  apiVersion: v1
  items:
  - kind: ReplicationController
  apiVersion: v1
  metadata:
      name: frontend
      labels:
      name: frontend
  spec:
      replicas: 1
      selector:
      name: frontend
      template:
      metadata:
          labels:
          name: frontend
      spec:
          containers:
          - name: frontend
          image: katacoda/docker-http-server:health
          readinessProbe:
              httpGet:
              path: /
              port: 80
              initialDelaySeconds: 1
              timeoutSeconds: 1
          livenessProbe:
              httpGet:
              path: /
              port: 80
              initialDelaySeconds: 1
              timeoutSeconds: 1
  - kind: ReplicationController
  apiVersion: v1
  metadata:
      name: bad-frontend
      labels:
      name: bad-frontend
  spec:
      replicas: 1
      selector:
      name: bad-frontend
      template:
      metadata:
          labels:
          name: bad-frontend
      spec:
          containers:
          - name: bad-frontend
          image: katacoda/docker-http-server:unhealthy
          readinessProbe:
              httpGet:
              path: /
              port: 80
              initialDelaySeconds: 1
              timeoutSeconds: 1
          livenessProbe:
              httpGet:
              path: /
              port: 80
              initialDelaySeconds: 1
              timeoutSeconds: 1
  ```

- Kết quả:

  - Kiểm tra bằng câu lệnh `kubectl get pod` các pod có label `frontend` đều khởi động và chạy thành công do pass qua được 2 quá trình `liveness` và `readiness`. Trong khi đó các pods có label `bad-frontend` khởi chạy thất bại do quá trình `liveness` sử dụng http request nhưng trả về mã 500.
  - pod nào không pass được `readiness` thì coi như pod đó không sẵn sàng => ko được sử dụng (ví dụ như service sẽ xóa pod đó ra khỏi danh sách các endpoint)
  - pod nào không pass được `liveness` thì pod đó sẽ không khởi chạy thành công => có thể restart tùy theo `restartPolicy`

- Mở rộng:

  - Ngoài quá trình `readiness` và `liveness` thì còn có quá trình `startupProbe` để kiểm tra tính khởi tạo của pod. Nếu pass qua giai đoạn `startupProbe` thì mới đến giai đoạn kiểm tra 2 service trên.

  - Có nhiều loại kiểm tra: http request dựa vào mã response, tcp socket dựa vào kết nối, command line dựa vào mã trả về.

  - More: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
