# Домашнее задание к занятию «Хранение в K8s»

## ` Дмитрий Климов `

## Задание 1. Volume: обмен данными между контейнерами в поде

### Задача
### Создать Deployment приложения, состоящего из двух контейнеров, обменивающихся данными.

### Шаги выполнения
   1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
   2. Настроить busybox на запись данных каждые 5 секунд в некий файл в общей директории.
   3. Обеспечить возможность чтения файла контейнером multitool.

### Что сдать на проверку

  * Манифесты:
     *containers-data-exchange.yaml
  * Скриншоты:
    * описание пода с контейнерами (kubectl describe pods data-exchange)
    * вывод команды чтения файла (tail -f <имя общего файла>)

## Ответ:

---

### Манифест: `containers-data-exchange.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange
  template:
    metadata:
      labels:
        app: data-exchange
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'while true; do date >> /shared/data.txt; sleep 5; done']
        volumeMounts:
        - name: shared-storage
          mountPath: /shared
      - name: multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        volumeMounts:
        - name: shared-storage
          mountPath: /shared
      volumes:
      - name: shared-storage
        emptyDir: {}
```
 ### Контейнер 2: `multitool` для чтения данных
 ```yaml
  - name: multitool
    image: wbitt/network-multitool
    env:
    - name: HTTP_PORT
      value: "8080"
    volumeMounts:
    - name: shared-storage
      mountPath: /shared
  volumes:
  - name: shared-storage
    emptyDir: {}
```
---

### Шаги выполнения и проверки

1.  ### **Примените манифест:**
    ```bash
    microk8s kubectl apply -f containers-data-exchange.yaml
    ```

2.  ### **Запуск пода и получение его актуальное имя:**
    ```bash
    microk8s kubectl get pods
    # Текущий под: data-exchange-697f4448db-4xghf
    # В следующих командах используем это имя.
    ```

3.  ### Эта команда покажет подробную информацию о поде, включая список контейнеров и                       смонтированный `emptyDir` volume.
    ```bash
    microk8s kubectl describe pod data-exchange-697f4448db-4xghf
    ```
<img width="1920" height="1080" alt="Снимок экрана (2798)" src="https://github.com/user-attachments/assets/6cc6230a-1d51-4e23-8bc0-18bde6139065" />

<img width="1920" height="1080" alt="Снимок экрана (2799)" src="https://github.com/user-attachments/assets/632601c8-667d-4da0-92ad-0f7b8a69db55" />


4.  ### Подключение к контейнеру `multitool` и прочитаем общий файл, в который `busybox`                  записывает данные.
    ```bash
    microk8s kubectl exec data-exchange-697f4448db-4xghf -c multitool -- tail -f /shared/data.txt
    ```
<img width="1920" height="1080" alt="Снимок экрана (2797)" src="https://github.com/user-attachments/assets/81648a98-a6b3-4c66-a544-0c2edafac57f" />

---

## Задание 2. PV, PVC

### Задача

### Создать Deployment приложения, использующего локальный PV, созданный вручную.

### Шаги выполнения

  1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool, использующего созданный ранее PVC
  2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
  3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который             busybox записывает данные каждые 5 секунд.
  4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему. (Используйте команду        kubectl describe pv).
  5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV. Продемонстрировать, что произошло с            файлом после удаления PV. Пояснить, почему.

### Что сдать на проверку
   * Манифесты:
      * pv-pvc.yaml
   * Скриншоты:
      * каждый шаг выполнения задания, начиная с шага 2.
   * Описания:
      * объяснение наблюдаемого поведения ресурсов в двух последних шагах.

## Ответ:

## 1. Манифест ` pv-pvc.yaml `

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/pv-shared"
  storageClassName: manual

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-local
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-test
  template:
    metadata:
      labels:
        app: storage-test
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ['sh', '-c', 'while true; do date >> /data/shared_data.txt; sleep 5; done']
          volumeMounts:
            - name: my-persistent-storage
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          env:
            - name: HTTP_PORT
              value: "8080"
          volumeMounts:
            - name: my-persistent-storage
              mountPath: /data
      volumes:
        - name: my-persistent-storage
          persistentVolumeClaim:
            claimName: pvc-local
```

## 2. Описание и объяснение поведения ресурсов

### Cоздания директории и включения ` hostpath-storage `

<img width="1920" height="1080" alt="Снимок экрана (2800)" src="https://github.com/user-attachments/assets/cb61bacd-a55c-40c3-a242-e995073d4c18" />

### Применения манифеста `pv-pvc.yaml `

<img width="1920" height="1080" alt="Снимок экрана (2801)" src="https://github.com/user-attachments/assets/9477a8d8-5357-41bd-bb39-b1693cd6107c" />

### Выводы kubectl `get pv,pvc` и `kubectl get pods`

<img width="1920" height="1080" alt="Снимок экрана (2803)" src="https://github.com/user-attachments/assets/11eec6b7-b528-4e9d-8198-d86ae4d5b78a" />

### Вывод `tail -f` из контейнера `multitool`

<img width="1920" height="1080" alt="Снимок экрана (2804)" src="https://github.com/user-attachments/assets/6ceb42cc-4ddd-4fb5-ba41-a79c018245b6" />

