# kubernetes-on-archlinux

**__OS__**
Archlinux (use install script -> https://cdn.discordapp.com/attachments/943470527090143283/1029056478243463330/install_UEFI.sh)

**__First__**
**RUN AS NORMAL USER**
```sh
cat <<EOF > /etc/hosts
127.0.0.1 archlinux localhost
EOF
sudo pacman -S git fakeroot make gcc --noconfirm
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ../
rm -rf ./yay
yay -S kubectl-bin kubelet-bin kubeadm-bin cni-plugins-bin ethtool ebtables socat conntrack-tools containerd neovim
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
sudo systemctl enable --now containerd 
sudo bash -c "cat <<EOF > /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
EOF" 
sudo bash -c "cat <<EOF >> /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_EXTRA_ARGS=--protect-kernel-defaults=false"
EOF"
sudo systemctl daemon-reload
sudo systemctl enable --now kubelet 
cat > ~/init_kubelet.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- token: "$(openssl rand -hex 3).$(openssl rand -hex 8)"
  description: "kubeadm bootstrap token"
  ttl: "24h"
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0" # Used by Prometheus Operator
scheduler:
  extraArgs:
    bind-address: "0.0.0.0" # Used by Prometheus Operator
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
protectKernelDefaults: true
EOF
```

**__Second__**```sh
nvim /etc/default/grub```**add `systemd.unified_cgroup_hierarchy=false` to `GRUB_CMDLINE_LINUX_DEFAULT`**

```sh
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot
```

**__Third__**
```sh
sudo kubeadm init --config ./init_kubelet.yaml
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
    --namespace kube-system
```
