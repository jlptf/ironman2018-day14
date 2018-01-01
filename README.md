## Day 14 - Volume (1)

### 本日共賞

* Volume
* Volume 種類

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)

<br/>

#### Volume

今天想聊聊 k8s 中的 Volume。還記得我們在 [Day 10 - 建構組件](https://ithelp.ithome.com.tw/articles/10193513) 曾經提到過 Pod 有臨時性的特性 (ephemeral)，意思就是，當 Pod 被刪除的同時，所有屬於 Pod 的資料便會一並被刪除。而當 Pod 被刪除時，k8s 會嘗試重新建立 Pod 這時候新建的 Pod 的內容並 **不會包含被刪除 Pod 的內容**。

> * 如果直接部署 Pod 物件，當刪除 Pod 物件時 k8s 並不會建立新的 Pod，而是建立 Deployment 物件後，刪除該物件對應的 Pod，k8s 才會重新建立 Pod。
> * `不會包含被刪除 Pod 的內容` 指的是 Pod 運行後產生的內容，並非映像檔內容，映像檔原始內容還是會一致。

因此，為了克服這個問題，k8s 提供了 Volume 物件來儲存需要保存的資料。簡單來看可以當作是一個目錄且包含在 Pod 內獨立存活在容器外。

> 還記得 Pod 內的容器是共用 Volume 嗎？

<br/>

#### Volume 種類

Volume 種類其實還蠻多的，詳細內容可參考 [官網說明](https://kubernetes.io/docs/concepts/storage/volumes/)，底下列舉幾個常用的種類

*1. emptyDir*

`emptyDir` 依舊是屬於臨時性的目錄，當 Pod 被刪除的同時該目錄也會被刪除。而在 Pod 被建立的同時，k8s 會自動地分配一個目錄，即無須指定 Node 上的目錄。另外要注意的一點是，如果容器崩潰 (crash)，該目錄還是會留著。只有 Pod 被刪除才會一併刪除。

`emptyDir` 可以應用在

* 暫存空間
* 多個容器的共享空間


底下為 `emptyDir` 的一個例子

```
# volume-empty.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: volume-empty-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /tmp/conf  <=== 將 empty-volume 掛載到 /tmp/conf 下
      name: empty-volume
  volumes:
  - name: empty-volume
    emptyDir: {}            <=== 指定為 emptyDir 
```

試試部署到 k8s 並觀察狀態

```bash
$ kubectl apply -f volume-empty.yaml
pod "volume-empty-nginx" created

$ kubectl get pods
NAME                 READY     STATUS    RESTARTS   AGE
volume-empty-nginx   1/1       Running   0          16m
```

接著透過指令連到 `volume-empty-nginx` Pod 並查看 `/tmp/conf`

```bash
$ kubectl exec -it volume-empty-nginx bash
root@volume-empty-nginx:/# ls -al /tmp
total 12
drwxrwxrwt 1 root root 4096 Dec  4 07:22 .
drwxr-xr-x 1 root root 4096 Dec  4 07:22 ..
drwxrwxrwx 2 root root 4096 Dec  4 07:22 conf
```

> `kubectl exec` 請參考 [Get a Shell to a Running Container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/)

你會發現 `/tmp/conf` 已經掛載成功

> 可以試試看把 yaml 中的 Volume 宣告移除後再部署到 k8s 看看有什麼變化 
> 
> 記得先移除 Pod，忘記怎麼移除可以參考 [Day 8 - 與 k8s 溝通: kubectl 之 8. 刪除資源](https://ithelp.ithome.com.tw/articles/10193502)

*2. hostPath*

使用 `hostPath` 可以指定一個 Node 的目錄掛載到 Pod 中使用，即便 Pod 被刪除，該目錄的內容也都會存在。`hostPath` 形態的 Volume 可以從 Node 掛載一個檔案或者目錄到 Pod。

底下是一個 `hostPath` Volume 的例子

```
# volume-hostpath.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /tmp/conf     <=== 一樣掛載到 /tmp/conf，不同的 Pod 不怕
      name: hostpath-volume
  volumes:
  - name: hostpath-volume
    hostPath:
      path: /tmp/hostpathdata  <=== 指定 Node 目錄，可更換成其他目錄
```

接著部署到 k8s 中，在 Pod 正常運行後連到該 Pod

```bash
$ kubectl apply -f volume-hostpath.yaml
pod "volume-hostpath-nginx" created

$ kubectl exec -it volume-hostpath-nginx bash
root@volume-hostpath-nginx:/#
```

接著在 `/tmp/conf` 建立一個 mypath 檔案後，鍵入 `exit` 離開：

```bash
root@volume-hostpath-nginx:/# echo "My Host Path" >> /tmp/conf/mypath

root@volume-hostpath-nginx:/# exit

```

接著把 `volume-hostpath-nginx` 移除

```bash
$ kubectl delete pods volume-hostpath-nginx
pod "volume-hostpath-nginx" deleted
```

接著，我們到 Node 查看剛剛建立的檔案是否還存在，由於我們使用 minikube，因此 Node 指的就是 minikube 運行的虛擬機。可以利用下列指令連到 minikube 

> 在 k8s 叢集中 Node 可能會有很多台，只是目前我們使用 minikube 做示範故只有一台 Node

```bash
$ minikube ssh
                         _             _            
            _         _ ( )           ( )           
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __  
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ cat /tmp/hostpathdata/mypath 
My Host Path
```

mypath 還存在在 `/tmp/hostpathdata` 目錄底下，證明的確不會因為 Pod 刪除而被刪除。

雖然 `hostPath` 是保存資料的方法，不過一般不建議這樣使用，因為概念上來看，Pod 與 Node 是沒有依存關係。如果使用這種方式，則 Pod 的運行可能會需要 Node 上的某個檔案。但是，如果是需要直接存取函式庫 (理論上每台 Node 都應該要有) 這時候就可以使用 `hostPath`。

> 由於 Pod 是會被部署到任意 Node 中，因此透過 `hostPath` 產生的檔案，很有可能不存在在所有的 Node 中，你應該可以預見會有什麼事情發生了吧！

*3. gcePersistentDisk*

k8s 可以直接支援 [Google Compute Engine (GCE)](https://cloud.google.com/compute/docs/) 上的磁碟，用的就是 `gcePersistentDisk` 這類型的 Volume。在後面的文章我們再來說明如何使用。

*4. awsElasticBlockStore*

類似 `gcePersistentDisk`，k8s 同樣支援 [Amazon Web Service (AWS)](https://aws.amazon.com/tw/) 中的 [EBS Volume](https://aws.amazon.com/tw/ebs/) 使用的就是 `awsElasticBlockStore` 

*5. azureDisk*

`azureDisk` 指的是 [Microsoft Azure](https://azure.microsoft.com/zh-tw/) 的 [Data Disk](https://docs.microsoft.com/zh-tw/azure/virtual-machines/linux/about-disks-and-vhds?toc=%2Fazure%2Fvirtual-machines%2Flinux%2Ftoc.json)，同樣的，k8s 可與 Azure 串接。

*6. secret*

Secret，k8s 中另一個重要的物件且可以用 Volume 的形式掛載到 Pod 內使用。

> 關於 Secret 我們會在之後的文章再更詳細的說明。

明天我們再接著欣賞 Persisten Volumes 與 Persistent Volume Claims

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day14/](https://jlptf.github.io/ironman2018-day14/)