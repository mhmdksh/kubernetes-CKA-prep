# Troubleshooting
## Troubleshooting Application Failure
Application failure can happen for many reasons, but there are ways within Kubernetes that make it a little easier to discover why. In this lesson, we’ll fix some broken pods and show common methods to troubleshoot.

The YAML for a pod with a termination reason:

    apiVersion: v1
    kind: Pod
    metadata:
      name: pod2
    spec:
      containers:
      - image: busybox
        name: main
        command:
        - sh
        - -c
        - 'echo "I''ve had enough" > /var/termination-reason ; exit 1'
        terminationMessagePath: /var/termination-reason

One of the first steps in troubleshooting is usually to describe the pod:

    kubectl describe po pod2

The YAML for a liveness probe that checks for pod health:

    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness
    spec:
      containers:
      - image: linuxacademycontent/candy-service:2
        name: kubeserve
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8081

View the logs for additional detail:

    kubectl logs pod-with-defaults

Export the YAML of a running pod, in the case that you are unable to edit it directly:

    kubectl get po pod-with-defaults -o yaml --export > defaults-pod.yaml

Edit a pod directly (i.e., changing the image):

    kubectl edit po nginx
## Troubleshooting Control Plane Failure
The Kubernetes Control Plane is an important component to back up and protect against failure. There are certain best practices you can take to ensure you don’t have a single point of failure. If your Control Plane components are not effectively communicating, there are a few things you can check to ensure your cluster is operating efficiently.

Check the events in the kube-system namespace for errors:

    kubectl get events -n kube-system

Get the logs from the individual pods in your kube-system namespace and check for errors:

    kubectl logs [kube_scheduler_pod_name] -n kube-system

Check the status of the Docker service:

    sudo systemctl status docker

Start up and enable the Docker service, so it starts upon bootup:

    sudo systemctl enable docker && systemctl start docker

Check the status of the kubelet service:

    sudo systemctl status kubelet

Start up and enable the kubelet service, so it starts up when the machine is rebooted:

    sudo systemctl enable kubelet && systemctl start kubelet

Turn off swap on your machine:

    sudo su -
    swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab

Check if you have a firewall running:

    sudo systemctl status firewalld

Disable the firewall and stop the firewalld service:

    sudo systemctl disable firewalld && systemctl stop firewalld
## Troubleshooting Worker Node Failure
Troubleshooting worker node failure is a lot like troubleshooting a non-responsive server, in addition to the kubectl tools we have at our disposal. In this lesson, we’ll learn how to recover a node and add it back to the cluster and find out how to identify when the kublet service is down.

Listing the status of the nodes should be the first step:

    kubectl get nodes

Find out more information about the nodes with kubectl describe:

    kubectl describe nodes chadcrowell2c.mylabserver.com

You can try to log in to your server via SSH:

    ssh chadcrowell2c.mylabserver.com

Get the IP address of your nodes:

    kubectl get nodes -o wide

Use the IP address to further probe the server:

    ssh cloud_user@172.31.29.182

Generate a new token after spinning up a new server:

    sudo kubeadm token generate

Create the kubeadm join command for your new worker node:

    sudo kubeadm token create [token_name] --ttl 2h --print-join-command

View the journalctl logs:

    sudo journalctl -u kubelet

View the syslogs:

    sudo more syslog | tail -120 | grep kubelet
## Troubleshooting Networking
Network issues usually start to arise internally or when using a service. In this lesson, we’ll go through the many methods to see if your app is serving traffic by creating a service and testing the communication within the cluster.

Run a deployment using the container port 9376 and with three replicas:

    kubectl run hostnames --image=k8s.gcr.io/serve_hostname \
                            --labels=app=hostnames \
                            --port=9376 \
                            --replicas=3

List the services in your cluster:

    kubectl get svc

Create a service by exposing a port on the deployment:

    kubectl expose deployment hostnames --port=80 --target-port=9376

Run an interactive busybox pod:

    kubectl run -it --rm --restart=Never busybox --image=busybox:1.28 sh

From the pod, check if DNS is resolving hostnames:

    # nslookup hostnames

From the pod, cat out the /etc/resolv.conf file:

    # cat /etc/resolv.conf

From the pod, look up the DNS name of the Kubernetes service:

    # nslookup kubernetes.default

Get the JSON output of your service:

    kubectl get svc hostnames -o json

View the endpoints for your service:

    kubectl get ep

Communicate with the pod directly (without the service):

    wget -qO- 10.244.1.6:9376

Check if kube-proxy is running on the nodes:

    ps auxw | grep kube-proxy

Check if kube-proxy is writing iptables:

    iptables-save | grep hostnames

View the list of kube-system pods:

    kubectl get pods -n kube-system

Connect to your kube-proxy pod in the kube-system namespace:

    kubectl exec -it kube-proxy-cqptg -n kube-system -- sh

Delete the flannel CNI plugin:

    kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml

Apply the Weave Net CNI plugin:

    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
