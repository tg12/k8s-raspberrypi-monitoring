# k8s-raspberrypi-monitoring
k8s-raspberrypi-monitoring

# Setting Up Grafana, InfluxDB, and Kubernetes Dashboard on Raspberry Pi Kubernetes Cluster

This guide will help you set up Grafana, InfluxDB, and the Kubernetes Dashboard on your Raspberry Pi Kubernetes cluster with persistent storage.

## Prerequisites

- A Raspberry Pi with Raspbian OS installed.
- A stable internet connection.

## Step 1: Set Up the Raspberry Pi

1. **Update and upgrade the system:**
    ```sh
    sudo apt-get update
    sudo apt-get upgrade -y
    ```

2. **Install Docker:**
    ```sh
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker pi
    ```
    Log out and log back in to apply the Docker group membership.

3. **Install kubeadm, kubelet, and kubectl:**
    ```sh
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

4. **Disable swap:**
    ```sh
    sudo dphys-swapfile swapoff
    sudo dphys-swapfile uninstall
    sudo systemctl disable dphys-swapfile
    ```

5. **Initialize the Kubernetes cluster:**
    ```sh
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    ```

6. **Set up local kubeconfig:**
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

7. **Install a network plugin (Flannel):**
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```

8. **Untaint master node (optional, if you plan to run pods on the master node):**
    ```sh
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

## Step 2: Deploy InfluxDB and Grafana with Persistent Storage

1. **Create Persistent Volumes for InfluxDB and Grafana:**

    ```yaml
    # pv-influxdb.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: influxdb-pv
    spec:
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /mnt/data/influxdb
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: influxdb-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
    ```

    ```yaml
    # pv-grafana.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: grafana-pv
    spec:
      capacity:
        storage: 5Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /mnt/data/grafana
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: grafana-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
    ```

    Apply the PV and PVC:
    ```sh
    kubectl apply -f pv-influxdb.yaml
    kubectl apply -f pv-grafana.yaml
    ```

2. **Deploy InfluxDB:**

    ```yaml
    # influxdb-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: influxdb
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: influxdb
      template:
        metadata:
          labels:
            app: influxdb
        spec:
          containers:
          - name: influxdb
            image: influxdb:latest
            ports:
            - containerPort: 8086
            volumeMounts:
            - name: influxdb-storage
              mountPath: /var/lib/influxdb
          volumes:
          - name: influxdb-storage
            persistentVolumeClaim:
              claimName: influxdb-pvc
    ```

    Apply the deployment:
    ```sh
    kubectl apply -f influxdb-deployment.yaml
    ```

3. **Deploy Grafana:**

    ```yaml
    # grafana-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: grafana
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: grafana
      template:
        metadata:
          labels:
            app: grafana
        spec:
          containers:
          - name: grafana
            image: grafana/grafana:latest
            ports:
            - containerPort: 3000
            volumeMounts:
            - name: grafana-storage
              mountPath: /var/lib/grafana
          volumes:
          - name: grafana-storage
            persistentVolumeClaim:
              claimName: grafana-pvc
    ```

    Apply the deployment:
    ```sh
    kubectl apply -f grafana-deployment.yaml
    ```

4. **Expose the services:**

    ```yaml
    # influxdb-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: influxdb
    spec:
      ports:
      - port: 8086
      selector:
        app: influxdb
    ```

    ```yaml
    # grafana-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: grafana
    spec:
      ports:
      - port: 3000
      selector:
        app: grafana
    ```

    Apply the services:
    ```sh
    kubectl apply -f influxdb-service.yaml
    kubectl apply -f grafana-service.yaml
    ```

## Step 3: Install the Kubernetes Dashboard

1. **Deploy the Kubernetes Dashboard:**

    ```sh
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
    ```

2. **Create a service account and cluster role binding:**

    ```yaml
    # dashboard-adminuser.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    ```

    Apply the configuration:
    ```sh
    kubectl apply -f dashboard-adminuser.yaml
    ```

3. **Obtain the access token:**

    ```sh
    kubectl -n kubernetes-dashboard create token admin-user
    ```

4. **Access the Kubernetes Dashboard:**

    Start a proxy:
    ```sh
    kubectl proxy
    ```

    Access the dashboard at the following URL:
    ```
    http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
    ```

    Use the token obtained in the previous step to log in.

## Conclusion

You have successfully set up InfluxDB, Grafana, and the Kubernetes Dashboard on your Raspberry Pi Kubernetes cluster with persistent storage. For further customization and management, refer to the official documentation of these tools.
