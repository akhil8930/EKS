# Lab 3: Integrating Persistent Storage with EKS Using Dynamic Provisioning

## Objective
Learn how to use dynamic persistent storage with Kubernetes on Amazon EKS by leveraging Amazon Elastic Block Store (EBS).

## Prerequisites
- A running EKS cluster.
- `kubectl` configured to communicate with your cluster.
- Basic understanding of Kubernetes objects like Pods, Deployments, and Services.

## Detailed Steps

### Step 1: Create a Kubernetes Storage Class

1. Create a file named `ebs-sc.yaml` with the following content:
   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: ebs-sc
   provisioner: kubernetes.io/aws-ebs
   parameters:
     type: gp2
     fsType: ext4
   reclaimPolicy: Retain
   allowVolumeExpansion: true
   ```

2. Apply the storage class:
   ```bash
   kubectl apply -f ebs-sc.yaml
   ```

### Step 2: Create a Persistent Volume Claim (PVC)

1. Create a file named `ebs-pvc.yaml` with the following content:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: ebs-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: ebs-sc
     resources:
       requests:
         storage: 4Gi
   ```

2. Apply the PVC:
   ```bash
   kubectl apply -f ebs-pvc.yaml
   ```

### Step 3: Attach the PVC to a Deployment

1. Create a deployment that uses the PVC. Here’s an example deployment using an NGINX image which mounts the volume at `/usr/share/nginx/html`.
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
   spec:
     selector:
       matchLabels:
         app: nginx
     replicas: 1
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:latest
           ports:
           - containerPort: 80
           volumeMounts:
           - name: ebs-storage
             mountPath: /usr/share/nginx/html
         volumes:
         - name: ebs-storage
           persistentVolumeClaim:
             claimName: ebs-pvc
   ```

2. Apply the deployment:
   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

### Step 4: Verify the Deployment and PVC

1. Check the status of the PVC:
   ```bash
   kubectl get pvc
   ```
   Ensure the status is `Bound` and that it is dynamically provisioning an EBS volume.

2. Check the deployment:
   ```bash
   kubectl get deployments
   ```
   Ensure the deployment is successfully rolled out.

### Step 5: Expose the NGINX Deployment as a Service

1. Create a YAML file named `nginx-service.yaml` with the following content:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 80
       protocol: TCP
     selector:
       app: nginx
   ```

2. Apply the Service Configuration:
   ```bash
   kubectl apply -f nginx-service.yaml
   ```

3. Retrieve the External IP Address:
   ```bash
   kubectl get svc nginx-service
   ```
   Look for the `EXTERNAL-IP` column in the output. This IP address is the public endpoint for your NGINX server.

4. Access the NGINX Server:
   Once the `EXTERNAL-IP` is available, you can access the NGINX server by entering the IP address in a web browser. You should see the default NGINX welcome page, which confirms that the server is running and accessible.

## Additional Considerations
- **Security Groups**: Ensure that the security groups associated with your EKS nodes and the load balancer allow traffic on port 80. You may need to adjust the inbound rules to permit this traffic.
- **DNS and Custom Domains**: For a production environment, instead of using the raw IP address, you might want to configure a domain name to point to the load balancer. This can be managed through Route 53 or another DNS service.

By completing this lab, you will have successfully integrated dynamic persistent storage into your Kubernetes application using Amazon EBS. This setup is crucial for stateful applications that require data persistence across pod recreations and node failures.