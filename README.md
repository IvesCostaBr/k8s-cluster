# Configurando Cluster k8s CLI

## 1 - Ativar tráfego em ponte do iptables

```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
    sudo sysctl --system

```

## 2 Desabilite a troca em todos os nós

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 3 Instalar Docker Container Runtime em todos os nós

```bash
    sudo apt-get update -y
    sudo apt-get install -y \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg \
        lsb-release
    
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

    echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update -y
    sudo apt-get install docker-ce docker-ce-cli containerd.io -y

    sudo apt-get update -y
    sudo apt-get install docker-ce docker-ce-cli containerd.io -y

    sudo systemctl enable docker
    sudo systemctl daemon-reload
    sudo systemctl restart docker
```

## 4 Instale Kubeadm & Kubelet & Kubectl em todos os nós.

```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update -y
    sudo apt-get install -y kubelet kubeadm kubec

    apt update
    apt-cache madison kubeadm | tac
    
    #você pode especificar a versão conforme mostrado abaixo.
    sudo apt-get install -y kubelet=1.22.8-00 kubectl=1.22.8-00 kubeadm=1.22.8-00

    #Adicione retenção aos pacotes para evitar atualizações.
    sudo apt-mark hold kubelet kubeadm kubectl
```

## 5 Inicialize o Kubeadm no nó mestre para configurar o plano de controle

##### obs:

IPADDR="10.0.0.10" ,- IP publico do nó mestre
NODENAME=$(hostname -s)


```bash
    sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=192.168.0.0/16 --node-name $NODENAME --ignore-preflight-errors Swap
```

Se der erro remova ```--ignore-preflight-errors Swap``` e tente novamente.

Pode ocorrer o erro de service unknow.Caso aconteça execute o comando abaixo removendo e restartando o serviço de container.

```bash
    rm /etc/containerd/config.toml
    systemctl restart containerd
```

## 6 criar o kubeconfig

```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 7 verificar kubeconfig executando 

```
    kubectl get po -n kube-system
```

## [optional] 
 Por padrão, os aplicativos não serão agendados no nó mestre. Se você quiser usar o nó mestre para agendar aplicativos, manche o nó mestre.

```bash
    kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 8 Instalar calico

```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml 
```
## 9 Unir nós de trabalho ao nó mestre do Kubernetes

```bash
    kubeadm token create --print-join-command
```

vair retornar algo assim:

```bash
    sudo kubeadm join 10.128.0.37:6443 --token j4eice.33vgvgyf5cxw4u8i \
    --discovery-token-ca-cert-hash sha256:37f94469b58bcc8f26a4aa44441fb17196a585b37288f85e22475b00c36f1c61
```

Basta executar esse comando nas outras maquinas do cluster para conectar ao node 
master.

Verificar se foram adicionadas:

```bash
    kubectl get nodes
```

### POssiveis problemas do kubeadm
    A seguir estão os possíveis problemas que você pode encontrar na configuração do kubeadm.

1 - Pod sem memória e CPU: o nó mestre deve ter no mínimo 2 vCPU e 2 GB de memória.
2 - Os nós não podem se conectar ao mestre: verifique o firewall entre os nós e certifique-se de que todos os nós podem se comunicar entre si nas portas necessárias do kubernetes.
3 - Reinicializações do pod Calico: às vezes, se você usar o mesmo intervalo de IP para a rede de nó e pod, os pods Calico podem não funcionar conforme o esperado. Portanto, certifique-se de que os intervalos de IP do nó e do pod não se sobreponham. A sobreposição de endereços IP também pode resultar em problemas para outros aplicativos em execução no cluster.
