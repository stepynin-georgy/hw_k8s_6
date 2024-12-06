# Домашнее задание к занятию «Хранение в K8s. Часть 1»

### Цель задания

В тестовой среде Kubernetes нужно обеспечить обмен файлами между контейнерам пода и доступ к логам ноды.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке MicroK8S](https://microk8s.io/docs/getting-started).
2. [Описание Volumes](https://kubernetes.io/docs/concepts/storage/volumes/).
3. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1 

**Что нужно сделать**

Создать Deployment приложения, состоящего из двух контейнеров и обменивающихся данными.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-hw1
spec:
  selector:
    matchLabels:
      app: volume-hw1
  replicas: 1
  template:
    metadata:
      labels:
        app: volume-hw1
    spec:
      containers:
      - name: busybox
        image: busybox:1.28
        command: ['sh', '-c', 'mkdir -p /volumes && while true; do echo "$(date) - Test message" >> /volumes/success.txt; sleep 5; done']
        volumeMounts:
        - name: volume
          mountPath: /volumes
      - name: multitool
        image: wbitt/network-multitool
        command: ['sh', '-c', 'tail -f /volumes']
        volumeMounts:
        - name: volume
          mountPath: /volumes
      volumes:
      - name: volume
        emptyDir: {}
```

```
user@k8s:/opt/hw_k8s_6$ microk8s kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
volume-hw1   1/1     1            1           5s
user@k8s:/opt/hw_k8s_6$ microk8s kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
volume-hw1-565947f95c-zwbfr   2/2     Running   0          10s
```

2. Сделать так, чтобы busybox писал каждые пять секунд в некий файл в общей директории.
3. Обеспечить возможность чтения файла контейнером multitool.

```
user@k8s:/opt/hw_k8s_6$ microk8s kubectl logs volume-hw1-7b8bcdd455-q6hvq multitool
Fri Dec  6 13:10:09 UTC 2024 - Test message
Fri Dec  6 13:10:14 UTC 2024 - Test message
Fri Dec  6 13:10:19 UTC 2024 - Test message
```

4. Продемонстрировать, что multitool может читать файл, который периодоически обновляется.

```
user@k8s:/opt/hw_k8s_6$ microk8s kubectl exec -ti volume-hw1-565947f95c-zwbfr -c busybox -- sh
/ # ls -l /volumes/success.txt
-rw-r--r--    1 root     root          1232 Dec  6 13:06 /volumes/success.txt
/ # cat /volumes/success.txt
Fri Dec  6 13:04:19 UTC 2024 - Test message
Fri Dec  6 13:04:24 UTC 2024 - Test message
Fri Dec  6 13:04:29 UTC 2024 - Test message
Fri Dec  6 13:04:34 UTC 2024 - Test message
Fri Dec  6 13:04:39 UTC 2024 - Test message
Fri Dec  6 13:04:44 UTC 2024 - Test message
Fri Dec  6 13:04:49 UTC 2024 - Test message
Fri Dec  6 13:04:54 UTC 2024 - Test message
```

```
user@k8s:/opt/hw_k8s_6$ microk8s kubectl exec -ti volume-hw1-565947f95c-zwbfr -c multitool -- sh
/ # ls -l /volumes/success.txt
-rw-r--r--    1 root     root          1892 Dec  6 13:07 /volumes/success.txt
/ # cat /volumes/success.txt
Fri Dec  6 13:04:19 UTC 2024 - Test message
Fri Dec  6 13:04:24 UTC 2024 - Test message
Fri Dec  6 13:04:29 UTC 2024 - Test message
Fri Dec  6 13:04:34 UTC 2024 - Test message
Fri Dec  6 13:04:39 UTC 2024 - Test message
Fri Dec  6 13:04:44 UTC 2024 - Test message
Fri Dec  6 13:04:49 UTC 2024 - Test message
Fri Dec  6 13:04:54 UTC 2024 - Test message
```

5. Предоставить манифесты Deployment в решении, а также скриншоты или вывод команды из п. 4.

[deployment.yml](https://github.com/stepynin-georgy/hw_k8s_6/blob/main/deployment.yml)

------

### Задание 2

**Что нужно сделать**

Создать DaemonSet приложения, которое может прочитать логи ноды.

1. Создать DaemonSet приложения, состоящего из multitool.

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset
  labels:
    app: multitool
spec:
  selector:
    matchLabels:
      name: daemonset
  template:
    metadata:
      labels:
        name: daemonset
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: logdir
          mountPath: /nodes-logs/syslog
          subPath: syslog
        - name: varlog
          mountPath: /var/log/syslog
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: logdir
        hostPath:
          path: /var/log
      - name: varlog
        hostPath:
          path: /var/log
```

```
user@k8s:/opt/hw_k8s_6$ microk8s kubectl apply -f daemonset.yml
daemonset.apps/daemonset created
user@k8s:/opt/hw_k8s_6$ microk8s kubectl get daemonsets
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset   1         1         1       1            1           <none>          25s
user@k8s:/opt/hw_k8s_6$ microk8s kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
daemonset-n8rtz   1/1     Running   0          28s
```

2. Обеспечить возможность чтения файла `/var/log/syslog` кластера MicroK8S.
3. Продемонстрировать возможность чтения файла изнутри пода.

```
user@k8s:/opt/hw_k8s_6$ microk8s kubectl exec -ti daemonset-n8rtz -- sh
/ # cd nodes-logs/
/nodes-logs # ls -l
total 5376
-rw-r-----    1 103      adm        5503889 Dec  6 13:21 syslog
/nodes-logs # tail -f syslog
2024-12-06T13:20:19.389171+00:00 k8s systemd[1]: Cannot find unit for notify message of PID 303832, ignoring.
2024-12-06T13:20:19.682235+00:00 k8s systemd[1]: run-containerd-runc-k8s.io-d7d4329a02b83721db01aa7baf76fe63d7dfdca99d7731c33b908b6f1aba95ea-runc.zgc3do.mount: Deactivated successfully.
2024-12-06T13:20:21.145768+00:00 k8s systemd[1]: run-containerd-runc-k8s.io-5a0e495b4177865e19c4f3b2fffe0c5b6d6283e1c1b78fe5eb3329883441cf46-runc.z5kClg.mount: Deactivated successfully.
2024-12-06T13:20:43.092990+00:00 k8s systemd[2032]: Started snap.microk8s.microk8s-e3123ade-e45e-441d-9c04-a85ff07a76b9.scope.
2024-12-06T13:20:43.304871+00:00 k8s systemd[2032]: Started snap.microk8s.microk8s-1d55fdd8-154f-4386-884b-52c1e863d874.scope.
2024-12-06T13:20:55.554659+00:00 k8s systemd[1]: Cannot find unit for notify message of PID 305024, ignoring.
2024-12-06T13:20:55.722135+00:00 k8s systemd[1]: Cannot find unit for notify message of PID 305099, ignoring.
2024-12-06T13:20:57.629359+00:00 k8s systemd[2032]: Started snap.microk8s.microk8s-5e493322-c462-4d4b-816a-d8408a441161.scope.
2024-12-06T13:21:02.123051+00:00 k8s systemd[2032]: Started snap.microk8s.microk8s-98489a81-39e4-4ddc-8932-465b457bf1af.scope.
2024-12-06T13:21:09.684844+00:00 k8s systemd[1]: run-containerd-runc-k8s.io-d7d4329a02b83721db01aa7baf76fe63d7dfdca99d7731c33b908b6f1aba95ea-runc.dOO2Gm.mount: Deactivated successfully.
```

4. Предоставить манифесты Deployment, а также скриншоты или вывод команды из п. 2.

[daemonset.yml](https://github.com/stepynin-georgy/hw_k8s_6/blob/main/daemonset.yml)

------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------
