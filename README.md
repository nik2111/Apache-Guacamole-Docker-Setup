# Apache-Guacamole-Docker-Setup
 GUAC_FRONTEND=guacamole1.3.0
 GUAC_BACKEND=guacd1.3.0
 GUAC_DATABASE=guac-database-postgress
 GUAC_PASSWORD=

deploy the guacd container
sudo docker run -d --name $GUAC_BACKEND guacamole/guacd:1.3.0

check if the container is running 

sudo docker ps

deploy the postgresq container https://dev.to/shree_j/how-to-install-and-run-psql-using-docker-41j2

sudo docker run -d --name $GUAC_DATABASE -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=$GUAC_PASSWORD -v /docker/data/guacamole/database:/var/lib/postgresql/data  postgres

# Copy the container Id in the Environment variable named ID
ID=$(sudo docker inspect --format="{{.Id}}" $GUAC_DATABASE)

# we want to enter the CLI for the postgres container using the command
sudo docker exec -it $ID bash

# we want to create the database that guacamole will use. The name guacamole_db is required for guacamole to work
createdb -U postgres guacamole_db

# check that the database exits
psql -U postgres
postgres=# \l
#exit

# now we want to actually run a database initialization script on the postgres container guacamole provides this script. All we have to do is run it in the container. For this we need to place the script as a .sql file inside the container
# first we will generate the script and store it in the /tmp folder. --rm means that the contaner is run transiently. This will not build the guacamole frontend. 
sudo docker run --rm guacamole/guacamole:1.3.0 /opt/guacamole/bin/initdb.sh --postgres > /tmp/initdb.sql

# if there is no error all is well but still check if the file exists 
ls /tmp | grep initdb.sql

# now copy this into the docker container for postgres
sudo docker cp /tmp/initdb.sql $GUAC_DATABASE:/initdb.sql

# enter the CLI for the postgres again 
sudo docker exec -it $ID bash

#run the database initialization script
psql -U postgres guacamole_db -f /initdb.sql


# exit the CLI of the container
exit
#build guacamole frontend
sudo docker run -d --name $GUAC_FRONTEND --link $GUAC_BACKEND:guacd --link $GUAC_DATABASE:postgres -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=$GUAC_PASSWORD -e POSTGRES_DATABASE=guacamole_db -v /docker/data/guacamole/guacamole:/guac   -e GUACAMOLE_HOME=/guac -p 8080:8080 guacamole/guacamole:1.3.0
