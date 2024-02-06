FROM postgres:14.1-alpine

EXPOSE 5432

COPY scripts/*.sql /docker-entrypoint-initdb.d/
