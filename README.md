Aplikacja powstała na potrzeby ćwiczeń Usługi i platformy deweloperskie dla aplikacji w chmurze.
Została ona utworzona w języku programowania Java i został w niej użyty framework SpringBoot.
Opiera się ona o wzorzec JPA czyli standard z grupy tzw. ORM (ang. Object-Relational Mapping).

Narzędzia potrzebne do wykonania poniższych instrukcji:

* ![Azure CLI](https://docs.microsoft.com/pl-pl/cli/azure/install-azure-cli)
* ![Maven](https://maven.apache.org/download.cgi)
* ![Java JDK](https://www.oracle.com/java/technologies/downloads/)
* ![Docker](https://www.docker.com/)


Aby zdeployowac aplikację na Azure Spring Cloud należy wykonać następujące polecenia:

- Konfiguracja zmiennych środowiskowych za pomocą poleceń:

```
AZ_RESOURCE_GROUP=azure-spring-workshop
AZ_DATABASE_NAME=<YOUR_DATABASE_NAME>
AZ_LOCATION=<YOUR_AZURE_REGION>
AZ_MYSQL_USERNAME=spring
AZ_MYSQL_PASSWORD=<YOUR_MYSQL_PASSWORD>
AZ_LOCAL_IP_ADDRESS=<YOUR_LOCAL_IP_ADDRESS>
```

- Tworzenie grupy zasobów:

```
az group create \
    --name $AZ_RESOURCE_GROUP \
    --location $AZ_LOCATION \
    | jq
```

- Tworzenie wystąpienia usługi Azure Database for MySQL

```
az mysql server create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME \
    --location $AZ_LOCATION \
    --sku-name B_Gen5_1 \
    --storage-size 5120 \
    --admin-user $AZ_MYSQL_USERNAME \
    --admin-password $AZ_MYSQL_PASSWORD \
    | jq
```

- Konfigurowanie reguły zapory dla serwera MySQL
  Otwieranie zapory serwera:
 

```
az mysql server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME-database-allow-local-ip \
    --server-name $AZ_DATABASE_NAME \
    --start-ip-address $AZ_LOCAL_IP_ADDRESS \
    --end-ip-address $AZ_LOCAL_IP_ADDRESS \
    | jq
```

  Zezwolenie na dostęp zapory z zasobów platformy Azure:
  
```
az mysql server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name allAzureIPs \
    --server-name $AZ_DATABASE_NAME \
    --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0 \
    | jq
```

- Wdrażanie aplikacji

```
mvn package com.microsoft.azure:azure-webapp-maven-plugin:1.12.0:deploy
``

Budowanie mikroserwisu Spring Boot jako obraz Dockera i umieszczanie go na Azure Container Instance

- Generowanie obrazy przy użyciu Mavena:

```
mvn spring-boot:build-image
```

- Sprawdzenie, czy obraz został poprawnie wygenerowany poprzez uruchomienie go lokalnie:

```
docker run -p 8080:8080 demo:0.0.1-SNAPSHOT
```

- Utorzenie instancji kontenera, używając utworzonej wczesniej grupy

```
az acr create --resource-group $AZ_RESOURCE_GROUP --name $AZ_CONTAINER_NAME --sku Basic
```

- Aby wypchnąć obraz Dockera do Container Registry, należy go oznaczyć używając FQDN rejestru:

```
docker tag demo:0.0.1-SNAPSHOT $AZ_CONTAINER_NAME.azurecr.io/demo:0.0.1-SNAPSHOT
```

- Zalogowanie się do utworzonego kontenera:

```
az acr login --name $AZ_CONTAINER_NAME
```

- Wypchanie lokalnego obrazu do kontenera:

```
docker push $AZ_CONTAINER_NAME.azurecr.io/demo:0.0.1-SNAPSHOT
```

- Sprawdzenie, czy nasza aplikacja "demo" została prawidłowo wypchana

```
az acr repository list --name $AZ_CONTAINER_NAME --output table
```

- Tworzenie kontenera, na którym uruchomiony zostanie obraz Dockera

```
az container create --resource-group $AZ_CONTAINER_NAME --name $AZ_RUN_CONTAINER_NAME --image $AZ_CONTAINER_NAME.azurecr.io/demo:0.0.1-SNAPSHOT --dns-name-label demo-spring-azure --ports 8080
```

- Przed próbą zalogowania trzeba wyświetlić dane do logowania, oraz uruchomić użytkownika admina

```
az acr credential show --name $AZ_CONTAINER_NAME
```

- Logowanie do kontenera

```
az acr update --name $AZ_CONTAINER_NAME --admin-enabled true
```

- Uruchomienie i sprawdzenie, czy konener działa prawidłowo

```
curl 'http://$AZ_RUN_CONTAINER_NAME.eastus.azurecontainer.io:8080/hello'
```


Dodatkowa dokumentacja:

* ![Azure Spring Cloud](https://azure.microsoft.com/pl-pl/services/spring-cloud/)
* ![Container Instance](https://azure.microsoft.com/pl-pl/services/container-instances/)

