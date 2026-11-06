# Neo4j Docker Image with APOC

Docker image for Neo4j 5.26.26 with APOC plugin pre-installed (for dynamic properties support).The image is used e.g. by [jejune_extract_knowledge_graph](https://github.com/EricBoix/jejune_extract_knowledge_graph).

## Usage

```bash
docker build -t jejune:jj_neo4j_docker https://github.com/EricBoix/jj_neo4j_docker.git
```

```bash
docker run \
    --publish=7474:7474 --publish=7687:7687 \
    --env NEO4J_AUTH=neo4j/your_password \
    -v `pwd`/data:/data \
    --rm \
    jejune:jj_neo4j_docker
```

## What's Included

- Neo4j 5.26.26
- APOC 5.26.26 core plugin
- Ports exposed: 7474 (HTTP), 7687 (Bolt)

## Devel debug notes

### Testing connections with the standard neo4j image

Let us run the standard neo4j container and realize a port forwarding of the `bolt` protocol to an alternative (host) port (7666) with

```bash
docker run --rm --name neo4j \
--publish=7666:7687  \
--env NEO4J_AUTH=neo4j/your_password \
--detach neo4j
```

Notice that the log of this container (run it without the `--detach` flag) signals that `Bolt enabled on 0.0.0.0:7687` because the container is not aware of the port forwarding (either use `docker ps` or e.g. `lsof -i -P | grep -i "listen"` on OSX to discover which host ports are listening).

Then we are able to run the following commands, that both validate the `your_password` configured password

- ```bash
  docker exec -it neo4j /var/lib/neo4j/bin/cypher-shell -a bolt://localhost:7687 -u neo4j -p your_password "MATCH () RETURN count(*) as count"
  ```

  that is executed inside the (already running) container (and thus only sees the neo4j native port `7687`)

- ```bash
  docker run --network host neo4j /var/lib/neo4j/bin/cypher-shell -a bolt://127.0.0.1:7666 -u neo4j -p your_password "MATCH () RETURN count(*) as count"
  ```

  that is executed inside a new neo4j container (and thus only sees the forwarded port `7666` on condition that the `--network host` flag is used in order for this new container to have access to its host ports).

### Password configuration is fixed with this image

The following commands should work

```bash
docker run --rm --name neo4j_epoc --publish=7777:7687 --env NEO4J_AUTH=neo4j/your_password \
-e NEO4J_apoc_export_file_enabled=true \
-e NEO4J_apoc_import_file_enabled=true \
-e NEO4J_apoc_import_file_use__neo4j__config=true \
--detach jejune:jj_neo4j_docker
```

and we can the password configuration with the running of the standard neo4j container

```bash
docker run --network host neo4j /var/lib/neo4j/bin/cypher-shell -a bolt://127.0.0.1:7777 -u neo4j -p your_password "MATCH () RETURN count(*) as count"
```  

### Note: concerning the user/password neo4j configuration

Un-interestingly enough, the history of configuring the user/password of a neo4j database seems to have been not trivial. Indeed, there are way many outdated posts concerning neo4j password configuration topic e.g.

- [this StackOverflow](https://stackoverflow.com/questions/27645951/how-to-configure-user-and-password-for-neo4j-cluster-without-rest-api) that provides some partial answers/clues.
- [this issue](https://github.com/neo4j/docker-neo4j/issues/114) that concerns neo4j container healthcheck (and that mentions that
  
  ```bash
  [ "CMD", "/var/lib/neo4j/bin/cypher-shell", "-u", "${NEO4J_USERNAME}", "-p", "${NEO4J_PASSWORD}", "MATCH () RETURN count(*) as count" ]
  ```

  command is the way to go).

The difficulty seems to be that the initial temporary password (provided when launching the container) gets changed/invalidated when a UI based process requires the initial password to be changed. And although initializing the password can be done through the UI on first login, it seems hard to realize that password configuration with a simple CLI command...