# Syncing to a read-only repo

1. Создать кластер

2. Подключится к кластеру 

   ```
   gcloud container clusters get-credentials [MY-CLUSTER]
   ```

3. Создать роль кластер админа 

```
kubectl create clusterrolebinding cluster-admin-binding  --clusterrole cluster-admin --user [USER_ACCOUNT_EMAIL] 
```

4. Установить nomos 

   1. Скачать
      `gsutil cp gs://config-management-release/released/latest/linux_amd64/nomos nomos`
   2. Дать права
      `chmod +x /path/to/nomos`

5. Установить Config Sync Operator

   1. Скачать
      `gsutil cp gs://config-management-release/released/latest/config-sync-operator.yaml config-sync-operator.yaml`
   2. Поднять инфраструктуру 
      `kubectl apply -f config-sync-operator.yaml`

6. Создать файл  **config-management.yaml** и добавить в него код:

   ```yaml
   apiVersion: configmanagement.gke.io/v1
   kind: ConfigManagement
   metadata:
     name: config-management
   spec:
     # clusterName is required and must be unique among all managed clusters
     clusterName: my-cluster
     git:
       syncRepo: https://github.com/GoogleCloudPlatform/csp-config-management/
       syncBranch: 1.0.0
       secretType: none
       policyDir: "foo-corp"
   ```

7. Принять конфигурацию 

   ```
   kubectl apply -f config-management.yaml
   ```

8. Проверить наличие поднятых под
   `kubectl get pods -n config-management-system`
   Вывод:

   ```
   NAME                                   READY     STATUS    RESTARTS   AGE
   git-importer-5f8bdb59bd-7nn5m          2/2       Running   0          2m
   monitor-58c48fbc66-ggrmd               1/1       Running   0          2m
   syncer-7bbfd7686b-dxb45                1/1       Running   0          2m
   ```

9. Посмотреть список namespaces которые менеджит Config Sync:
   `kubectl get ns -l app.kubernetes.io/managed-by=configmanagement.gke.io`
   Вывод:

   ```
   NAME               STATUS   AGE
   audit              Active   4m
   shipping-dev       Active   4m
   shipping-prod      Active   4m
   shipping-staging   Active   4m
   ```

10. Просмотреть роли кластера
    `kubectl get clusterroles -l app.kubernetes.io/managed-by=configmanagement.gke.io`
    Вывод:

    ```
    NAME               AGE
    namespace-reader   6m52s
    pod-creator        6m52s
    ```

