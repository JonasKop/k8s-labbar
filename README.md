# Kubernetes labbar


## Allmänt sköna debuggingkommandon
I Kubernetes finns det otroligt många kommandon men alla bygger på någorlunda samma struktur. Om någonting strular brukar man oftast få en förklaring till varför med hjälp av dessa. 
- Hämta alla poddar
```bash
kubectl get pods
```
- Följ alla poddar
```bash
kubectl get pods --watch
```
- Hämta en pods konfiguration som yaml
```bash
kubectl get pod $POD_NAME -o yaml 
```
- Hämta information gällande körningen av en pod
```bash
kubectl describe pod $POD_NAME -o yaml 
```
- Öppna ett exekverbart skal, i detta fall SH, till en pod
```bash
kubectl exec --stdin --tty $POD_NAME -- sh
```
- Spana in en pods loggar
```bash
kubectl logs $POD_NAME
```
- Följ en pods loggar
```bash
kubectl logs $POD_NAME -f
```
- Ta bort en resurs
```bash
kubectl delete pod $RESOURCE $NAME

# example
kubectl delete deployment nginx
```



## Bra saker att kunna 
- UNIX - `CTRL`+`C` avbryter ett körande program
- UNIX - `env` skriver ut alla environment variabler
- Webbläsare cachar grejer vilket gör webbutveckling lite frustrerande. Om dina ändringar inte visas så testa refresha utan cache.
  - Linux/Windows:
    - Chrome: `CTRL`+`F5`
    - Firefox: `CTRL`+`F5`
  - macOS:
    - Chrome: `CMD`+`SHIFT`+`N`
    - Firefox: `CMD`+`SHIFT`+`N`
    - Safari: `CMD`+`OPTION`+`N`
- Allting som jag skriver med dollartecken före, tex `$POD_NAME`, vill jag att du ersätter med rätt värde. 


## 1. Installera minikube
Vi börjar med att installera minikube, en lokal kubernetesversion som körs på din dator. Ett cloudbaserat kluster (typ EKS, GKE) är oftast bättre på de flesta plan men minikube är fantastiskt smidigt om man ska testa någonting lite snabbt. 

