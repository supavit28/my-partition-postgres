FROM postgres:15
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
RUN apt-get -y install postgresql-15-cron
RUN echo "shared_preload_libraries='pg_cron'" >> /var/lib/postgresql/data/postgresql.conf
RUN echo "cron.database_name='postgres'" >> /var/lib/postgresql/data/postgresql.conf