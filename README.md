Last update: 2023/11  
# Pure-PortworxBackup-WorkGuide

### Pure Portworx Backup簡易介紹     

Pure Portworx 是Pure買進的雲原生解決方案  
Pure Storage 的 Portworx® 提供了全面整合的解決方案，包含下列功能：持久性儲存、資料保護、災害復原、資料安全性、跨雲端和資料搬移，以及針對在 Kubernetes 上運行的應用程式自動進行容量管理。  

主要分為三個功能  
Portworx Enterprise  
Portworx Backup  
Portworx Data Services  

本篇主要針對Portworx Backup進行安裝測試  

TIPs:許多進階功能，包含HA，資料安全都必須整合Portworx entprise才能使用  
Portworx Backup也是Portworx其中一個套件，但是可以單獨使用  
計價方式: 節點數量  

[參考網站](https://www.purestorage.com/tw/products/cloud-native-applications/portworx.html "link")  


測試不包含性能測試  

### 安裝步驟   

## 事前準備  


  
## Linux Client 準備  

環境更新及安裝基本套件  
```
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get -y install vim build-essential curl ssh
sudo apt-get install net-tools
```

安裝Docker engine    
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

確認安裝版本
```
sudo docker --version
```

安裝helm
```
cd
sudo apt-get install -y curl
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
``` 

安裝MINIO  
測試使用在地端建立一個S3空間的方式  
```
sudo docker run -d -p 9000:9000 -p 9090:9090 --name pure   -e "MINIO_ROOT_USER=pure"   -e "MINIO_ROOT_PASSWORD=P@ssw0rd"   -v /mnt/data:/data   --restart=always  minio/minio server /data --console-address ':9090'
```

之後透過網頁來進行產生s3 bucket

![img](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide/blob/main/img/minio01.png "img")  
![img](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide/blob/main/img/minio02.png "img")  
![img](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide/blob/main/img/xx "img")  

安裝NFS  
同時間也安裝NFS  
Portworx Backup同時支援兩種模式(S3,NFS)  
```
sudo apt-get install nfs-kernel-server nfs-common
mkdir nfsshare
sudo chmod -R 777 /home/ubuntu/nfsshare/
```
編輯/etc/exports  
```
sudo vim /etc/exports  
#新增以下
/home/ubuntu/nfsshare/    *(rw,sync,no_root_squash,no_all_squash)
```

重啟服務  
```
/etc/init.d/nfs-kernel-server restart
``` 

## Kubernetes 操作及環境準備    

確認Kuberentes服務  
已下指令是Tanzu環境登入的指令  
如果是其它Kubernetes的平台，需要確認能夠正常的執行Kubernetes的相關操作  
可以跳轉到下一章節  

登入到Taznu環境(如果是其他native環境可以不用)  
```
kubectl vsphere login --server=x.x.x.x --insecure-skip-tls-verify  --vsphere-username xx@xx.com --tanzu-kubernetes-cluster-name  [tkc-name]
kubectl config use-context [tkc-name]
```

下載gcallowroot yaml(TKC需要、Pure Portworx需要)  
```
sudo apt-get install -y git
cd 
git clone https://github.com/ReSin-Yan/NTUSTCourse
cd NTUSTCourse/Kubernetes
kubectl apply -f gcallowroot.yaml  
```

安裝NFS sub-dir(如有Kubernetes本身已有StorageClass也建議設定)  
```
cd 
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=[ip] \
    --set nfs.path=/home/ubuntu/nfsshare
```
安裝stork(portworx連結所需要功能)  
以下提供單獨stork功能安裝，不包含portworx entprise  
```
curl -fsL -o stork-spec.yaml "https://install.portworx.com/pxbackup?comp=stork&storkNonPx=true"
kubectl apply -f stork-spec.yaml
```

## 安裝說明  
請參考以下安裝參數設定 
會標明必須或是需要討論  
版本為2.6  
安裝方式為對外(可以改成air-gapped)  
可以由以下兩種方式取得安裝指令  
由官網產生  



或是直接參考以下指令  
```
kubectl create ns central
helm repo add portworx http://charts.portworx.io/ && helm repo update
helm install px-central portworx/px-central --namespace central \
--create-namespace \
--version 2.6.0 \
--set persistentStorage.enabled=true,persistentStorage.storageClassName="wcppolicy",pxbackup.enabled=true
```

### 可以討論的參數  

[參考網站](https://docs.portworx.com/portworx-backup-on-prem/reference/install-helm-chart "link")  

沒有特別需要注意的參數  
比較需要特別注意的是AirGapped的安裝過程會需要輸入比較多的參數  

如果可以配合建立命名空間 central就輸入以下，不行則填入可以提供的namespace  
```
--namespace=central
```



ingress網路 or LoadBalance  
網路服務此段會根據使用者有沒有L4 or L7的服務來進行設定  
考量到的點是，POC情境下，使用者不接受tunnel的方式or服務正式上線  
Portworx預設為Loadbalance，如果沒有安裝可以參考以下  
[MetalLB安裝過程](https://github.com/ReSin-Yan/kubernetes-Course/tree/main/Kubernetes/3.service/LoadBalance "link")  

L4 LoadBalance(預設為loadbalance)  
```
--set service.pxBackupUIServiceType=LoadBalancer
```

儲存空間    
wcppolicy可以更換成自己的storageclass名稱
```
--set persistentStorage.enabled=true
--set persistentStorage.storageClassName="wcppolicy"
--set pxbackup.enabled=true
```

### 額外指令  

刪除指令參考以下  
```
helm uninstall k10 -n kasten-io
```


## 環境設定    

### Cluster Profile  

### Location Profile  

通過以下指令找到服務UI  
```
kubectl get svc -n central
```

預設帳號密碼皆為`admin`  

新增cluster至portworx backup 


Location Profile目前總共支援6種模式  
其中分為三大公有雲空間  
以及地端三種空間NFS、S3、VBR  
其中VBR只有支援Tanzu  

新增Cloud Accounts  
其中地端S3請選擇AWS / S3 這一欄  
img  
img  



新增S3空間  
使用預先準備的minio空間([MINIO](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide#linux-client-%E6%BA%96%E5%82%99   "link") )   
![img](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide/blob/main/img/03s3.png "img")  

設定完成可以再minio空間看到資料夾建立  
![img](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide/blob/main/img/04s3.png "img")  

設定VBR前需要先設置vsphere Infrastructure Profiles([Infrastructure](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide#infrastructure-profiles   "link") )   
之後依序輸入資訊  
![img](https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide/blob/main/img/05vbr.png "img")  



新增NFS空間  
使用NFS當作儲存空間  



## 測試環境建立    

建立兩個命名空間  
分別對應不同種類的CSI(測試環境為vmware-csi,NFS-subdir csi)  
```
kubectl create ns nfs-csi
kubectl create ns vsan-csi
```

根據兩個volume建立不同的部屬環境  

```
cd
git clone https://github.com/ReSin-Yan/Veeam-Kasten-WorkGuide.git
cd Veeam-Kasten-WorkGuide/nfscsi/
kubectl apply -f pre.yaml  -n nfs-csi
kubectl apply -f post.yaml  -n nfs-csi
kubectl get svc -n nfs-csi
```

```
cd 
cd Veeam-Kasten-WorkGuide/vsancsi/
kubectl apply -f pre.yaml  -n vsan-csi
kubectl apply -f post.yaml  -n vsan-csi
kubectl get svc -n vsan-csi
```

分別找到服務網址  
```
kubectl get svc -A
```

開啟服務之後  
預設帳號密碼為`admin`   


## 測試功能建立    

分別備份以下  