### Удаления `Deployment` и `PVC`, `kubectl describe pv`

<img width="1920" height="1080" alt="Снимок экрана (2805)" src="https://github.com/user-attachments/assets/6c84715f-1463-4137-a1b9-c899222a37d3" />

### `cat` файла на локальном диске ноды

<img width="1920" height="1080" alt="Снимок экрана (2806)" src="https://github.com/user-attachments/assets/322911e1-1b8f-4589-9743-5629508b778b" />

### удаления `PV` и финальной проверки наличия файла на диске

<img width="1920" height="1080" alt="Снимок экрана (2807)" src="https://github.com/user-attachments/assets/2285d307-d290-4510-a448-00d49e269139" />

### Удаление `Deployment и PVC`. Состояние `PV`.

```
Наблюдаемое поведение:
После удаления Deployment и PVC команда kubectl get pv показывает, что статус pv-local изменился с Bound на Released.

Пояснение:
Статус Released означает, что запрос (PVC), который удерживал этот объем, был удален. Поскольку в манифесте PV была указана политика persistentVolumeReclaimPolicy: Retain, Kubernetes не удаляет сам объем и данные в нем. Однако PV не возвращается автоматически в статус Available, так как он все еще содержит данные предыдущего пользователя (PVC) и требует ручного вмешательства администратора для повторного использования.
```

### Сохранность файла на локальном диске ноды.

```
Наблюдаемое поведение:
При проверке директории на хост-системе (/data/pv-shared/shared_data.txt) файл остается на месте и содержит все записанные ранее данные.

Пояснение:
Это результат работы политики Retain. Она гарантирует, что физические данные на ноде сохраняются независимо от жизненного цикла объектов PVC в кластере.
```

### Удаление PV и состояние файла.

```
Наблюдаемое поведение:
После удаления объекта PV с помощью kubectl delete pv pv-local, файл на локальном диске по адресу /data/pv-shared/shared_data.txt по-прежнему существует.

Пояснение:
Для типов томов hostPath Kubernetes управляет только объектом в API (самой записью о PV). При удалении ресурса PV из кластера, Kubernetes удаляет только "логическое" описание тома. Удаление фактических данных в локальной директории на ноде не производится. Это механизм защиты от случайной потери данных, требующий от администратора системы ручной очистки дискового пространства на физическом уровне.
```

## Задание 3. StorageClass

### Задача

#### Создать `Deployment` приложения, использующего `PVC`, созданный на основе `StorageClass`.

### Шаги выполнения

1. Создать `Deployment` приложения, состоящего из контейнеров `busybox` и `multitool`, использующего созданный ранее `PVC`.
2. Создать `SC` и `PVC` для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что контейнер multitool может читать данные из файла в смонтированной директории, в который busybox     записывает данные каждые 5 секунд.

### Что сдать на проверку

  * Манифесты:
     * sc.yaml
  * Скриншоты:
     * каждый шаг выполнения задания, начиная с шага 2
   
## Ответ:

## 1. Манифест `sc.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
name: local-path-sc
provisioner: microk8s.io/hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: sc-pvc
spec:
storageClassName: local-path-sc
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 500Mi

apiVersion: apps/v1
kind: Deployment
metadata:
name: sc-deployment
spec:
replicas: 1
selector:
matchLabels:
app: sc-test
template:
metadata:
labels:
app: sc-test
spec:
containers:
- name: busybox
image: busybox
command: ['sh', '-c', 'while true; do date >> /data/dynamic_data.txt; sleep 5; done']
volumeMounts:
- name: dynamic-storage
mountPath: /data
- name: multitool
image: wbitt/network-multitool
env:
- name: HTTP_PORT
value: "8080"
volumeMounts:
- name: dynamic-storage
mountPath: /data
volumes:
- name: dynamic-storage
persistentVolumeClaim:
claimName: sc-pvc
```
## 2. Описание процесса и результаты

В данном задании был реализован механизм динамического выделения ресурсов:
1. Создан **StorageClass**, использующий провиженер `microk8s.io/hostpath`.
2. Создан **PVC**, который автоматически инициировал создание соответствующего **PV** (статус `Bound` подтверждает успешную связку).
3. Развернут **Deployment** с двумя контейнерами, которые используют общий динамический том для обмена данными.

## 3. Скриншоты

### 1. Применение манифеста `sc.yaml`.

<img width="1920" height="1080" alt="Снимок экрана (2808)" src="https://github.com/user-attachments/assets/7f0761f3-b5dd-4167-b215-6e0df3ebd2e4" />

### 2. Вывод `kubectl get sc,pvc,pv`, демонстрирующий автоматическое создание PV.

<img width="1920" height="1080" alt="Снимок экрана (2809)" src="https://github.com/user-attachments/assets/65cbd2ce-9efd-4f28-9037-32286c0d0b7c" />

### 3. Вывод `tail -f /data/dynamic_data.txt` из контейнера `multitool`, подтверждающий запись и чтение данных.

<img width="1920" height="1080" alt="Снимок экрана (2811)" src="https://github.com/user-attachments/assets/0f756d6d-d425-4cfb-b617-f272e22ef031" />





















