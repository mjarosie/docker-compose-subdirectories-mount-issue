Repo created to reproduce an issue with docker compose volume mount when configs are in subdirectories.

We've got two subdirectories, each containing a `docker-compose.yaml` configuration with an example service (MySQL database in this case).

Each service mounts its own volume by bind mounting `some-data-dir` from its corresponding subdirectory: `./some-data-dir:/some-data-dir`

`some-data-dir` directories contain `hello-a.txt` and `hello-b.txt` respectively.

That works and gives expected results:

```bash
cd subdirectory-a
docker compose up -d
docker exec -it subdirectory-a-db-a-1 bash -c "ls /some-data-dir" # Outputs: `hello-a.txt`
docker compose down --volumes --remove-orphans
cd ..
cd subdirectory-b
docker compose up -d
docker exec -it subdirectory-b-db-b-1 bash -c "ls /some-data-dir" # Outputs: `hello-b.txt` - that's correct.
docker compose down --volumes --remove-orphans
cd ..
```

That gives unexpected results (run from the root directory):

```bash
docker compose -p test -f subdirectory-a/docker-compose.yaml -f subdirectory-b/docker-compose.yaml up -d
docker exec -it test-db-a-1 bash -c "ls /some-data-dir" # Outputs: `hello-a.txt`
docker exec -it test-db-b-1 bash -c "ls /some-data-dir" # Outputs: `hello-a.txt` - that's not correct!
docker inspect test-db-a-1 | jq .[0].HostConfig.Binds
docker inspect test-db-b-1 | jq .[0].HostConfig.Binds
docker compose -p test -f subdirectory-a/docker-compose.yaml -f subdirectory-b/docker-compose.yaml down --volumes --remove-orphans
```

Invoking `docker inspect test-db-a-1 | jq .[0].HostConfig.Binds` outputs:

```json
["/home/user/path-to-project/subdirectory-a/some-data-dir:/some-data-dir:rw"]
```

Invoking `docker inspect test-db-b-1 | jq .[0].HostConfig.Binds` outputs the same bind mount source directory as `test-db-a-1` service, coming from `subdirectory-a`:

```json
["/home/user/path-to-project/subdirectory-a/some-data-dir:/some-data-dir:rw"]
```
