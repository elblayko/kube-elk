# ElasticSearch on Kubernetes Using WSL
This project creates an Elastic stack within Kubernetes using Docker.

## Prerequisties
- [Windows Subsystem for Linux/Ubuntu distribution installed](https://docs.microsoft.com/en-us/windows/wsl/install)
- [Docker Desktop for Windows](https://docs.docker.com/desktop/windows/install)

# Installation
Download and install k3d:
```
sudo su -c "wget -q https://github.com/k3d-io/k3d/releases/download/v5.4.1/k3d-linux-amd64 -O /opt/k3d \
&& chmod +x /opt/k3d \
&& ln -s /opt/k3d /usr/local/bin/k3d" \
&& k3d --version
```

Download and install kubectl (if needed, usually provided by Docker Desktop):
```
sudo su -c "wget -q https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl -O /opt/kubectl \
&& chmod +x /opt/kubectl \
&& ln -s /opt/kubectl /usr/local/bin/kubectl" \
&& kubectl version
```

Set `vm.max_map_count` if applicable:
```
sudo sysctl -w vm.max_map_count=262144
```

Execute `ip address` and locate the IP address value of eth0, example: `172.29.144.81`
```
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:83:6e:c8 brd ff:ff:ff:ff:ff:ff
    inet 172.29.144.81/20 brd 172.29.159.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:fe83:6ec8/64 scope link
       valid_lft forever preferred_lft forever
```

Update Windows DNS as follows:
1. Press the Windows key.
2. Type Notepad in the search field.
3. In the search results, right-click Notepad and select Run as Administrator.
4. From Notepad, open `C:\Windows\system32\drivers\etc\hosts`.
5. Add the following lines at the end of the file:
```
<WSL IP Address> es.elastic.local
<WSL IP Address> kibana.elastic.local
<WSL IP Address> logstash.elastic.local
<WSL IP Address> maps.elastic.local
```
Example:
```
172.25.227.201 es.elastic.local
172.25.227.201 kibana.elastic.local
172.25.227.201 logstash.elastic.local
172.25.227.201 maps.elastic.local
```
Save the changes and close Notepad.

# Cluster Configuration
Create a three node cluster.  Exposes port HTTP, HTTPS, Logstash Beats, and Logstash TCP.  Mounts the `maps` directory from the host to worker nodes.
```
k3d cluster create my-cluster \
--servers 1 --agents 2 \
--port 80:80@loadbalancer \
--port 443:443@loadbalancer \
--port 5000:5000@loadbalancer \
--port 5044:5044@loadbalancer \
--volume $PWD/maps:/elastic-maps-service@agent:0-1 \
--k3s-arg "--disable=traefik@server:0"
```

Deploy Nginx Ingress Controller:
```
kubectl apply -f app/nginx-ingress.yml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s
```

Deploy the Elastic Stack.
```
kubectl apply -f app/

kubectl wait --namespace elasticsearch \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/part-of=elk \
  --timeout=300s
```

Review cluster event log as needed:
```
kubectl rollout status -n elasticsearch sts/elasticsearch
kubectl get events --sort-by=.metadata.creationTimestamp --watch
```

## Result
View Kibana at http://kibana.elastic.local.

Add an item to Elasticsearch via TCP:
```
echo "Ben is a nerd" | nc -q0 172.25.227.201 5000
```

# Uninstall
Delete the cluster:
```
k3d cluster delete my-cluster
```
Uninstall k3d:
```
sudo su -c "rm /usr/local/bin/k3d && rm /opt/k3d"
```
