# springboot-keycloak-openldap

The goal of this project is to create a simple [Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/) REST API, called `clinico-omics`, and secure it with [`Keycloak`](https://www.keycloak.org). Furthermore, the API users will be loaded into `Keycloak` from [`OpenLDAP`](https://www.openldap.org) server.

> **Note 1:** In [`springboot-react-keycloak`](https://github.com/ivangfr/springboot-react-keycloak) repository, we have implemented a `movies-app` using `Keycloak` (with `PKCE`). This application consists of two services: the backend that was implemented using `Spring Boot` and the frontend implemented with `ReactJS`.

> **Note 2:** In [`docker-swarm-environment`](https://github.com/ivangfr/docker-swarm-environment) repository, it's shown how to deploy this project into a cluster of Docker Engines in swarm mode. Besides, we will be running a Keycloak cluster with more than one instance.

## Application

- ### clinico-omics

  `Spring Boot` Web Java application that exposes two endpoints:
  
  - `/api/public`: endpoint that can be access by anyone, it is not secured;
  - `/api/private`: endpoint that can just be accessed by users that provide a `JWT` token issued by `Keycloak` and the token must contain the role `USER`.

## Prerequisites

- [`Java 11+`](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)
- [`Docker`](https://www.docker.com/)
- [`Docker-Compose`](https://docs.docker.com/compose/install/)
- [`jq`](https://stedolan.github.io/jq)

## Start Environment

- Open a terminal and inside `springboot-keycloak-openldap` root folder run
  ```
  docker-compose up -d
  ```

- Wait a bit until `MySQL` and `Keycloak` containers are `Up (healthy)`. In order to check the status of the containers, run the command
  ```
  docker-compose ps
  ```

## Import OpenLDAP Users

The `LDIF` file that we will use, `springboot-keycloak-openldap/ldap/ldap-mycompany-com.ldif`, contains a pre-defined structure for `mycompany.com`. Basically, it has 2 groups (`developers` and `admin`) and 4 users (`Bill Gates`, `Steve Jobs`, `Mark Cuban` and `Ivan Franchin`). Besides, it's defined that `Bill Gates`, `Steve Jobs` and `Mark Cuban` belong to `developers` group and `Ivan Franchin` belongs to `admin` group.
```
Bill Gates > username: bgates, password: 123
Steve Jobs > username: sjobs, password: 123
Mark Cuban > username: mcuban, password: 123
Ivan Franchin > username: ifranchin, password: 123
```

There are two ways to import those users: running a script or using `phpldapadmin` website

### Running a script

In a terminal and inside `springboot-keycloak-openldap` root folder run
```
./import-openldap-users.sh
```

### Using phpldapadmin website

- Access https://localhost:6443

- Login with the credentials
  ```
  Login DN: cn=admin,dc=mycompany,dc=com
  Password: admin
  ```

- Import the file `springboot-keycloak-openldap/ldap/ldap-mycompany-com.ldif`

- You should see a tree like the one shown in the picture below

  ![phpldapadmin](images/phpldapadmin.png)

## Configure Keycloak

There are two ways: running a script or using `Keycloak` website

### Running a script

- In a terminal, make sure you are inside `springboot-keycloak-openldap` root folder

- Run the script below to configure `Keycloak` for `clinico-omics` application
  ```
  ./init-keycloak.sh
  ```

  This script creates `company-services` realm, `clinico-omics` client, `USER` client role, `ldap` federation and the users `bgates` and `sjobs` with the role `USER` assigned.

- Copy `SIMPLE_SERVICE_CLIENT_SECRET` value that is shown at the end of the script. It will be needed whenever we call `Keycloak` to get a token to access `clinico-omics`

### Using Keycloak website

![keycloak](images/keycloak.png)

#### Login

- Access http://localhost:8080/auth/admin/

- Login with the credentials
  ```
  Username: admin
  Password: admin
  ```

#### Create a new Realm

- Go to top-left corner and hover the mouse over `Master` realm. A blue button `Add realm` will appear. Click on it
- On `Name` field, write `company-services`. Click on `Create`

#### Create a new Client

- Click on `Clients` menu on the left
- Click `Create` button
- On `Client ID` field type `clinico-omics`
- Click on `Save`
- On `Settings` tab, set the `Access Type` to `confidential`
- Still on `Settings` tab, set the `Valid Redirect URIs` to `http://localhost:9080`
- Click on `Save`
- Go to `Credentials` tab. Copy the value on `Secret` field. It will be used on the next steps
- Go to `Roles` tab
- Click `Add Role` button
- On `Role Name` type `USER`
- Click on `Save`

#### LDAP Integration

- Click on the `User Federation` menu on the left
- Select `ldap`
- On `Vendor` field select `Other`
- On `Connection URL` type `ldap://ldap-host`
- Click on `Test connection` button, to check if the connection is OK
- On `Users DN` type `ou=users,dc=mycompany,dc=com`
- On `Custom User LDAP Filter` set `(gidnumber=500)` to just get developers
- On `Bind DN` type `cn=admin,dc=mycompany,dc=com`
- On `Bind Credential` set `admin`
- Click on `Test authentication` button, to check if the authentication is OK
- Click on `Save`
- Click on `Synchronize all users`

#### Configure users imported

- Click on `Users` menu on the left
- Click on `View all users`. 3 users will be shown
- Edit user `bgates`
- Go to `Role Mappings` tab
- In the search field `Client Roles`, select `clinico-omics`
- Select the role `USER` present in `Available Roles` and click on `Add selected`
- Done. `bgates` has now the role `USER` as one of his `Assigned Roles`
- Do the same for the user `sjobs`
- Let's leave `mcuban` without `USER` role

## Run clinico-omics using Maven

- Open a new terminal

- In `springboot-keycloak-openldap` root folder, run the command below to start `clinico-omics` application
  ```
  ./mvnw clean spring-boot:run --projects clinico-omics -Dspring-boot.run.jvmArguments="-Dserver.port=9080"
  ```

## Test using curl

1. Open a new terminal

1. Call the endpoint `GET /api/public`
   ```
   curl -i http://localhost:9080/api/public
   ```
   
   It should return
   ```
   HTTP/1.1 200
   It is public.
   ```
   
1. Try to call the endpoint `GET /api/private` without authentication
   ``` 
   curl -i http://localhost:9080/api/private
   ```
   
   It should return
   ```
   HTTP/1.1 302
   ```
   > Here, the application is trying to redirect the request to an authentication link

1. Create an environment variable that contains the `Client Secret` generated by `Keycloak` to `clinico-omics` at [Configure Keycloak](#configure-keycloak) step
   ```
   SIMPLE_SERVICE_CLIENT_SECRET=...
   ```

1. Run the command below to get an access token for `bgates` user
   ```
   BGATES_ACCESS_TOKEN=$(curl -s -X POST \
     "http://localhost:8080/auth/realms/company-services/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=bgates" \
     -d "password=123" \
     -d "grant_type=password" \
     -d "client_secret=$SIMPLE_SERVICE_CLIENT_SECRET" \
     -d "client_id=clinico-omics" | jq -r .access_token)
   ```

1. Call the endpoint `GET /api/private`
   ```
   curl -i -H "Authorization: Bearer $BGATES_ACCESS_TOKEN" http://localhost:9080/api/private
   ```
   
   It should return
   ```
   HTTP/1.1 200
   bgates, it is private.
   ```

1. Run the command below to get an access token for `mcuban` user
   ```
   MCUBAN_ACCESS_TOKEN=$(curl -s -X POST \
     "http://localhost:8080/auth/realms/company-services/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=mcuban" \
     -d "password=123" \
     -d "grant_type=password" \
     -d "client_secret=$SIMPLE_SERVICE_CLIENT_SECRET" \
     -d "client_id=clinico-omics" | jq -r .access_token )
   ```

1. Try to call the endpoint `GET /api/private`
   ```
   curl -i -H "Authorization: Bearer $MCUBAN_ACCESS_TOKEN" http://localhost:9080/api/private
   ```
   As `mcuban` does not have the `USER` role, he cannot access this endpoint.
   
   The endpoint return should be
   ```
   HTTP/1.1 403
   {
     "timestamp":"2018-12-26T13:14:10.493+0000",
     "status":403,
     "error":"Forbidden",
     "message":"Forbidden",
     "path":"/api/private"
   }
   ```

1. Go to `Keycloak` and add the role `USER` to the `mcuban`

1. Run the command on `step 7)` again to get a new access token for `mcuban` user

1. Call again the endpoint `GET /api/private` using the `curl` command presented on `step 8`

   It should return
   ```
   HTTP/1.1 200
   mcuban, it is private.
   ```

1. The access token default expiration period is `5 minutes`. So, wait for this time and, using the same access token, try to call the private endpoint.

   It should return
   ```
   HTTP/1.1 401
   WWW-Authenticate: Bearer realm="company-services", error="invalid_token", error_description="Token is not active"
   ```

## Test using Swagger

1. Access http://localhost:9080/swagger-ui.html

   ![clinico-omics-swagger](images/clinico-omics-swagger.png)

1. Click on `GET /api/public` to open it. Then, click on `Try it out` button and, finally, click on `Execute` button

   It should return
   ```
   Code: 200
   Response Body: It is public.
   ```

1. Now click on `GET /api/private`, it is a secured endpoint. Let's try it without authentication

1. Click on `Try it out` button and then on `Execute` button
  
   It should return
   ```
   TypeError: Failed to fetch
   ```

1. In order to access the private endpoint, you need an access token. So, open a terminal

1. Create an environment variable that contains the `Client Secret` generated by `Keycloak` to `clinico-omics` at [Configure Keycloak](#configure-keycloak) step
   ```
   SIMPLE_SERVICE_CLIENT_SECRET=...
   ```
  
1. Run the following commands
   ```
   BGATES_ACCESS_TOKEN=$(curl -s -X POST \
     "http://localhost:8080/auth/realms/company-services/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=bgates" \
     -d "password=123" \
     -d "grant_type=password" \
     -d "client_secret=$SIMPLE_SERVICE_CLIENT_SECRET" \
     -d "client_id=clinico-omics" | jq -r .access_token)
     
   echo $BGATES_ACCESS_TOKEN
   ```

1. Copy the token generated and go back to `Swagger`

1. Click on the `Authorize` button, paste the access token in the value field. Then, click on `Authorize` and, to finalize, click on `Close`

1. Go to `GET /api/private`, click on `Try it out` and then on `Execute` button

   It should return
   ```
   Code: 200
   Response Body: bgates, it is private.
   ```

## Using client_id and client_secret to get access token

You can get an access token to `clinico-omics` using `client_id` and `client_secret`

### Configuration

- Access http://localhost:8080/auth/admin/
- Select `company-services` realm (if it is not already selected)
- Click on `Clients` on the left menu
- Select `clinico-omics` client
- On `Settings` tab, turn `ON` the field `Service Accounts Enabled`
- Click on `Save`
- On `Service Account Roles`tab, select `clinico-omics` on the `Client Roles` search field
- Select the role `USER` present in `Available Roles` and click on `Add selected`

### Test

1. Open a terminal

1. Create an environment variable that contains the `Client Secret` generated by `Keycloak` to `clinico-omics` at [Configure Keycloak](#configure-keycloak) step
   ```
   SIMPLE_SERVICE_CLIENT_SECRET=...
   ```
  
1. Run the following command
   ```
   CLIENT_ACCESS_TOKEN=$(curl -s -X POST \
     "http://localhost:8080/auth/realms/company-services/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=client_credentials" \
     -d "client_secret=$SIMPLE_SERVICE_CLIENT_SECRET" \
     -d "client_id=clinico-omics" | jq -r .access_token)
   ```
  
1. Try to call the endpoint `GET /api/private`
   ```
   curl -i http://localhost:9080/api/private -H "authorization: Bearer $CLIENT_ACCESS_TOKEN"
   ```
  
   It should return
   ```
   HTTP/1.1 200
   service-account-clinico-omics, it is private.
   ```

## Running clinico-omics as a Docker container

- Build Docker Image
  ```
  ./mvnw clean compile jib:dockerBuild --projects clinico-omics
  ```
  | Environment Variable | Description                                                 |
  | -------------------- | ----------------------------------------------------------- |
  | `KEYCLOAK_HOST`      | Specify host of the `Keycloak` to use (default `localhost`) |
  | `KEYCLOAK_PORT`      | Specify port of the `Keycloak` to use (default `8080`)      |

- Run `clinico-omics` docker container, joining it to docker-compose network
  ```
  docker run -d --rm -p 9080:8080 \
    --name clinico-omics \
    --network=springboot-keycloak-openldap_default \
    --env KEYCLOAK_HOST=keycloak \
    docker.mycompany.com/clinico-omics:1.0.0
  ```

- Create an environment variable that contains the `Client Secret` generated by `Keycloak` to `clinico-omics` at [Configure Keycloak](#configure-keycloak) step
  ```
  SIMPLE_SERVICE_CLIENT_SECRET=...
  ```  

- Run the commands below to get an access token for `bgates` user
  ```
  BGATES_TOKEN=$(
      docker exec -t -e CLIENT_SECRET=$SIMPLE_SERVICE_CLIENT_SECRET keycloak bash -c '
        curl -s -X POST \
        http://keycloak:8080/auth/realms/company-services/protocol/openid-connect/token \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -d "username=bgates" \
        -d "password=123" \
        -d "grant_type=password" \
        -d "client_secret=$CLIENT_SECRET" \
        -d "client_id=clinico-omics"')
  
  BGATES_ACCESS_TOKEN=$(echo $BGATES_TOKEN | jq -r .access_token)
  ```

- Call the endpoint `GET /api/private`
  ```
  curl -i -H "Authorization: Bearer $BGATES_ACCESS_TOKEN" http://localhost:9080/api/private
  ```

## Shutdown

- Stop `clinico-omics` application
  - If it was started with `Maven`, go to the terminal where it is running and press `Ctrl+C`
  - If it was started as a Docker container, run in a terminal the command below
    ```
    docker stop clinico-omics
    ```

- To stop and remove docker-compose containers, networks and volumes, run the command below in `springboot-keycloak-openldap` root folder
  ```
  docker-compose down -v
  ```

## Useful Links/Commands

- **jwt.io**

  With [jwt.io](https://jwt.io) you can inform the JWT token received from `Keycloak` and the online tool decodes the token, showing its header and payload.

- **ldapsearch**

  It can be used to check the users imported into `OpenLDAP`
  ```
  ldapsearch -x -D "cn=admin,dc=mycompany,dc=com" \
    -w admin -H ldap://localhost:389 \
    -b "ou=users,dc=mycompany,dc=com" \
    -s sub "(uid=*)"
  ```

## References

- https://www.keycloak.org/docs/latest/server_admin/