# Домашнее задание к занятию «Хранение в K8s. Часть 2»

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке NFS в MicroK8S](https://microk8s.io/docs/nfs). 
2. [Описание Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). 
3. [Описание динамического провижининга](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). 
4. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1

**Что нужно сделать**

Создать Deployment приложения, использующего локальный PV, созданный вручную.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории. 
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.
5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

### Ответ 

Манифест для pv (persistent_volume.yaml):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - fhmrlj60b9llbp4hne8n  # имя ноды
```

Манифест для pvc (pvc.yaml):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: local-storage
  resources:
    requests:
      storage: 1Gi
```

Манифест deployment (deployment.yaml), в нем для теста в одном pod-е создается два контейнера:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-multitool
  template:
    metadata:
      labels:
        app: busybox-multitool
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh"]
          args:
            - -c
            - >
              while true; do
                echo "$(date)" >> /shared/data.txt;
                sleep 5;
              done
          volumeMounts:
            - name: shared-storage
              mountPath: /shared
        - name: multitool
          image: praqma/network-multitool
          command: ["/bin/sh"]
          args:
            - -c
            - >
              while true; do
                clear;
                cat /shared/data.txt;
                sleep 5;
              done
          volumeMounts:
            - name: shared-storage
              mountPath: /shared
      volumes:
        - name: shared-storage
          persistentVolumeClaim:
            claimName: local-pvc
```

Вывод команд создание pv, pvc, deployment:
```bash 
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl apply -f persistent_volume.yaml
persistentvolume/local-pv created
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl apply -f pvc.yaml 
persistentvolumeclaim/local-pvc created
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv   1Gi        RWX            Retain           Bound    default/local-pvc   local-storage   <unset>                          29s
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ 
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
local-pvc   Bound    local-pv   1Gi        RWX            local-storage   <unset>                 32s
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ 
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl apply -f deployment.yaml
deployment.apps/busybox-multitool created
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
busybox-multitool-8d5948b97-n7hl5   2/2     Running   0          9s
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ 
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
busybox-multitool-8d5948b97-n7hl5   2/2     Running   0          27s   10.1.85.70   fhmrlj60b9llbp4hne8n   <none>           <none>
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$
```

Проверка наличия доступа к файлу из контейнера multitool:

```bash
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl exec -it busybox-multitool-8d5948b97-n7hl5  -c multitool -- /bin/bash
bash-5.1# ls -la
total 76
drwxr-xr-x    1 root     root          4096 Sep  8 12:07 .
drwxr-xr-x    1 root     root          4096 Sep  8 12:07 ..
drwxr-xr-x    1 root     root          4096 Jan  5  2022 bin
drwx------    2 root     root          4096 Jan  5  2022 certs
drwxr-xr-x    5 root     root           360 Sep  8 12:07 dev
drwxr-xr-x    1 root     root          4096 Jan  5  2022 docker
drwxr-xr-x    1 root     root          4096 Sep  8 12:07 etc
drwxr-xr-x    2 root     root          4096 Nov 12  2021 home
drwxr-xr-x    1 root     root          4096 Jan  5  2022 lib
drwxr-xr-x    5 root     root          4096 Nov 12  2021 media
drwxr-xr-x    2 root     root          4096 Nov 12  2021 mnt
drwxr-xr-x    2 root     root          4096 Nov 12  2021 opt
dr-xr-xr-x  241 root     root             0 Sep  8 12:07 proc
drwx------    1 root     root          4096 Jan  5  2022 root
drwxr-xr-x    1 root     root          4096 Sep  8 12:07 run
drwxr-xr-x    1 root     root          4096 Jan  5  2022 sbin
drwxr-xr-x    2 root     root          4096 Sep  8 09:40 shared
drwxr-xr-x    2 root     root          4096 Nov 12  2021 srv
dr-xr-xr-x   13 root     root             0 Sep  8 12:07 sys
drwxrwxrwt    2 root     root          4096 Nov 12  2021 tmp
drwxr-xr-x    1 root     root          4096 Jan  5  2022 usr
drwxr-xr-x    1 root     root          4096 Jan  5  2022 var
bash-5.1# cd shared/
bash-5.1# ls -la
total 16
drwxr-xr-x    2 root     root          4096 Sep  8 09:40 .
drwxr-xr-x    1 root     root          4096 Sep  8 12:07 ..
-rw-r--r--    1 root     root          4988 Sep  8 12:09 data.txt
bash-5.1# 
bash-5.1# 
bash-5.1# cat data.txt 
Sun Sep  8 09:40:00 UTC 2024
Sun Sep  8 09:40:05 UTC 2024
Sun Sep  8 09:40:10 UTC 2024
Sun Sep  8 09:40:15 UTC 2024
Sun Sep  8 09:40:20 UTC 2024
```
Удаление pvc и deployment:
```yaml
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl delete -f deployment.yaml
deployment.apps "busybox-multitool" deleted
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl delete -f pvc.yaml 
persistentvolumeclaim "local-pvc" deleted
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM               STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv   1Gi        RWX            Retain           Released   default/local-pvc   local-storage   <unset>                          4m42s
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ cd 
ubuntu@fhmrlj60b9llbp4hne8n:~$ pwd
/home/ubuntu
ubuntu@fhmrlj60b9llbp4hne8n:~$ cd ..
ubuntu@fhmrlj60b9llbp4hne8n:/home$ ls -la
total 12
drwxr-xr-x  3 root   root   4096 Sep  8 09:23 .
drwxr-xr-x 19 root   root   4096 Sep  8 09:24 ..
drwxr-xr-x  6 ubuntu ubuntu 4096 Sep  8 12:06 ubuntu
ubuntu@fhmrlj60b9llbp4hne8n:/home$ cd 
ubuntu@fhmrlj60b9llbp4hne8n:~$ cd ../..
ubuntu@fhmrlj60b9llbp4hne8n:/$ pwd
/
ubuntu@fhmrlj60b9llbp4hne8n:/$ ls -la
total 72
drwxr-xr-x  19 root root  4096 Sep  8 09:24 .
drwxr-xr-x  19 root root  4096 Sep  8 09:24 ..
lrwxrwxrwx   1 root root     7 Dec 14  2022 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Aug 31 18:48 boot
drwxr-xr-x  15 root root  3800 Sep  8 09:23 dev
drwxr-xr-x  79 root root  4096 Sep  8 09:26 etc
drwxr-xr-x   3 root root  4096 Sep  8 09:23 home
lrwxrwxrwx   1 root root     7 Dec 14  2022 lib -> usr/lib
lrwxrwxrwx   1 root root     9 Dec 14  2022 lib32 -> usr/lib32
lrwxrwxrwx   1 root root     9 Dec 14  2022 lib64 -> usr/lib64
lrwxrwxrwx   1 root root    10 Dec 14  2022 libx32 -> usr/libx32
drwx------   2 root root 16384 Dec 14  2022 lost+found
drwxr-xr-x   3 root root  4096 Dec 14  2022 media
drwxr-xr-x   3 root root  4096 Sep  8 09:39 mnt
drwxr-xr-x   4 root root  4096 Sep  8 09:25 opt
dr-xr-xr-x 239 root root     0 Sep  8 09:22 proc
drwx------   5 root root  4096 Sep  8 09:53 root
drwxr-xr-x  24 root root   740 Sep  8 12:01 run
lrwxrwxrwx   1 root root     8 Dec 14  2022 sbin -> usr/sbin
drwxr-xr-x   6 root root  4096 Sep  8 09:25 snap
drwxr-xr-x   2 root root  4096 Jul 31  2020 srv
dr-xr-xr-x  13 root root     0 Sep  8 09:22 sys
drwxrwxrwt  10 root root  4096 Sep  8 12:12 tmp
drwxr-xr-x  14 root root  4096 Dec 14  2022 usr
drwxr-xr-x  12 root root  4096 Sep  8 09:24 var
ubuntu@fhmrlj60b9llbp4hne8n:/$ cd mnt/
ubuntu@fhmrlj60b9llbp4hne8n:/mnt$ ls -la
total 12
drwxr-xr-x  3 root root 4096 Sep  8 09:39 .
drwxr-xr-x 19 root root 4096 Sep  8 09:24 ..
drwxr-xr-x  2 root root 4096 Sep  8 09:40 data
ubuntu@fhmrlj60b9llbp4hne8n:/mnt$ cat data/
cat: data/: Is a directory
ubuntu@fhmrlj60b9llbp4hne8n:/mnt$ cd data/
ubuntu@fhmrlj60b9llbp4hne8n:/mnt/data$ ls -la
total 16
drwxr-xr-x 2 root root 4096 Sep  8 09:40 .
drwxr-xr-x 3 root root 4096 Sep  8 09:39 ..
-rw-r--r-- 1 root root 5481 Sep  8 12:11 data.txt
ubuntu@fhmrlj60b9llbp4hne8n:/mnt/data$ cat data.txt 
Sun Sep  8 09:40:00 UTC 2024
Sun Sep  8 09:40:05 UTC 2024
Sun Sep  8 09:40:10 UTC 2024
Sun Sep  8 09:40:15 UTC 2024
Sun Sep  8 09:40:20 UTC 2024
```
После удаления pvc, pv перешел в статус Released

Удаление pv и проверка доступности папки на хосте:
```bash
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl delete -f persistent_volume.yaml
persistentvolume "local-pv" deleted
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ 
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ kubectl get pv
No resources found
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ 
ubuntu@fhmrlj60b9llbp4hne8n:~/k8s$ cd ../..
ubuntu@fhmrlj60b9llbp4hne8n:/home$ ls 
ubuntu
ubuntu@fhmrlj60b9llbp4hne8n:/home$ cd ../..
ubuntu@fhmrlj60b9llbp4hne8n:/$ ls -la
total 72
drwxr-xr-x  19 root root  4096 Sep  8 09:24 .
drwxr-xr-x  19 root root  4096 Sep  8 09:24 ..
lrwxrwxrwx   1 root root     7 Dec 14  2022 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Aug 31 18:48 boot
drwxr-xr-x  15 root root  3800 Sep  8 09:23 dev
drwxr-xr-x  79 root root  4096 Sep  8 09:26 etc
drwxr-xr-x   3 root root  4096 Sep  8 09:23 home
lrwxrwxrwx   1 root root     7 Dec 14  2022 lib -> usr/lib
lrwxrwxrwx   1 root root     9 Dec 14  2022 lib32 -> usr/lib32
lrwxrwxrwx   1 root root     9 Dec 14  2022 lib64 -> usr/lib64
lrwxrwxrwx   1 root root    10 Dec 14  2022 libx32 -> usr/libx32
drwx------   2 root root 16384 Dec 14  2022 lost+found
drwxr-xr-x   3 root root  4096 Dec 14  2022 media
drwxr-xr-x   3 root root  4096 Sep  8 09:39 mnt
drwxr-xr-x   4 root root  4096 Sep  8 09:25 opt
dr-xr-xr-x 239 root root     0 Sep  8 09:22 proc
drwx------   5 root root  4096 Sep  8 09:53 root
drwxr-xr-x  24 root root   740 Sep  8 12:01 run
lrwxrwxrwx   1 root root     8 Dec 14  2022 sbin -> usr/sbin
drwxr-xr-x   6 root root  4096 Sep  8 09:25 snap
drwxr-xr-x   2 root root  4096 Jul 31  2020 srv
dr-xr-xr-x  13 root root     0 Sep  8 09:22 sys
drwxrwxrwt  10 root root  4096 Sep  8 12:14 tmp
drwxr-xr-x  14 root root  4096 Dec 14  2022 usr
drwxr-xr-x  12 root root  4096 Sep  8 09:24 var
ubuntu@fhmrlj60b9llbp4hne8n:/$ cd mnt/data/
ubuntu@fhmrlj60b9llbp4hne8n:/mnt/data$ ls -la 
total 16
drwxr-xr-x 2 root root 4096 Sep  8 09:40 .
drwxr-xr-x 3 root root 4096 Sep  8 09:39 ..
-rw-r--r-- 1 root root 5481 Sep  8 12:11 data.txt
ubuntu@fhmrlj60b9llbp4hne8n:/mnt/data$ cat data.txt 
Sun Sep  8 09:40:00 UTC 2024
Sun Sep  8 09:40:05 UTC 2024
Sun Sep  8 09:40:10 UTC 2024
Sun Sep  8 09:40:15 UTC 2024
```

После удаления PV файл на ноде останется, так как политика Reclaim Policy была установлена в Retain, что оставляет данные на месте после удаления PV.

------

### Задание 2

**Что нужно сделать**

Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

1. Включить и настроить NFS-сервер на MicroK8S.
2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.
3. Продемонстрировать возможность чтения и записи файла изнутри пода. 
4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

### Ответ 
Установка NFS-сервер на узле MicroK8S (нода):
```bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
```

Создадим директорию для экспорта:
```bash
sudo mkdir -p /srv/nfs/kubedata
sudo chown nobody:nogroup /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata
```
Экспорт для NFS:
```bash
sudo nano /etc/exports
```

Добавим строку:
/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)

Применим изменения экспортов:
```bash
sudo exportfs -rav
```

Запустим и проверим сервис NFS:
```bash
sudo systemctl restart nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

Создаем StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: example.com/nfs
parameters:
  archiveOnDelete: "false"

```

Создаем Persistent Volume Claim (PVC):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Deployment приложения multitool

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
        - name: multitool
          image: praqma/network-multitool
          command: ["/bin/sh"]
          args:
            - -c
            - >
              while true; do
                echo "$(date)" >> /mnt/data/data.txt;
                sleep 5;
              done
          volumeMounts:
            - name: nfs-storage
              mountPath: /mnt/data
      volumes:
        - name: nfs-storage
          persistentVolumeClaim:
            claimName: nfs-pvc
```
------
