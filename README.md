# Домашнее задание к занятию «Хранение в K8s»

## ` Дмитрий Климмов `

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
































