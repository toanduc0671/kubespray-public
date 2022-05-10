# Các bước cài đặt k8s bằng kubespray & cài đặt kubesphere lên k8s

## 1. Chuẩn bị

- Khởi tạo 4 VM trên google cloud chung một subnet có cấu hình như sau:

  - VM làm bastion có ip: 10.240.0.11
  - 3 VM làm k8s cluster có ip lần lượt là: 10.240.0.2 (master), 10.240.0.3 (worker) , 10.240.0.4 (worker)

- Thêm ssh key trên các VM làm k8s node để từ `bastion` có thể truy cập được.

## 2. Cài đặt k8s cluster bằng kubespray

> Tất cả công việc dưới đây đều được thực hiện trên con `bastion`

- Cài đặt pip3 (đã có python3)

  ```code
  sudo apt-get install python3-pip
  ```

- Clone repo `kubespray`

  ```code
  git clone https://github.com/kubernetes-sigs/kubespray.git
  ```

- Chạy các script dưới đây:

  ```ShellSession
  # access folder kubespray
  cd kubespray/
  # Install dependencies from ``requirements.txt``
  sudo pip3 install -r requirements.txt

  # Copy thư mục ``inventory/sample`` ra thư mục ``inventory/mycluster``
  cp -rfp inventory/sample inventory/mycluster

  # Liệt kê danh sách các IP của các VM dự định chạy k8s
  declare -a IPS=(10.240.0.2 10.240.0.4 10.240.0.4)

  # chạy lệnh python này để sinh file
  CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

  ```

- Sửa đổi file `hosts.yaml` vừa được sinh ở trên (chuyển 2 master node --> 1 master node)

  ```console
  all:
  hosts:
      node1:
      ansible_host: 10.240.0.2
      ip: 10.240.0.2
      access_ip: 10.240.0.2
      node2:
      ansible_host: 10.240.0.3
      ip: 10.240.0.3
      access_ip: 10.240.0.3
      node3:
      ansible_host: 10.240.0.4
      ip: 10.240.0.4
      access_ip: 10.240.0.4
  children:
      kube_control_plane:
      hosts:
          node1:
      kube_node:
      hosts:
          node1:
          node2:
          node3:
      etcd:
      hosts:
          node1:
      k8s_cluster:
      children:
          kube_control_plane:
          kube_node:
      calico_rr:
      hosts: {}
  ```

- Sửa đổi version k8s sẽ cài đặt bằng cách sửa file `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`:

  > Tìm đến dòng có chứa tham số `kube_version` chỉnh sửa thành `kube_version: v1.20.3`

- Chạy ansible-playbook để khởi tạo k8s cluster:

  ```console
  ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
  ```

- chờ đợi playbook cài đặt thành công (khoảng 10-20 phút)

- cấu hình kubectl để kết nối với cluster vừa khởi tạo

  - cài đặt kubectl

    ```console
    curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```

  - ssh vào máy master node

    ```console
    ssh 10.240.0.2
    ```

  - copy nội dung file config

    ```console
    cat /etc/kubernetes/admin.conf
    ```

  - thoát ssh máy master node trở về máy bastion, sửa đổi file `$HOME/.kube/config` có nội dung copy ở trên. Trong đó sửa đổi tham số `server` trong file thành: `server: https://10.240.0.2:6443`

## 3. Cài đặt kubesphere lên k8s cluster

### 3.1. Tạo nfs server trên bastion

- Cài đặt

  ```ShellSession
    # update system
    sudo apt update
    # install package
    sudo apt install nfs-kernel-server
    # tạo foler share và cấp quyền
    sudo mkdir -p /mnt/nfs_share
    sudo chown -R nobody:nogroup /mnt/nfs_share/
    sudo chmod 777 /mnt/nfs_share/
  ```

- sửa đổi file `/etc/exports` có nội dung như sau:

  > k8s node đều thuộc subnet `10.240.0.0/24`

  ```console
  /mnt/nfs_share  10.240.0.0/24(rw,sync,no_subtree_check)
  ```

### 3.2. Cài đặt csi-driver lên k8s cluster

> ssh vào master node để thực hiện các câu lệnh dưới đây:

```console
git clone https://github.com/kubernetes-csi/csi-driver-nfs.git
cd csi-driver-nfs
./deploy/install-driver.sh master local
```

### 3.3. Cài đặt storage class mặc định

- Tạo storage class sử dụng nfs server đã tạo ở trên

  > Dùng `kubectl apply -f` file nội dung như sau:

  ```console
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: nfs-csi
  provisioner: nfs.csi.k8s.io
  parameters:
    server: 10.240.0.11
    share: /mnt/nfs_share
  reclaimPolicy: Delete
  volumeBindingMode: Immediate
  mountOptions:
    - hard
    - nfsvers=4.1
  ```

- Biến storage class `nfs-csi` trên thành `default` để phục vụ `Dynamic Volume Provisioning`

  ```console
  kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
  ```

### 3.4. Cài đặt kubesphere:

- Cài đặt kubesphere

  ```console
  kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.1.1/kubesphere-installer.yaml

  kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.1.1/cluster-configuration.yaml

  ```

- Xem logs quá trình cài đặt

  ```console
  kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
  ```

- Quá trình cài đặt thành công sẽ tạo ra một service dạng `NodePort` cổng 30880. Truy cập vào trang đó với địa chỉ: `http://{ip-của-node-k8s-bất-kỳ}:30880` với tài khoản, mật khẩu admin/P@88w0rd
