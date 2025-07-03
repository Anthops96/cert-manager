# cert-manager
Creating and storing SSL certificates for applications deployed within a Kubernetes cluster. Within our cluster, we can deploy different applications, which in many cases we isolate in namespaces, in order to achieve better organization and security for our projects. Many of these applications rely on SSL certificates to load their configurations and create their secrets, so through this method, we seek to automate the creation of these certificates, as well as make them accessible regardless of whether the applications are in other namespaces.

##  Creation and storage of the certs
First of all, we create the namespace where all the resources responsible for managing the certificates will be deployed. Our first resource to deploy will be the volume where the certificates will be created and stored.

```sh
kubectl create ns cert-manager
kubectl apply -f pvc.yaml
```

The next step is to generate the CA for signing certificates. A cronjob will be deployed with the Elastic image that will be in charge of this task. Once this is achieved, we can create the SSL certificates, another cronjob in this case using the Ubuntu Noble image.

```sh
cd cronjobs
kubectl apply -f cronjob-ca #Generate CA
```
```yaml
         containers:
          - name: elastic-ca-generator
            image: elasticsearch/elasticsearch:8.17.0 #Generate ca certs using elastic image
            securityContext:
              runAsUser: 0
            command:
            - /bin/sh
            - -c
            - |
              set -e
              rm -rf /certs/ca*
              echo "Generating CA..."
              bin/elasticsearch-certutil ca --ca-dn "CN=SERV Root CA,OU=TET,O=COMP,L=Havana,ST=Havana,C=CU" --days 3650 --pem -out /certs/ca.zip --pass "Password" 
              unzip /certs/ca.zip -d /certs
              ls -la /certs
```

```sh
kubectl apply -f generate-cert.yaml #Generate certificates
```
```yaml
      containers:
          - name: generate-cert-global 
            image: ubuntu:noble 
            securityContext:
              runAsUser: 0
            command:
            - /bin/bash
            - -c
            - |
              set -e
              apt update && apt install -y openssl curl zip
              declare -A fqdns
              fqdns=(
                  [1]='my-app')
              rm -rf /certs/certificates.zip
              for fqdn in "${fqdns[@]}"; do rm -rf /certs/$fqdn/; done
              for fqdn in "${fqdns[@]}"; do
                mkdir -p /tmp/certs_generation/$fqdn
                echo "Generate certificates for: $fqdn"
                openssl genrsa -out /tmp/certs_generation/$fqdn/$fqdn.key 2048
                
                echo "Creating CSR (Certificate Signing Request)"
                openssl req -new -key /tmp/certs_generation/$fqdn/$fqdn.key \
                  -subj "/CN=$fqdn.domain.com" -out /tmp/certs_generation/$fqdn/$fqdn.csr
                    
                echo "subjectAltName=DNS:$fqdn.domain.com" > /tmp/certs_generation/$fqdn/$fqdn.ext
                    
                echo "Sign certificate with CA"
                openssl x509 -req -in /tmp/certs_generation/$fqdn/$fqdn.csr \
                  -CA /certs/ca/ca.crt -CAkey /certs/ca/ca.key -passin pass:Password \
                  -CAcreateserial -out /tmp/certs_generation/$fqdn/$fqdn.crt \
                  -days 3650 -sha256 -extfile /tmp/certs_generation/$fqdn/$fqdn.ext
                cd /tmp/certs_generation && zip -r /certs/certificates.zip $fqdn
                unzip /certs/certificates.zip -d /certs
                rm -rf /tmp/certs_generation
              done
              ls -la /certs
```
## Create Nginx Server
The certificates generated previously cannot be consulted by applications that are not from the same namespace as the volume where they are stored, this is because Kubernetes does not allow it, so it is necessary to create an Nginx server, so that other pods, regardless of the namespace in which they are located, can access them.

```sh
cd nginx-server
kubectl apply -f  configm-nginx.yaml #ConfigMap with nginx configuration(https configured
kubectl apply -f server-nginx.yaml #Deployment of nginx server with two replicas and https
kubectl apply -f service-nginx.yaml #Required for access by other pods within the cluster
```

## Configure App Deployment with initcontainer
In order for our application to be able to obtain the certificates, it is necessary to download them through the previously created nginx server. To do this, in our deployment we will add an initcontainer responsible for downloading these certificates, which will be stored in a volume shared with our application.

```sh
cd certs-consumer
kubectl apply -f certs-pvc.yaml #Shared volume between containers
kubectl apply -f deployment.yaml #Deploy app and initcontainer
```
```yaml
     initContainers: 
      - name: cert-downloader #container responsible for downloading certificates from the server
        image: curlimages/curl:8.2.1 
        command:
        - sh
        - -c
        - |
          mkdir -p /shared-certs
          echo "Start downloading the CRT file" 
          curl -k -sSf --retry 3 \
            "https://cert-service.cert-manager.svc.cluster.local/certs/my-app/my-app.crt" \
            -o /shared-certs/tls.crt
          sleep 5
          echo "CRT file download complete"
          echo "Start downloading the KEY file"
          curl -k -sSf --retry 3 \
            "https://cert-service.cert-manager.svc.cluster.local/certs/my-app/my-app.key" \
            -o /shared-certs/tls.key
          sleep 5
          echo "KEY file download complete"
        volumeMounts:
        - name: shared-certs
          mountPath: /shared-certs
      containers:
      - name: my-app #application that will consume the certificates
        image: my-app:image
        volumeMounts:
        - name: shared-certs
          mountPath: /path/of/certs
        ports:
        - containerPort: 8443
      volumes:
      - name: shared-certs
        persistentVolumeClaim:
          claimName: certificates-pvc
```