11. Добавить в  свой репозиторий папки и файлы из проекта  [foo-corp repo](https://github.com/GoogleCloudPlatform/csp-config-management/) 
    `foo-corp
    ci-pipeline
    ci-pipeline-unstructured
    crds
    CODEOWNERS`
    Установить путь к папке с конфигами( путь к папке должен быть полный)
     `./nomos vet --path=/path/to/repo/foo-corp`

12.  Дать права файлу
    `chmod +x .git/hooks/pre-commit.sample`

13. Cгенерировать гит-ключи

    ```
    kubectl create secret generic git-creds --namespace=config-management-system  --from-file=ssh=/home/user/.ssh/id_rsa
    ```

14. Отредактировать **config-management.yaml** 

    1. Поменять  **syncRepo** на свой
    2. Поменять **secretType** на ssh
    3.  Поменять  **syncBranch** на ветку в своем репозитории

    ```yaml
    apiVersion: configmanagement.gke.io/v1
    kind: ConfigManagement
    metadata:
      name: config-management
    spec:
      # clusterName is required and must be unique among all managed clusters
      clusterName: my-cluster
      git:
        syncRepo: git@github.com:my-github-username/csp-config-management.git
        syncBranch: master
        secretType: ssh
        policyDir: "foo-corp"
    ```



15.  Подтвердить изменения
    `kubectl apply -f config-management.yaml`

16. Проверить поды
    `kubectl get pods -n config-management-system`
    Вывод:

    ```
    NAME                                   READY     STATUS    RESTARTS   AGE
    git-importer-5f8bdb59bd-7nn5m          2/2       Running   0          2m
    monitor-58c48fbc66-ggrmd               1/1       Running   0          2m
    syncer-7bbfd7686b-dxb45                1/1       Running   0          2m
    ```

17.  Открыть дополнительное окно терминала. В первом, рабочем, ввести команду
    `kubectl get clusterrolebindings namespace-readers -o yaml --watch`

    Чтобы убедиться, что изменения прошли,  во втором терминале нужно открыть файл **cluster/namespace-reader-clusterrolebinding.yaml** и изменить его к примеру вот так

    ```yaml
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: namespace-readers
    subjects:
    - kind: User
      name: cheryl@foo-corp.com
      apiGroup: rbac.authorization.k8s.io
    - kind: User
      name: jane@foo-corp.com
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: namespace-reader
      apiGroup: rbac.authorization.k8s.io
    ```

Сохранить файл. Закомитить изменения в репозиторий.


```
git add cluster/namespace-reader-clusterrolebinding.yaml

git commit -m "Add Jane to namespace-reader"

git push origin master
```

Если все было настроено правильно, то в первом терминале выведет обновленную информацию(нужно немного подождать)

#                                         CI Pipeline

Поднять инфраструктуру

```
gsutil cp gs://config-management-release/released/latest/config-management-operator.yaml config-management-operator.yaml
kubectl apply -f config-management-operator.yaml
```

1.  В  файл **config-management.yaml** внести изменения, вставить код в  самом начале

   ```yaml
   apiVersion: configmanagement.gke.io/v1
   kind: ConfigManagement
   metadata:
     name: config-management
   spec:
     # Set to true to install and enable Policy Controller
     policyController:
       enabled: true
       # Uncomment to prevent the template library from being installed
       # templateLibraryInstalled: false
       # Uncomment to disable audit, adjust value to set audit interval
       # auditIntervalSeconds: 0
       # Uncomment to log all denies and dryrun failures
       # logDeniesEnabled: true
       
         # ...other fields...
   ```

2.   Подтвердить изменения 
   `kubectl apply -f config-management.yaml`

3. Проверить поднятые поды
   `kubectl get pods -n gatekeeper-system`
   Вывод:

   ```
   NAME                              READY   STATUS    RESTARTS   AGE
   gatekeeper-controller-manager-0   1/1     Running   1          53s
   ```

4. Просмотреть ограничения
   `kubectl get constrainttemplates`
   Вывод: 

   ```
   NAME                                      AGE
   k8sallowedrepos                           84s
   k8scontainerlimits                        84s
   k8spspallowprivilegeescalationcontainer   84s
   ...[OUTPUT TRUNCATED]...
   ```

5.  В облаке в разделе **Cloud Build** зайти в **Triggers**
6.  Cоздать триггер с **Event**  **Push to a branch**.
7.  Выбрать репозиторий, законнектить с ним триггер, ветку выбрать **master**
8.  Выбрать в качестве **Cloud Build Configuration File** ci-pipeline/cloudbuild.yaml
9. В  файле CODEOWNERS заменить **@OWNER** на гитхаб юзера 
   Пример:![image-20200907134612779](/home/anhelina/.config/Typora/typora-user-images/image-20200907134612779.png)

10.  В файле **ci-pipeline/cloudbuild.yaml** поменять  **CLUSTER_NAME** и  **ZONE** на текущие

11.  Закоммитить изменения

12. Открыть   [Cloud Build history](https://console.cloud.google.com/cloud-build/builds) и увидеть ошибку

13.  Редактировать **/ci-pipeline/config-root/namespaces/online/shipping-app-backend/shipping-prod/namespace.yaml**

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: shipping-prod
      labels:
        env: prod
        cost-center: "shipping.foo-corp.com"
      annotations:
        audit: "true"
    ```

14. Закоммитить изменения
