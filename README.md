# Dspace-python-api
used for blackbox testing, data-ingestion procedures

# How to migrate CLARIN-DSpace5.* to CLARIN-DSpace7.*

### Important:
Make sure that your email server is NOT running because some of the endpoints that are used
are sending emails to the input email addresses. 
For example, when using the endpoint for creating new registration data, 
there exists automatic function that sends email, what we don't want
because we use this endpoint for importing existing data.
```sh
grep mail.server.disabled local.cfg
mail.server.disabled=true

docker compose -p d7ws exec dspace /dspace/bin/dspace dsprop -p mail.server.disabled
true
```

### Prerequisites:
1. Install CLARIN-DSpace7.*. (postgres, solr, dspace backend) - you can use `docker compose`. At this point keep your `local.cfg` minimal, you'll modify it when the migration is done.

2. clone these sources

    2.1. Clone python-api: https://github.com/ufal/dspace-python-api (branch `main`)

    2.2. Clone submodules:
`git submodule update --init libs/dspace-rest-python/`

3. Get database dump (old CLARIN-DSpace) and unzip it into `input/dump` directory in `dspace-python-api` project.


***
4. Go to the `dspace/bin` in dspace7 installation and run the command `dspace database migrate force` (force because of local types).
**NOTE:** `dspace database migrate force` creates default database data that may be not in database dump, so after migration, some tables may have more data than the database dump. Data from database dump that already exists in database is not migrated.

5. Create an admin by running the command `dspace create-administrator` in the `dspace/bin`

***
### Prepare `dspace-python-api` project for migration


```sh
:~$ ls -R ./input

input:
dump

input/dump:
clarin-dspace.sql  clarin-utilities.sql

```
- Create CLARIN-DSpace5.* databases (dspace, utilities) from dump. Either:
  - run `scripts/start.local.dspace.db.bat` or use `scipts/init.dspacedb5.sh` directly with your database. 
  - do the import manually
     ```
      docker compose -p d7ws exec dspacedb createdb -p 5430 --username=dspace --owner=dspace --encoding=UNICODE clarin-dspace
      docker compose -p d7ws exec dspacedb createdb -p 5430 --username=dspace --owner=dspace --encoding=UNICODE clarin-utilities
      cat input/dump/clarin-utilities.sql | docker compose -p d7ws exec -T dspacedb psql -p 5430 --username=dspace clarin-utilities
      cat input/dump/clarin-dspace.sql | docker compose -p d7ws exec -T dspacedbpsql -p 5430 --username=dspace clarin-dspace
      ```
***
- install dependencies for this project (ideally in a python venv)
  ```
  pip install -r requirements.txt
  ```

***
- update `project_settings.py` with the db connection and admin user details
  - You can use an internal backend IP
  - With `"testing": True` the mechanism described in https://github.com/ufal/dspace-migrate/issues/4#issuecomment-3052818633 should kick in (if you also have the mentioned file)
    ```
    docker compose -p d7ws exec dspace bash -c "mkdir -p /tmp/asset && pushd /tmp/asset && curl -LJO https://github.com/user-attachments/files/21145749/57024294293009067626820405177604023574.zip && mkdir -p /dspace/assetstore/57/02/42 && zcat 570242* > /dspace/assetstore/57/02/42/57024294293009067626820405177604023574 && popd && rm -rf /tmp/asset"
    ```
    You don't want this in the final migration
  - Think about the `ignore` section, usually you don't want to ignore eperson `198` (but you might have other that you won't be migrating)
  - If you were using many custom licenses and their text was under `xmlui/page`
    you can use the `licenses` hash to do an automatic udpate of the urls 

***
- Make sure, your backend configuration (`local.cfg`) includes all handle prefixes, that you use, in the `handle.additional.prefixes` property, 
e.g.,`handle.additional.prefixes = 11858, 11234, 11372, 11346, 20.500.12801, 20.500.12800`. Get them from the old database:
  ```
  clarin-dspace=# select distinct(split_part(handle, '/', 1)) as prefix from handle;
  ```

- This project only migrates the data stored in the database(s) not the actual files. Copy `assetstore` from dspace5 to dspace7 (for bitstream import). `assetstore` is in the folder where you have installed DSpace `dspace/assetstore`.

***
### Run the migration
- **NOTE:** database must be up to date (`dspace database migrate force` must be called in the `dspace/bin`)
- **NOTE:** dspace server must be running
- run command `cd ./src && python repo_import.py`
- check the logs (by default) in `__logs` and especially when you see 500 errors check also the dspace.log (on backend in /dspace/log/dspace.log)
- if you need to rerun the migration you simply drop (including the volumes) the compose project (or just the database as suggested in https://github.com/ufal/dspace-migrate/issues/4#issuecomment-3044358816) and recreate the admin account etc. Also consider wiping the migration logs and temp files.
  ```sh
  docker compose -p d7ws down --volumes
  docker compose --env-file .env -p d7ws -f docker/docker-compose.yml -f docker/docker-compose-rest.yml up -d
  docker compose --env-file .env -p d7ws -f docker/docker-compose.yml -f docker/docker-compose-rest.yml -f docker/cli.yml run --rm dspace-cli create-administrator -e test@test.edu -f Sys -l Admin -p password -c en -o UFAL
  docker compose -p d7ws exec dspace bash -c "mkdir -p /tmp/asset && pushd /tmp/asset && curl -LJO https://github.com/user-attachments/files/21145749/57024294293009067626820405177604023574.zip && mkdir -p /dspace/assetstore/57/02/42 && zcat 570242* > /dspace/assetstore/57/02/42/57024294293009067626820405177604023574 && popd && rm -rf /tmp/asset"
  ```
  ```sh
  rm -rf __logs/ src/__temp/ input/tempdbexport_v*
  ```
  

## !!!Migration notes:!!!
- The values of table attributes that describe the last modification time of dspace object (for example attribute `last_modified` in table `Item`) have a value that represents the time when that object was migrated and not the value from migrated database dump.
- If you don't have valid and complete data, not all data will be imported.
- check if license link contains XXX. This is of course unsuitable for production run!

## Check import consistency

Use `tools/repo_diff` utility, see [README](tools/repo_diff/README.md).
