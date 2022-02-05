### Mateusz Kuleta

### Źródła

   ```bash
   https://www.youtube.com/watch?v=2m9APyQqzFU 
   https://www.youtube.com/watch?v=VFpEyWQ9FeA 
   https://www.youtube.com/watch?v=jSz3_6j9-bw
   ```

# Aplikacja "Hello World!" w chmurze Azure przy użyciu Docker oraz Terraform

Github

   ```bash
   git clone https://github.com/matejiPLnd/Projek-Docker-TF.git
   ``` 

## Stacja robocza

1. Rekomendowany: Linux (Ubuntu) / MacOS
2. Zainstalowane:

   - azure-cli ([instalacje](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli))
   - Docker
   
    ```bash
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io
    ```
    
   - kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```

   - terraform (https://learn.hashicorp.com/tutorials/terraform/install-cli)

## Praca z użyciem maszyny wirtualnej Azure poprzez ssh


1. Utwórz swój klucz ssh:

   ```bash
   ssh-keygen -f ~/.ssh/wsb_id_rsa -t rsa -b 4092
   ls ~/.ssh
   ```

2. Utwórz maszynę wirtualną dodając nasz klucz ssh (patrz ostatnie ćwiczenia [01_exercise](../01_exercise/manual.md)): 

   ```bash
   az vm create \
     --resource-group <nazwa-grupy-zasobów> \
     --name <nazwa-maszyny> \
     --size "Standard_B1ls" \
     --image "Canonical:0001-com-ubuntu-server-focal:20_04-lts-gen2:latest"  \
     --public-ip-sku Standard \
     --admin-username ubuntu \
     --ssh-key-value ~/.ssh/wsb_id_rsa.pub
   ```

3. zaloguj się z uzyciem klucza ssh:

   ```bash
   ssh ubuntu@<IP_ADDRESS>

   # jak debugować?
   ssh ubuntu@<IP_ADDRESS>  -vvv

   # a teraz ze wskazaniem klucza,
   # który chcemy uzyc
   ssh ubuntu@<IP_ADDRESS> -i ~/.ssh/wsb_id_rsa
   ```


## Tworzenie obrazu Dockera

1. Będąc w folderze naszego projektu tworzymy obraz Dockera
   
   ```bash
   sudo docker build .
   ```

2. Po poprawnym utworzeniu obrazu wyświetli się komunikat:
   
   ```bash
   Successfully built <IMAGE ID>
   ```
   
   Listę utworzonych obrazów możemy sprawdzić
   
   ```bash
   sudo docker images
   ```
   
   ```bash
   REPOSITORY                       TAG               IMAGE ID       CREATED          SIZE
   <none>                           <none>            bff02e38c885   12 seconds ago   300MB
   mateuszkuletaaksacr.azurecr.io   latest            749fefffce8b   24 minutes ago   300MB
   python                           3.8-slim-buster   df39427ca45b   6 days ago       114MB
   ```
3. Tworzymy tag do utworzonego obrazu

   ```bash
   sudo docker tag <IMAGE ID> <TAG>
   
   sudo docker tag bff02e38c885 mateuszkuletaaksacr.azurecr.io/myapp:latest
   ```
   
   ```bash
   REPOSITORY                       TAG               IMAGE ID       CREATED          SIZE
   mateuszkuletaaksacr.azurecr.io/myapp   latest      bff02e38c885   24 minutes ago   300MB
   mateuszkuletaaksacr.azurecr.io   latest            749fefffce8b   24 minutes ago   300MB
   ```
## Logowanie do Azure


1. Logujemy się do Azure

   ```bash
   az login
   ```

   Powinno otworzyć się okno przeglądarki w którym logujemy się do Azure Portal.
   
2. Inicjujemy terraforma
   
   ```bash
   terraform init
   ```
   
   ```bash
   Terraform has been successfully initialized!
   ```
   
3. Tworzymy plan

    ```bash
   terraform plan
   ```

4. Komendą apply wykonujemy instrukcje proponowane w planie Terraforma

   ```bash
   terraform apply
   ```

5. Zostaniemy potwierdzenie wykonania planu, wpisujemy `yes`.


5. Program utworzył nam grupę zasobów i klaster.

## Zaloguj się do interfejsu Azure Container Registry

1. Login:

   ```bash
   sudo az acr login -n <NAZWA KLASTRA>
   ```

2. Robimy `push` do repozytorium

   ```bash
   sudo docker push mateuszkuletaaksacr.azurecr.io/myapp
   ```

3. Uzyskiwanie dostępu dla zarządzanego klastra

   ```bash
   az aks get-credentials --name mateuszkuleta-aks --resource-group mateuszkuleta_aks_tf_rg
   ```

## Wdrażanie aplikacji

   ### Uwaga!
   W pliku YAML musimy zmienić nazwę naszego obrazu
   
   ```bash
   image: <NAZWA NASZEGO OBRAZU>
   ```
   
1. Wdrażamy poprzez `kubect apply`

   ```bash
   kubectl apply -f mateuszdeploy.yaml
   ```

   Zakończone powodzeniem:

   ```bash
   mateji@ubuntu:~/Desktop/python-docker$ kubectl get all
   NAME                                       READY   STATUS    RESTARTS   AGE
   pod/my-python-deployment-dc7cfcf49-l4lgb   1/1     Running   0          35s
   pod/my-python-deployment-dc7cfcf49-xp8pq   1/1     Running   0          25s

   NAME                        TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)          AGE
   service/kubernetes          ClusterIP      10.0.0.1     <none>           443/TCP          29m
   service/my-python-app-svc   LoadBalancer   10.0.5.14    20.103.101.152   5000:32225/TCP   4m38s

   NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/my-python-deployment   2/2     2            2           4m38s

   NAME                                             DESIRED   CURRENT   READY   AGE
   replicaset.apps/my-python-deployment-cd7cbd84f   0         0         0       4m38s
   replicaset.apps/my-python-deployment-dc7cfcf49   2         2         2       35s
   ```
   Zakończone niepowodzeniem:
   
   ```bash
   mateji@ubuntu:~/Desktop/python-docker$ kubectl get all
   NAME                                       READY   STATUS         RESTARTS   AGE
   pod/my-python-deployment-cd7cbd84f-hq78l   0/1     ErrImagePull   0          58s
   pod/my-python-deployment-cd7cbd84f-whx4r   0/1     ErrImagePull   0          58s

   NAME                        TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)          AGE
   service/kubernetes          ClusterIP      10.0.0.1     <none>           443/TCP          25m
   service/my-python-app-svc   LoadBalancer   10.0.5.14    20.103.101.152   5000:32225/TCP   58s

   NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/my-python-deployment   0/2     2            0           58s

   NAME                                             DESIRED   CURRENT   READY   AGE
   ```

2. Uruchamianie aplikacji poprzez `curl`

   ```bash
   curl <ADRES IP>
   
   curl 20.103.101.152:5000
   ```

