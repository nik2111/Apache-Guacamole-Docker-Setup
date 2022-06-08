Guacamole actually consists of three parts the guacamole frontend, the backend and the database. For more in depth understanding please refer the [manual](https://guacamole.apache.org/doc/gug/guacamole-architecture.html) 
[[File:Guacamole overview.jpg|thumb|center|Guacamole overview]]
Create the following environment varaiables that represent the container names for the guacamole setup. The password for the database should be saved in GUAC_PASSWORD


``` 
GUAC_FRONTEND=guacamole1.3.0
GUAC_BACKEND=guacd1.3.0
GUAC_DATABASE=guac-database-postgress
GUAC_PASSWORD=
 ```

Deploy the guacamole backend container 
```
sudo docker run -d --name $GUAC_BACKEND guacamole/guacd:1.3.0
```
check if the container is running using(you can also use portainer)
```
sudo docker ps
```
deploy the postgresq container. For a quick postgress deployment tutorial see [link](https://dev.to/shree_j/how-to-install-and-run-psql-using-docker-41j2) 
```
sudo docker run -d --name $GUAC_DATABASE -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=$GUAC_PASSWORD \
 -v /docker/data/guacamole/database:/var/lib/postgresql/data  postgres
```
Copy the container Id in the Environment variable named ID
```
ID=$(sudo docker inspect --format="{{.Id}}" $GUAC_DATABASE)
```
Enter the CLI for the postgres container using the command
```
sudo docker exec -it $ID bash
```
Create the database that guacamole will use. The name guacamole_db is required for guacamole to work. Check [this link](https://www.postgresql.org/docs/9.1/app-createdb.html) if you have trouble with psql commands
```
createdb -U postgres guacamole_db
```
check that the database exits
```
 psql -U postgres
 postgres=# \l
```
You should see the database guacamole_db in the list of databases

Now we want to actually run a database initialization script on the postgres container. Guacamole provides this script inside the frontend container. All we have to do is run it in the database container container. For this we need to place the script as a .sql file inside the database container. First we will generate the script and store it in the /tmp folder. 
```
sudo docker run --rm guacamole/guacamole:1.3.0 /opt/guacamole/bin/initdb.sh --postgres > /tmp/initdb.sql
 ```
In the above command --rm means that the contaner is run transiently. This will not build the guacamole frontend even though it is the same container. The container will be killed once the command is executed.

If there is no error all is well but still check if the file exists 
```
ls /tmp | grep initdb.sql
```
now copy this into the docker container for postgres
```
sudo docker cp /tmp/initdb.sql $GUAC_DATABASE:/initdb.sql 
```
enter the CLI for the postgres again
```
sudo docker exec -it $ID bash
```
run the [database initialization script](https://stackoverflow.com/questions/8208181/create-postgres-database-using-batch-file-with-template-encoding-owner-and)
```
psql -U postgres guacamole_db -f /initdb.sql
```
Exit the CLI of the container and build guacamole frontend with the folowing command
```
 sudo docker run -d --name $GUAC_FRONTEND --link $GUAC_BACKEND:guacd --link $GUAC_DATABASE:postgres \
 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=$GUAC_PASSWORD -e POSTGRES_DATABASE=guacamole_db \
 -v /docker/data/guacamole/guacamole:/guac   -e GUACAMOLE_HOME=/guac -p 8080:8080 guacamole/guacamole:1.3.0
```
