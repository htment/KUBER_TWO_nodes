Похоже, у вас проблемы с доступностью конкретных версий пакетов Kubernetes в репозиториях. Давайте решим это правильно:

1. Сначала полностью очистим систему от остатков Kubernetes:
bash
sudo kubeadm reset -f
sudo apt-get purge -y kubelet kubeadm kubectl --allow-change-held-packages
sudo apt-get autoremove -y
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd $HOME/.kube
2. Обновим список пакетов и установим зависимости:
bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
3. Добавим официальный репозиторий Kubernetes (новый формат):
bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
4. Установим конкретные версии компонентов:
bash
sudo apt-get install -y kubelet=1.28.5-1.1 kubeadm=1.28.5-1.1 kubectl=1.28.5-1.1
sudo apt-mark hold kubelet kubeadm kubectl
Если конкретная версия 1.28.5 не доступна, попробуйте установить последнюю стабильную версию 1.28:

bash
sudo apt-get install -y kubelet kubeadm kubectl
5. Проверим установленные версии:
bash
kubeadm version
kubectl version --client
kubelet --version


----------------------------------------------------------------------


1.1. Отключим swap (обязательно для Kubernetes):

bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab


2. Настроим container runtime (containerd):
2.1. Проверим статус containerd:

bash
sudo systemctl status containerd
2.2. Если containerd не работает:

bash
sudo systemctl enable --now containerd
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
3. Переустановим kubelet с правильными параметрами:
3.1. Сбросим предыдущие настройки:

bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet
3.2. Настроим cgroup driver:

bash
sudo mkdir -p /var/lib/kubelet
cat <<EOF | sudo tee /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
4. Инициализируем кластер с явным указанием runtime:
bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.31.119 \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-cert-extra-sans=192.168.31.119 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --ignore-preflight-errors=Swap





----------------------------------------------------------------------





6. Инициализация кластера (после успешной установки):
bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.31.119 \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-cert-extra-sans=192.168.31.119
Если пакеты всё ещё недоступны:
Проверим доступные версии:

bash
apt-cache madison kubelet
Можно попробовать альтернативный репозиторий:

bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
Или использовать более новую версию Kubernetes (например, 1.29):

bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl







1. Настроим доступ к кластеру для текущего пользователя:
bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config




. Установим сетевой плагин (Flannel):
bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
3. Проверим состояние кластера:
bash
kubectl get nodes
kubectl get pods -n kube-system
Через несколько минут нода должна перейти в состояние Ready, а все системные поды в состояние Running.

4. Для добавления worker-ноды (Deb-testkube4):
На worker-ноде выполните команду, которая была показана в выводе (сохраните её):

bash
kubeadm join 192.168.31.119:6443 --token 01hw1r.xlnkgt1ak69u0s6y \
        --discovery-token-ca-cert-hash sha256:752929bda15a6626eb132601c21f11660285e3f116a6c0f4e5ba51a5ecede33d
5. Дополнительные проверки:
Убедимся, что все компоненты работают корректно:

bash
kubectl get cs
kubectl get all -A
6. Если нужно использовать кластер с локальной машины:
Скопируйте /etc/kubernetes/admin.conf на свою локальную машину в ~/.kube/config и установите правильные права.

7. (Опционально) Включение schedule на master-ноде:
Если нужно запускать поды на master-ноде:

bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
8. Полезные команды для проверки:
bash
# Проверить логи kubelet
journalctl -u kubelet -f

# Просмотреть события кластера
kubectl get events -A -w

# Проверить состояние сети
kubectl run -it --rm --restart=Never busybox --image=busybox -- sh
Теперь у вас есть работающий Kubernetes кластер! Для дальнейшей настройки вы можете:

Установить Ingress-контроллер (например, Nginx)

Настроить хранилище (StorageClass)

Установить dashboard или другие инструменты мониторинга








🔹 Как сделать токен бессрочным?
Не рекомендуется для безопасности, но можно создать токен с очень большим TTL:

bash
kubeadm token create --ttl 87600h --print-join-command  # 10 лет
