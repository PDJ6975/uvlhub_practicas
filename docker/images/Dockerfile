FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends mariadb-client

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000


