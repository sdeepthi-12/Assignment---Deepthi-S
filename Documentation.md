Prerequisites

-> Install kubectl on Windows 
    https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-on-windows:~:text=avoid%20unforeseen%20issues.-,Install%20kubectl%20on%20Windows,-The%20following%20methods

    Run this command to verify: kubectl version

-> Install DockerDesktop
    https://docs.docker.com/desktop/setup/install/windows-install/

-> Install Minikube
    https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download

    Run this command to verify: minikube version

-> Install helm
    https://github.com/helm/helm/releases

    Run this command to verify: helm version

Note: Add the path to environment variables (minikube, helm, minikube)

Step 1: Connect to the Kubernetes Cluster

-> Open/Run Docker Desktop
-> Run the minikube cluster
    >> minikube start
-> Verify Minikube is running:
    >> minikube status
-> Check connectivity to the cluster:
    >> kubectl cluster-info

Step 2: Install Helm and KEDA

-> Initialize helm
    >>helm repo add bitnami https://charts.bitnami.com/bitnami

-> Install KEDA Using Helm
    >>helm repo add kedacore https://kedacore.github.io/charts
    >>helm repo update

-> Install KEDA into the cluster:
    >> helm install keda kedacore/keda --namespace keda --create-namespace

-> Verify KEDA Installation
    >> kubectl get pods -n keda

-> Check the custom resource definitions (CRDs) installed by KEDA:
    >> kubectl get crds


Step 3: Create Deployment

-> Public image and tag from DockerHub: nginx:1.23.0
-> Define a Namespace: nginx-namespace.yaml
    >>kubectl apply -f nginx-namespace.yaml
-> Define the Deployment: nginx-deployment.yaml
    >> kubectl apply -f nginx-deployment.yaml
    >> kubectl get pods -n nginx-namespace   #verify the Service
-> Define the Service: nginx-service.yaml
    >> kubectl apply -f nginx-service.yaml
    >> kubectl get svc -n nginx-namespace #verify the deployment
-> Add KEDA Autoscaling: nginx-scaledobject.yaml (used a CPU trigger as metric)
    >> kubectl apply -f nginx-scaledobject.yaml
    >> kubectl get scaledobject -n nginx-namespace #Verify the ScaledObject
-> Deployment Details
    >> kubectl describe deployment nginx-deployment -n nginx-namespace
    >> kubectl get hpa -n nginx-namespace  # check replica count in Autoscaling


To Test the Autoscaling:

-> We can use Stress tool to simulate high CPU usage
-> Pull a Stress Testing Image from DockerHub
    >> docker pull polinux/stress
-> Deploy a a separate pod that runs the stress workload : stress-pod.yaml
-> Add a Sidecar Container to NGINX Deployment - Uses one CPU core (--cpu 1) for 60 seconds to generate load.
    >> - name: stress
          image: polinux/stress
          command: ["stress"]
          args: ["--cpu", "1", "--timeout", "60s"] # Adjust CPU and duration
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"

-> Apply the Stress Pod:
    >> kubectl apply -f stress-pod.yaml
    >> kubectl apply -f nginx-deployment.yaml
-> Observer the autoscaling behavior:
    >> kubectl get pods -n nginx-namespace --watch   # Watch Pod Scaling
    >> kubectl describe scaledobject nginx-scaledobject -n nginx-namespace # View ScaledObject Status


Step 4: Health Status Retrieval

-> Check health status of the deployment, pods, metrics, and report any issues or failures in the deployment
-> Example output:

    Health Status:
    {
        'deployment': 
            {
                'replicas': 4, 
                'available_replicas': 2, 
                'ready': False, 
                'issues': 
                    [
                        'Mismatch: 2/4 replicas available.', 
                        'Deployment is not ready.'
                    ]
            }, 
        'pods': 
            [
                {
                    'name': 'nginx-deployment-559d76fc7b-f4ljc', 
                    'status': 'Running', 
                    'reason': ''
                }, 
                {
                    'name': 'nginx-deployment-559d76fc7b-mqcpc', 
                    'status': 'Running', 
                    'reason': ''
                }, 
                {
                    'name': 'nginx-deployment-5c96bb98f8-rpmtr', 
                    'status': 'Pending', 
                    'reason': 'ImagePullBackOff'
                }, 
                {
                    'name': 'nginx-deployment-5c96bb98f8-t49jj', 
                    'status': 'Pending', 
                    'reason': 'ImagePullBackOff'
                }
            ], 

        'metrics': 'Metrics API unavailable', 
        'issues': 
            [
                'Mismatch: 2/4 replicas available.', 
                'Deployment is not ready.', 
                "Pod nginx-deployment-5c96bb98f8-rpmtr is in 'Pending' state. Reason: ImagePullBackOff"
            ]
    }

Issues Detected:
- Metrics API unavailable
- Mismatch: 2/4 replicas available.
- Deployment is not ready.
- Pod nginx-deployment-5c96bb98f8-rpmtr is in 'Pending' state. Reason: ImagePullBackOff


Note:

To acess the Nginx url: curl http://<minikube_ip>:<node_port>
-> To get minikube IP: >> minikube ip
-> To get the Node port, check the Port(s) column: >> kubectl get service -n nginx-namespace

-> Sometimes, services running in Minikube are not directly accessible due to networking constraints. You can create a tunnel to expose the service:
    >> minikube tunnel

-> This command runs in the foreground, so open a new terminal for further commands.
    >> kubectl port-forward svc/nginx-service 8080:80 -n nginx-namespace
-> Now, access NGINX at: http://localhost:8080

-> Debugging Logs: >> kubectl logs <pod_name> -n nginx-namespace