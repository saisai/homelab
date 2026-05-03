# k8s on proxmox <br />

```
sudo apt update && sudo apt upgrade -y
sudo reboot
```

```
sudo apt install qemu-guest-agent -y
```

```
sudo swapoff -a
sudo nano /etc/fstab
```


and we comment out the line with `swap.image`  <br />

Now run `cat /etc/hostname` to see if all your hostnames are set correctly <br />
Also run `cat /etc/hosts` to see if each of them has host entry <br />

We need to load specific Kernel modules for overlay file system and bridged traffic so we need to run: <br />
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

If we run `cat /etc/modules-load.d/k8s.conf` we can see those 2 lines added to that k8s.conf file <br />
When your Linux system boots up, it reads all files in that directory and automatically loads the listed modules. <br />
It ensures the overlay (for containers) and br_netfilter (for bridge networking) are always there even after reboots. <br />

We can then run below commands to load these modules into memory immediately without the need to reboot the system: <br />
```
sudo modprobe overlay
sudo modprobe br_netfilter
```

Overlay File System is a file system used by docker images, so its necessary for kuberenetes as it job is to manage docker containers. <br />
BR Netfilter in technical terms allows the Linux kernel to pass traffic flowing through a network bridge (Layer 2) to the iptables/netfilter stack (Layer 3) for processing. <br />
Basically  overlay handles how containers see files, and br_netfilter handles how containers can talk to each other. <br />

Below are system parameters, you can copy-paste them and they should persist across reboots. <br />
Simply loading the previous module isn't enough - you also have to tell the kernel to actually use it for IPv4 and IPv6 traffic and that's why we need below: <br />

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

We can check what has changed by running `cat /etc/sysctl.d/k8s.conf` <br />
We should see those 3 lines from command above. <br />
Now we apply those sysctl parameters to running session with <br />
```
sudo sysctl --system
```

We are ready now to install kubernetes components. First we need to run: <br />
```
sudo apt install containerd -y
```


to install container runtime for our k8s cluster <br />
Run `systemctl status containerd` to see if this service is up and running <br />

Now we need to create default configuration for that containerd daemon with: <br />
```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
We can run `cat /etc/containerd/config.toml` to see the file. <br />
We can also grep for a specific line we have to change with <br />
```
cat /etc/containerd/config.toml | grep SystemdCgroup
```

We can see this value is currently set to 'false' and we need it 'true' so we run: <br />

```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

And if we click up arrow to that grep command - you will see it is now set to 'true'. So we can now run:  <br />

```
sudo systemctl restart containerd
```
then <br />
```
sudo systemctl enable containerd
sudo systemctl status containerd
```
We should see containerd service both - active and enabled (enabled means it will auto start after reboot) <br />

Then we install all kubernetes components. <br />
These commands will install curl and gpg commands, will add necessary repositories, will pull the packages and install them: <br />

```
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

# Download the public signing key
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```
While kubelet, kubeadm and kubectl work together to run your cluster, they have very distinct jobs: one is the **worker**, one is the **installer**, and one is the **remote control**. <br />
This last 'hold' command will lock the current version of kubelet, kubeadm and kubectl and this is advisable here because otherwise - if you run simple apt update and apt upgrade command - this could crash your kubernetes cluster. <br />
You don't want a standard apt upgrade to accidentally update these components to a newer version that might be incompatible with your current cluster state. <br />
We might want to upgrade them periodically but in a controlled way, and this 'hold' setting let's us do that. <br />

Now we have all packages installed, so its time to initialize our kubernetes cluster. <br />

Optional - reboot them all with: <br />
```
sudo reboot
```
Probably not necessary, but it's a good practise once you install a lot of new system components. <br />

Once rebooted we can close current session and open each one individually <br />
You can use `Ctrl + d` to close the tmux session. <br />

Let's ssh to each instance separately now and run the command on master node only: <br />

