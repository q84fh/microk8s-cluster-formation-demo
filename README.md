# Microk8s: Jak zbudować wysokodostępny klaster Kubernetes w kilka minut

## Co jest potrzebne
 - 3 x maszyny z linuksem (użyłem Ubuntu 22.04 LTS (4GB RAM, 25GB HDD, 2 CPU))

## Instalacja
Informacje o dostępnych wersjach:
```bash
snap info microk8s
```

`microk8s` instalujemy poleceniem:
```
sudo snap install microk8s --classic --channel=1.29/stable
sudo microk8s status --wait-ready
```

Dla wygody dodajmy się do grupy `microk8s`, żeby nie musieć za każdym razem wołać `sudo` i dodajmy alias, żeby móc napisać `kubectl` zamiast `microk8s.kubectl`:
```
sudo usermod -a -G microk8s q84fh
sudo chown -f -R q84fh ~/.kube
echo "alias kubectl='microk8s.kubectl'" >> ~/.bash_aliases
. ~/.bash_aliases
```

## Formowanie klastra
Na jednym (dowolnym) z nodów wydajemy polecenie generujące polecenie z tokenem potrzebnym do połączenia w klaster:
```
q84fh@kube1:~$ microk8s add-node
From the node you wish to join to this cluster, run the following:
microk8s join 172.16.32.201:25000/ee2ce19b7596134132272e482c51e7da/0db4af6dff3d

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 172.16.32.201:25000/ee2ce19b7596134132272e482c51e7da/0db4af6dff3d --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 172.16.32.201:25000/ee2ce19b7596134132272e482c51e7da/0db4af6dff3d
```

na drugim nodzie po prostu wklejamy wygenerowane polecenie:
```
q84fh@kube2:~$ microk8s join 172.16.32.201:25000/ee2ce19b7596134132272e482c51e7da/0db4af6dff3d
Contacting cluster at 172.16.32.201
Waiting for this node to finish joining the cluster. .. .. .. ..  
Successfully joined the cluster.
```

Powtarzamy tą procedurę dla każdego noda (nie trzeba czekać aż się zakończy, za to za każdym razem trzeba pozyskać nowy token poleceniem `microk8s add-node`).

Formowanie klastra trwa chwilę, stan możem obserwować poleceniem:
```
kubectl get nodes --watch
```
Możemy sprawdzić czy osiągnęliśmy wysoką dostępność poleceniem:
```
microk8s status
```

## Add-on'y
Domyślnie otrzymujemy zupełnie podstawowego Kubernetesa. Dodatkowe serwisy mogą zostać łatwo doinstalowane przez system addonów.
My będziemy deployowali aplikację webową, więc będziemy potrzebować `ingress` i load-balancer - dla loadbalancera możemy od razu podać zakres adresów IP zarezerwowanych dla niego w sieci (można to też zrobić później lub zmienić przez specjalny resource `IPAddressPool`)

```
microk8s enable ingress
microk8s enable metallb:172.16.32.210-172.16.32.210

kubectl get all --all-namespaces
microk8s status
```

## Uruchomienie aplikacji
Uruchomimy małą apkę demo z Internetów, tworzymy więc deployment, chcemy 3 repliki i tworzymy serwis. Moglibyśmy to opisać YAMLem, ale rzecz jest na tyle prosta, że chyba nie warto.
```
kubectl create deployment microbot --image=dontrebootme/microbot:v1
kubectl scale  deployment microbot --replicas=3
kubectl expose deployment microbot --port=80 --name=microbot-service
```

Sprawdźmy czy się uruchomiło:
```
kubectl get all --all-namespaces
```

Żeby dostać się do naszej aplikacji z zewnątrz klastra musimy skonfigurować ingress, to już jest trochę więcej, więc napiszmy manifest w YAML:
Ingress

```yaml
--- 
apiVersion: v1
kind: Service
metadata:
  name: ingress
  namespace: ingress
spec:
  selector:
    name: nginx-ingress-microk8s
  type: LoadBalancer
  loadBalancerIP: 172.16.32.210
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
--- 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microbot-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: microbot-service
            port:
              number: 80
```

Możemy przejść do przeglądarki i zobaczyć naszego robocika.

Możemy teraz wysadzić w powietrze jednego, albo dwa node i zobaczyć, czy nasza apka dalej będzie działać.
Jest szansa, że kilka z naszych żądań stimeoutuje, ale wszystkie kolejne już będą działać.


## Źródła
 - https://microk8s.io/
   - [Install a local Kubernetes with MicroK8s](https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s)
   - [How to build a highly available Kubernetes cluster with MicroK8s](https://ubuntu.com/tutorials/getting-started-with-kubernetes-ha)
   - [Addon: MetalLB](https://microk8s.io/docs/addon-metallb)
   - [Addon: Ingress](https://microk8s.io/docs/addon-ingress)