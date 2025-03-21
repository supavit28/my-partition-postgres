# Postgres Cron (pg_cron)

Reference :

- https://stackoverflow.com/questions/61582682/installing-and-using-pg-cron-extension-on-postgres-running-inside-of-docker-cont
- https://www.nico.fyi/blog/dockerfile-for-pgcron-postgres-cron-job

## Steps

1. Run script

   ```sh
   docker build -t postgres-with-cron .
   ```

2. Go to workdir

3. Run

   ```
   docker run  \
   --name postgres-cron \
   -e POSTGRES_USER=myuser \
   -e POSTGRES_PASSWORD=mysecretpassword \
   -v ${PWD}/data:/var/lib/postgresql/data \
   -p 5432:5432 \
   postgres-with-cron
   ```

   Where `/data` on your current directory will be config path for postgres.

4. Run

   ```
    psql -h localhost -U dev -d postgres
   ```

   to access to local Postgres (there will be a prompt to enter password)

5. Create sample database

   ```
    CREATE DATABASE sample OWNER myuser
   ```

6. Change your database config as proper

   Such as

   Change database that want to enable cron (default is postgres) -> change to `sample`

   ```
   cron.database_name='sample'
   ```

7. Now database `sample` can be scheduled by using **pg_cron.**

8. To change database just switch `cron.database_name` to another database name.
