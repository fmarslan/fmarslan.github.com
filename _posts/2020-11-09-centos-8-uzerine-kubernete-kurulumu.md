---
layout: post
title: "Centos 8 üzerine Kubernete Kurulumu"
categories: bash
---
Tavsiyeler
Öncelikle OS ve Data disk bölümünü eğer mümkünse disk olarak değilse partition oalrak ayırmanızı tavsiye ederim daha sonra olurda başka bir sunucuya taşınmak durumunda kalırsanız sadece image veya mount işlemi ile taşınabilirsiniz.
Data bölümünü küçük partitionlar halinde tutabilirseniz yine yeni ortama taşınma aşamasında işleri kolaylaştıracaktır.
Centos 8 Parition Yönetimi


Kubernete kurulumu yapmak isteyenler minikube, kubernete kurulumu şeklinde 2 farklı kubernete sürümü ile karşılaşacaklardır. mini kube tamamen bilgisayarınızda sanal bir makina oluşturarark size hazır ortam sunacaktır eğer ki basit test vs ile uğraşacaksanız localinizde çalıştıracaksanız bu kurulumu tavsiye ederim fakt burada ben 1 node kubernete kurulumunun adımlarını yazacağım

```bash
eğer deneme vs yaptıysanız daha önceden ilk adım mevcut docker sürümlerini kaldırmak olacaktır.
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
yum için gerekli utility kurulumu
sudo yum install -y yum-utils
repoyu ekleyelim
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
docker kuralım
sudo yum-config-manager --disable docker-ce-stable

Eğer diskimizi LVM olarak kullanacaksak buradaki ayarları yapmalısınız /etc/docker/daemon.json olarak kaydetmemiz gerekiyor.
Docker başlatıyoruz
 sudo systemctl start docker

Docker kurulumunu test ediyoruz
sudo docker run hello-world

Otomatik başlaması için servisi aktif hale getiriyoruz.
sudo  systemctl enable docker

 kubeadm, kubelet, kubectl komutlarını kuruyoruz.
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

Şimdi kubernete kurulumuna başlayabiliriz😊 evet şimdi başlıyoruz aşağıdaki komutu çalıştırıyoruz sonrasında bize çıktı veriyor 
kubeadm init

Sonuç içinden alınan komutlar buna benzer oalcaktır. bu komutlar kubernetesi yönetebilmemiz için ihtiyacımız olan kubeconfig dosyalarını oluşturuyor. eğer birden fazla node kuracaksak join bilgilerini paylaşıyor biz tek node kuracağımız için onlara bakmıyoruz

mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Kubernete network plugin kuruyoruz burada weawe kurduk 
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
Master tainti kaldırıyoruz.
kubectl taint nodes --all node-role.kubernetes.io/master-

kubernete kurulumunu test ediyoruz aşağıdaki komutu çalıştırınca node karşısında ready görmemiz lazım ama centos 8 de bunu ready oalrak  görebilmek için aşağıdaki adıma devam ediyoruz
kubectl get nodes 

centos 8 de iptables kaldırılıp yerine nftables kullanmaya başladı kubernete nftables ile çalışmıyor bu sebepten biz nftables kapatıp yerine iptables aktif hale getirmeliyiz. bu kısım için normalde aşağıdaki komutlar ile değişiklik işe yarıyor ama bazen duruma göre farklı müdahaleler gerekebiliyor ilk olarak firewall kapatıyoruz
systemctl disable --now firewalld
systemctl mask firewalld

sonra reboot edip iptables kuruyoruz ekstradan biz bunuda yaptık ama hack işlemi olduğu için güncellemeler ile değişecektir o yüzdne mantığı anlayıp o anki sürüme göre yöntem belirlemek en iyisi oalcaktır.
sudo yum install iptables-services
systemctl start iptables
systemctl enable iptables
bu işlemlerden sonra node read gördüğümüzde kubernete kurulumumuz tamamlanmıştır. storage deployment işlemlerinizi yapabilirsiniz.
```