To initialize cluster on master node, we run:  <br />
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Best is to not fiddle with this ip prefix - it has nothing to do with our static DHCP scope on router (the only requirement is that this prefix can't be the same as the one configured on our router).  <br />
While generally you can change this 10.244.0.0/16 prefix, it actually needs to match the CNI component we are going to install shortly. <br />
Because I am going to use Flannel - one of the most popular and simplest Container Network Plugin (CNI) - it uses that ip prefix so easiest for you to run this command as it is with that 10.244.0.0/16 ip prefix configured. <br />

When we run that command - at the end of the process, this command generates another 'join' command that we have to use on the worker nodes to join the cluster. <br />
Note that this join command is valid only for 24 hours <br />
If you create another worker node and want to join that node like a week later, you will notice this command stopped working <br />
You can refresh it with `kubeadm token create --print-join-command` <br />

Before we run those commands on worker nodes, let's run another command on master node that will configure kubectl for our user: <br />
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now we need to install so called CNI which is Container Network Interface, which is needed for our cluster as it creates overlay network for communication inside kubernetes cluster. <br />
If you dont't have CNI configured correctly, you will see all your nodes in 'Not Ready' status. <br />
Most popular CNI's are Flannel and Calico, I will go with Flannel which I already mentioned before. <br />
We install that on master node only: <br />

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

We can see everything was succesfully created, so we can copy that command that was generated on master node and paste it into our worker nodes. <br />
Note it says to run it as root so we need sudo, and for me the command was: <br />
```
sudo kubeadm join 192.168.1.191:6443 --token vyg76p.pz5k6dkrkaopvjhi --discovery-token-ca-cert-hash sha256:fb914ae294538ea1d35e18fab62df421f936dd68069eb877aa3b8e49634321a3  
```

All seems to be ok, we can run now `kubectl get nodes` on master node- that will display all master and worker nodes we have in our cluster <br />

You will see the ROLE for worker nodes is empty, but that is actually expected with more modern versions of kubernetes. <br />
The 'worker' role is simply assumed there by default for any nodes that are not master nodes. <br />

So yes - it works! Our entire cluster works as expected. <br />
At this stage you can deploy services to your new Kubernetes cluster. <br />

You could in theory just play with what you already created, but you will very quickly notice one big limitation - you dont have kubernetes service called 'LoadBalancer' available. <br />
You can deploy services of type NodePort or you can create HostNetwork, but if you want more Cloud-like solution, we can make our cluster better. <br />
To run services of type LoadBalancer, probably the easiest way is to add MetalLB service to our cluster. <br />
```
kubectl edit configmap -n kube-system kube-proxy
```
In 'ipvs' you will see `strictARP` set to false, you need to change it to true. <br />
For me it opens in vim, so i need to press `i` then change it, then `Esc` and `:wq` <br />
Then we run this command to install MetalLB: <br />
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
Run this command: <br />
```
kubectl get pods -n metallb-system
```
You should see 4 pods running. It might take a while before they are all up but they will eventually be up and running. <br />
Create a yaml file , i will call it proxmox-ip-pool.yaml in my /home/marek directory and will paste this code: <br />
Remember to adjust the private ip addresses that load balancer service is allowed to use

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: marek-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.245 # CHANGE THIS to your desired range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: layer2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - marek-pool
```

Now apply that config by running: <br />
```
kubectl apply -f proxmox-ip-pool.yaml
```

Now we are ready to test if we are able to deploy services of type LoadBalancer: <br />
Create another yaml file, for example `nano nginx-test.yaml` and paste: <br />
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

So this service is of type LoadBalancer, and the result we hope for is that this service is up and using one of the ip addresses supplied by that MetalLB service configured in previous step, so run: <br />
```
kubectl apply -f nginx-test.yaml
```

If deployment and service are shown as 'created' then that is a good sign, we can check the service with: <br />
```
kubectl get svc nginx-service
```

You should be able to see cluster ip and external ip - the external ip is the one you got from MetalLB and <br />
Just type that ip address in your browser (like `192.168.1.240` for me) and you should see 'Welcome to nginx' <br />
It's the same as running `http://192.168.1.240:80` <br />

You can build any services you like, you can add ingress controller, you can do whatever you want as it is now fully functional kubernetes cluster <br />