1. [Ladda ner](https://minikube.sigs.k8s.io/docs/start/) och installera minikube.

2. Starta ditt kluster 
```bash
# macOS (Om du inte kör med hyperkit så kommer du inte att kunna skapa en ingress på macOS pga drivrutinsproblem)
minikube start --driver=hyperkit

# Linux
minikube start
```
3. Installera kubectl
```bash
minikube kubectl -- get po -A
```
4. Linux/macOS - Sätt på autocompletion (tryck \<TAB\>) 
```bash
# Hämta ditt skal
echo $SHELL

# Om du har BASH
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Om du har ZSH
echo "[[ $commands[kubectl] ]] && source <(kubectl completion zsh)" >> ~/.zshrc
```
5. Testa hämta alla dina poddar
```bash
kubectl get pods -A
```
6. Spana in din dashboard och vänta så att allting har startat upp korrekt. 
```bash
minikube dashboard
```




## 2. Kör en container i kubernetes
Vi ska nu testa att köra vår första container i kubernetes. Det som ska göras är att skapa en deployment som skapar en nginx instans i kubernetes som vi sedan kan nå via vår webbläsare. Nginx är en webbserver och en webbproxy som bland annat kan användas för att lägga upp hemsidor. 

1. Skapa en fil `deployment.yaml` (namnet spelar egentligen ingen roll) med följande innehåll. 
```yaml
# Vilken API version av kubernetes som du vill använda
apiVersion: apps/v1
# Typ av resurs
kind: Deployment
# Metadata om din fil, här specas namn, namespace etc
metadata:
  name: nginx
# Själva konfigurationsdatan av din deployment
spec:
  # Används av en service för att veta vad den ska proxa requests till
  selector:
    matchLabels:
      app: nginx
  # Din podkonfiguration
  template:
    metadata:
      labels:
        app: nginx
    # Pod specification
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        # Öppna upp port 80
        ports:
        - containerPort: 80
```
2. Tillämpa din konfiguration på ditt kluster
```bash
kubectl apply -f deployment.yaml
```
3. Se så att din pod körs. Vänta på att de ska ha status `Running`.
```bash
kubectl get pods
```
4. Port forward en port, 8080, på din dator till port 80 på podden.
```bash
kubectl port-forward $POD_NAME 8080:80
```
5. Gå in på [http://localhost:8080](http://localhost:8080).



## 3. Konfigurera din nginx med en fil
Vi vill nu testa att konfigurera vår nginx-instans så att den svarar med vår alldeles egna hemsida. 
1. Skapa en fil `configmap.yaml` med följande data.
```yaml
# Vilken API version av kubernetes som du vill använda
apiVersion: v1
# Typ av resurs
kind: ConfigMap
# Metadata om din fil, här specas namn, namespace etc
metadata:
  name: nginx-conf
# Vår nginx config-fil. 
data:
  localhost.conf: |
    server {
      listen        80;
      root /var/www/html;
      index index.html;
      
      location / {
        try_files $uri $uri/ /index.html;
      }
    }
``` 
2. Tillämpa din configmap
```bash
kubectl apply -f configmap.yaml
```
3. Redigera din deployment.yaml under spec
```yaml
# Vilken API version av kubernetes som du vill använda
apiVersion: apps/v1
# Typ av resurs
kind: Deployment
# Metadata om din fil, här specas namn, namespace etc
metadata:
  name: nginx
# Själva konfigurationsdatan av din deployment
spec:
  # Används av en service för att veta vad den ska proxa requests till
  selector:
    matchLabels:
      app: nginx
  # Din podkonfiguration
  template:
    metadata:
      labels:
        app: nginx
    # Pod specification
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        # Öppna upp port 80
        ports:
        - containerPort: 80
########
# NYTT #
########
        # Fäst din config på en specifik path i containern
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
      # Lägg till din config map som en volym
      volumes:
      - name: config-volume
        configMap:
          name: nginx-conf
########
# NYTT #
########
```
4. Tillämpa dina ändringar
```bash
kubectl apply -f deployment.yaml
```
5. Skapa en `index.html` fil på rätt plats i din container. 
```bash
kubectl exec $POD_NAME -- sh -c 'mkdir -p /var/www/html && echo "<h1>Halojsan</h1>" > /var/www/html/index.html'
```
6. Port forward en port, 8080, på din dator till port 80 på podden.
```bash
kubectl port-forward $POD_NAME 8080:80
```
7. Gå in på [http://localhost:8080](http://localhost:8080)



## 4. Spara din hemsida mellan körningar
Vi har nu skapat en fil, `index.html`, i vår container. Det skulle vara skönt om den sparades när containern startas om så det ska vi fixa nu. 
1. Ta bort din pod. 
```bash
kubectl delete pod $POD_NAME 
```
2. Eftersom att vi använder en deployment så skapar kubernetes automatiskt en ny podd. 
```bash
kubectl get pods
```
3. Då vi tog bort vår pod så försvann även filen `index.html` som vi skapat.
```bash
kubectl exec $POD_NAME -- cat /var/www/html/index.html
```
4. För att bibehålla den vid omstart så lägger vi till en volym. Skapa en fil `pvc.yaml`
```yaml
# Vilken API version av kubernetes som du vill använda
apiVersion: v1
# Typ av resurs
kind: PersistentVolumeClaim
# Metadata om din fil, här specas namn, namespace etc
metadata:
  name: nginx-pvc
# Själva konfigurationsdatan av din deployment
spec:
  # Endast en kan skriva
  accessModes:
    - ReadWriteOnce
  # Storleken på vår pvc
  resources:
    requests:
      storage: 100Mi
```
5. Tillämpa din konfiguration
```bash
kubectl apply -f pvc.yaml
```
6. Redigera din `deployment.yaml` och lägg till din volym. 
```yaml
# Vilken API version av kubernetes som du vill använda
apiVersion: apps/v1
# Typ av resurs
kind: Deployment
# Metadata om din fil, här specas namn, namespace etc
metadata:
  name: nginx
# Själva konfigurationsdatan av din deployment
spec:
  # Används av en service för att veta vad den ska proxa requests till
  selector:
    matchLabels:
      app: nginx
  # Din podkonfiguration
  template:
    metadata:
      labels:
        app: nginx
    # Pod specification
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        # Öppna upp port 80
        ports:
        - containerPort: 80
        # Fäst din config på en specifik path i containern
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
########
# NYTT #
########
        # Fäst din volym
        - name: website
          mountPath: /var/www/html

      # Lägg till din nya volym
      volumes:
      - name: website
        persistentVolumeClaim:
          claimName: nginx-pvc
########
# NYTT #
########
      # Lägg till din config map som en volym
      - name: config-volume
        configMap:
          name: nginx-conf
```
7. Tillämpa din konfiguration
```bash
kubectl apply -f deployment.yaml
```
8. Skapa en `index.html` fil på rätt plats i din container. 
```bash
kubectl exec $POD_NAME -- sh -c 'mkdir -p /var/www/html && echo "<h1>Halojsan</h1>" > /var/www/html/index.html'
```
9. Ta bort din pod
```bash
kubectl delete pod $POD_NAME
```
10. Port forward en port, 8080, på din dator till port 80 på den nya podden.
```bash
kubectl port-forward $POD_NAME 8080:80
```
11. Gå in på [http://localhost:8080](http://localhost:8080)

## 5. Sätt upp en ingress
Vi vill nu ta och öppna upp vår hemsida till omvärlden, så att man kan besöka den via en domän och få olika svar beroende på vad vår URL är. 
1. Installera ingress nginx i vårt kluster. 
```bash
minikube addons enable ingress
```
2. Verifiera att den körs (allting ska ha statusen `Running` eller `Completed`)
```bash
kubectl get pods -n kube-system
```
3. Skapa en service som lastbalanserar våra requests till vår pod i filen `service.yaml`. 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  ports:
    - port: 8080
      targetPort: 80
  selector:
    app: nginx
```
4. Tillämpa din konfiguration
```bash
kubectl apply -f service.yaml
```
5. Skapa en ingress i filen `ingress.yaml` som gör så att vi via DNS kan nå vår pod. 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
    - host: nginx-grejer.info
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 8080
```
6. Tillämpa din konfiguration
```bash
kubectl apply -f ingress.yaml
```
7. Hämta din minikubes ip
```bash
minikube ip
```
8. Lägg till en DNS resolution IP i `/etc/hosts`. OBS. Detta är ingenting som man behöver göra på ett "riktigt" kluster då man då har riktiga domännamn.
```bash
# Öppna filen med en text-editor
sudo nano /etc/hosts

# Lägg till följande längst ner
$YOUR_MINIKUBE_IP nginx-grejer.info

# Spara filen med CTRL+O sedan ENTER sedan CTRL+X
```
9. Gå in på [http://nginx-grejer.info](http://nginx-grejer.info)
10. Skapa en ny fil `deployment-2.yaml`
```yaml
# Vilken API version av kubernetes som du vill använda
apiVersion: apps/v1
# Typ av resurs
kind: Deployment
# Metadata om din fil, här specas namn, namespace etc
metadata:
  name: test-server
# Själva konfigurationsdatan av din deployment
spec:
  # Används av en service för att veta vad den ska proxa requests till
  selector:
    matchLabels:
      app: test-server
  # Din podkonfiguration
  template:
    metadata:
      labels:
        app: test-server
    # Pod specification
    spec:
      containers:
      - name: test-server
        image: jonaskop/test-server
        # Sätt lite environment variabler
        env:
        - name: PORT
          value: "80"
        - name: TEXT
          value: "Hejsan, här har vi lite text"
        # Öppna upp port 80
        ports:
        - containerPort: 80
        # Fäst din config på en specifik path i containern
```
11. Tillämpa din konfiguration
```bash
kubectl apply -f deployment-2.yaml
```
12. Skapa en service som lastbalanserar våra requests till vår nya pod i filen `service-2.yaml`. 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-server-svc
spec:
  ports:
    - port: 8080
      targetPort: 80
  selector:
    app: test-server
```
13. Tillämpa din konfiguration
```bash
kubectl apply -f service-2.yaml
```
13. Redigera din `ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
    - host: nginx-grejer.info
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: test-server-svc
                port:
                  number: 8080
```
14. Tillämpa din konfiguration
```bash
kubectl apply -f service-2.yaml
```
15. Gå in på [http://nginx-grejer.info](http://nginx-grejer.info)

## 6. Avsluta allting
1. Stäng av ditt kluster
```bash
minikube stop
```
2. Ta bort ditt kluster
```bash
minikube delete
```
3. Rensa din `/etc/hosts`
```bash
# Öppna filen med en text-editor
sudo nano /etc/hosts

# Ta bort följande längst ner
$YOUR_MINIKUBE_IP nginx-grejer.info

# Spara filen med CTRL+O sedan ENTER sedan CTRL+X
```