# Kubernetes Guide to deploy a HA cluster with KeepAlived, HAProxy, metalLB and kubeadm

![diagramgit](https://user-images.githubusercontent.com/31323133/110505246-f90c9580-80cb-11eb-882f-2d7e601432db.png)

Ejecutar en los nodos Master y Workers

Tener en cuenta, realizar cada apt update en cada lugar que es indicado, debido a que cada apt update aporta diferentes actualizaciones en las diferentes fases de la instalación, debido a la inclusión de nuevos repositorios. 

sudo apt update

sudo apt -y upgrade && sudo systemctl reboot

sudo apt -y install curl apt-transport-https

Deshabilitar el swap 

sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

Comentar la linea /swapfile  o la variable label que apunta al swap

sudo nano /etc/fstab



Inicia la instalación 

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt -y install vim git curl wget kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl

sudo modprobe overlay

sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

Instalar Docker+Containerd como runtime

sudo apt update

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update

sudo apt install -y containerd.io docker-ce docker-ce-cli

sudo mkdir -p /etc/systemd/system/docker.service.d

sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker

systemctl enable kubelet

sudo kubeadm config images pull

echo 1 > /proc/sys/net/ipv4/ip_forward

nano /etc/sysctl.conf



Configuración de los Load Balancers

Para el balanceador de carga se utilizará haproxy, para garantizar High Avaibility se utilizará keepAlived con el protocolo Virtaul Redundacy Routing Protocol, estos comandos sólo iran en los dos nodos con la función LB.

Instalación de KeepAlived y configuración del VRRP

sudo apt-get update
sudo apt-get install build-essential libssl-dev
sudo apt-get install keepalived

LoadBalancer 1

vim /etc/keepalived/keepalived.conf

global_defs {
   notification_email {
     support@calipso.co
   }
   notification_email_from jhanc_diaz@calipso.co
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0   #Interfaz la cual vamos a usar para conectarnos a los masters
    virtual_router_id 101   
    priority 101    #Prioridad, define cual va a ser el nodo activo y cual el pasivo
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.74.84.75  #IP Virtual que queremos usar para el lb
    }
}

LoadBalancer 2

vim /etc/keepalived/keepalived.conf

global_defs {
   notification_email {
     support@calipso.co
   }
   notification_email_from jhanc_diaz@calipso.co
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0   #Interfaz la cual vamos a usar para conectarnos a los masters
    virtual_router_id 101
    priority 100      #Prioridad, define cual va a ser el nodo activo y cual el pasivo
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.185.73.67  #IP Virtual que queremos usar para el lb como entidad
    }
}

Ambos LoadBalancer

service keepalived start
sudo apt update && apt install -y haproxy
vim /etc/haproxy/haproxy.cfg


Al final agregar:

frontend kubernetes-frontend
    bind  10.185.73.67:6443     #Dirección general de ambos LoadBalancers y puerto usado por K8S
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-masterdev01 10.74.84.125:6443 check fall 3 rise 2   #Hostname, IP red interna y puerto k8s del master
    server k8s-masterdev02 10.185.73.68:6443 check fall 3 rise 2   #Hostname, IP red interna y puerto k8s del master


Ejecutar:
systemctl restart haproxy

Importante:

Este comando va a generar un error en el lb02, debido a que lb02 esta en state backup y aún no crea la IP virtual 10.74.84.75,
Haciendo el haproxy falle, entonces lo que se debe hacer en el lb01 es reiniciarlo con reboot, esto hará que el lb02 quede como maestro y se cree la IP virtual, luego en el lb02 ejecute systemctl restart haproxy

En el master01

kubeadm init --control-plane-endpoint=" 10.185.73.67:6443" --upload-certs --apiserver-advertise-address=<IPMaster01> --pod-network-cidr=192.168.0.0/16


Como resultado se obtendrá algo parecido al siguiente ejemplo

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.74.84.75:6443 --token d12xz3.l02dpzix58gy3k1m \
    --discovery-token-ca-cert-hash sha256:35415f7fd915aafcbc86b23ff074c07c1ef08abdf77a5811683ebfb210525ad4 \
    --control-plane --certificate-key 478bcd1bb75c459b9641ac614220c5ce88930e7f5292d2ff48482d753aa520c3 --apiserver-advertise-address=10.185.73.68

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.74.84.75:6443 --token d12xz3.l02dpzix58gy3k1m \
    --discovery-token-ca-cert-hash sha256:35415f7fd915aafcbc86b23ff074c07c1ef08abdf77a5811683ebfb210525ad4


Ejecutamos lo siguiente el master01

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Para agregar otro máster usamos lo siguiente y le agregamos --apiserver-advertise-address=<IPRedInternaNuevoMaster>

kubeadm join  10.185.73.67:6443 --token d12xz3.l02dpzix58gy3k1m \
    --discovery-token-ca-cert-hash sha256:35415f7fd915aafcbc86b23ff074c07c1ef08abdf77a5811683ebfb210525ad4 \
    --control-plane --certificate-key 478bcd1bb75c459b9641ac614220c5ce88930e7f5292d2ff48482d753aa520c3 --apiserver-advertise-address=<IPMaster>

PD: Este es un ejemplo, el suyo estará en la consola. Importante agregar ese parámetro al final, debido a que por defecto no viene

Luego de agregar las N cantidad de másters para HA se agregan la N cantidad de workers con el segundo proveido por k8s

kubeadm join  10.185.73.67:6443 --token d12xz3.l02dpzix58gy3k1m \
    --discovery-token-ca-cert-hash sha256:35415f7fd915aafcbc86b23ff074c07c1ef08abdf77a5811683ebfb210525ad4


Finalmente se agrega la subred Calico

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

Para ejecutar comandos en los demás másters (que no sea el master01) se debe usar el siguiente subfijo despues del kubectl

kubectl --kubeconfig=/etc/kubernetes/admin.conf

Ejemplo de un get nodes

kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes

![image](https://user-images.githubusercontent.com/31323133/110505591-4a1c8980-80cc-11eb-86dd-a50002076ef1.png)

