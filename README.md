# Project to setup the CHN (Canadian Hydro Netowrk) system with all the components in Docker

1. Install wsl2 and ubuntu-20.04

    wsl --install -d Ubuntu-20.04

2. install docker desktop
3. Install local installation of Visual Studio Code
   1. install the following extensions ("Remote Development")
4. open command prompt and start a terminal inside ubuntu

```
wsl
```
5. go to ~ and clone chn-compose project into specific directory

```
    cd ~
    git clone ??remote-url?? ./chn
```
6. go into the chn folder, setup workspace and run vscode
   1. this step includes getting the test data for the project and placing it in the "chn" folder (NHDFLOW_Catchment.gpkg and NHD_flow.csv)

```
cd chn
mkdir graphdb
mkdir gisdb
touch frontend.env
touch backend.env
touch gisdb.env
touch graphdb.env
touch pgadmin.env
touch servers.json

code .
```

TODO: describe each of the configuration files created above.

visual studio code server will be installed inside ubuntu and will open (check to make sure on the bottom left corner of vs code it will show green "WSL: Ubuntu-20.04")

7. Go to extensions and install "Docker" extension
8. Ctrl-Shift-P to bring up vscode command pallette and run "Git: Clone", then choose "Clone from Github", login to your Github account and 
9. Choose "chn-database", choose to place the project in "chn" folder, it will create a sub-folder
10. Repeat and choose "chn-frontend", choose to place the project in "chn" folder, it will create a sub-folder
11. Right-click on "docker-compose.system.yml" and choose "Compose Up"

12. Now we need to load the data into PostGIS

```
docker run --name qgisforgdalutils --network chn_internalonly -it --rm -v ~/chn:/root/mydata/ qgis/qgis:latest /bin/bash
# then inside the bash terminal of the docker container qgisforgdalutils run the following
ogr2ogr -f PostgreSQL "PG:host=gisdb user=postgres password=sherbrooke dbname=postgres" /root/mydata/NHDFLOW_Catchment.gpkg
exit
```
in the distro wsl prompt replace \<ubuntuuser\> with your own when you installed ubuntu
```
sudo chown -R <ubuntuuser>:<ubuntuuser> ~/chn/gisdb/pgadmin/
mv ~/chn/servers.json ~/chn/gisdb/pgadmin
sudo chown -R 5050:5050 ~/chn/gisdb/pgadmin/
```

(this doesnt' work yet) in pgadmin terminal, this will add the postgis database to the connection info
```
# don't run this command
/venv/bin/python3 setup.py --load-servers servers.json --sqlite-path /var/lib/pgadmin/pgadmin4.db
```
13. load neo4j data

make sure the mounted "import" folder has the right permissions in your distro
```
sudo chown -R <ubuntuuser>:<ubuntuuser> ~/chn/graphdb/import
mv ~/chn/NHD_flow.csv ~/chn/graphdb/import
sudo chown -R 7474:7474 ~/chn/graphdb/import
```

Execute the import commands to bring in csv data, run this docker exec in your distro terminal
```
docker exec --interactive --tty chn-graphdb-1 cypher-shell -u neo4j -p neo4j 
```

You should be able to paste this into the cypher-shell and it will load the data into neo4j
```
// 1. Create constraint that NodeID is Unique
CREATE CONSTRAINT FOR (s:Segment) REQUIRE s.NodeID IS UNIQUE;
// 2. Create nodes using MERGE
LOAD CSV WITH HEADERS FROM 'file:///NHD_flow.csv' AS line
MERGE (s:Segment {NodeID: line.NHDPlusID})
ON CREATE SET
    s.downStream = line.NHDPlusFlowlineVAA_ToNode,
    s.upStream = line.NHDPlusFlowlineVAA_FromNode;
// 3. Create relationship
MATCH
  (a:Segment),
  (b:Segment)
WHERE a.downStream = b.upStream
CREATE (b)-[r:upstream]->(a)
CREATE (a)-[e:downstream]->(b)
RETURN r,e;
```
