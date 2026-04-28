## LibreChat
- We will do a full quadlet system service setup to guarantee that the services (containers) are running on a system reboot
### Podman setup
  - network quadlet /etc/containers/systemd/cont-librechat-net.network
    ```
      [Unit]
      Description=Network for LibreChat Stack

      [Network]
      NetworkName=librechat-net
    ```
  - MongoDB
    - create the directory
      ``` # mkdir -p /opt/librechat/mongodb ```
    - quadlet /etc/containers/systemd/cont-librechat-mongodb.container
    ```
      [Unit]
      Description=LibreChat MongoDB Service
      After=network-online.target

      [Container]
      ContainerName=chat-mongodb
      Image=docker.io/mongo:latest

      # ':Z' for SELinux labeling
      Volume=/opt/librechat/mongodb:/data/db:Z

      # Networking
      Network=cont-librechat-net.network

      # The command to run
      Exec=mongod --noauth

      [Service]
      Restart=always

      [Install]
      WantedBy=multi-user.target
    ```
  - MeiliSearch
    - create the directory
    ```
      # mkdir -p /opt/librechat/meilisearch
    ```
    - quadlet /etc/containers/systemd/cont-librechat-meilisearch.container
    ```
      [Unit]
      Description=LibreChat Meilisearch (Keyword Search)
      After=network-online.target

      [Container]
      ContainerName=chat-meilisearch
      Image=docker.io/getmeili/meilisearch:latest

      # The ':Z' is vital for your /opt/ folder on Linux
      Volume=/opt/librechat/meilisearch:/meili_data:Z

      # Environment
      # Reference the central .env file
      EnvironmentFile=/opt/librechat/.env

      # Networking
      Network=cont-librechat-net.network

      [Service]
      Restart=always

      [Install]
      WantedBy=multi-user.target
    ```
  - PGVector
    - create the directory
    ```
      # mkdir -p /opt/librechat/pgvector
    ```
    - quadlet /etc/containers/systemd/cont-librechat-pgvector.container
    ```
      [Unit]
      Description=LibreChat pgvector Database (Semantic Storage)
      After=network-online.target

      [Container]
      ContainerName=chat-vectordb
      Image=docker.io/pgvector/pgvector:pg15-trixie

      # Environment
      # Reference the central .env file
      EnvironmentFile=/opt/librechat/.env

      # Persistence mapping
      Volume=/opt/librechat/pgvector:/var/lib/postgresql/data:Z

      # Networking
      Network=cont-librechat-net.network

      # Healthcheck: Ensures Postgres is ready before the RAG API tries to talk to it
      HealthCmd=pg_isready -U myuser -d mydatabase || exit 1
      HealthInterval=10s
      HealthTimeout=5s
      HealthRetries=5

      [Service]
      Restart=always

      [Install]
      WantedBy=multi-user.target
    ```
  - RAG_API
    - update the .env file
    ```
      # touch /opt/librechat/.env
    ```
    - quadlet /etc/containers/systemd/cont-librechat-rag-api.container
    ```
      [Unit]
      Description=LibreChat RAG API
      # Ensures pgvector is healthy before this starts
      After=cont-librechat-pgvector.service
      Wants=cont-librechat-pgvector.service

      [Container]
      ContainerName=chat-rag_api
      Image=registry.librechat.ai/danny-avila/librechat-rag-api-dev-lite:latest

      # Environment
      # Reference the central .env file
      EnvironmentFile=/opt/librechat/.env

      # Networking
      Network=cont-librechat-net.network

      [Service]
      Restart=always

      [Install]
      WantedBy=multi-user.target
    ```
  - LibreChat
    - create the librechat.yaml
    ```
      # touch /opt/librechat/librechat.yaml
    ```
    - create the directory
    ```
      # mkdir -p /opt/librechat/librechat/{images,uploads,logs}
      # chown -R 1000:1000 /opt/librechat/librechat/*
    ```
    - quadlet /etc/containers/systemd/cont-librechat-api.container
    ```
      [Unit]
      Description=LibreChat Main UI & API Service
      # Wait for the databases and the librarian to be ready
      After=cont-librechat-mongodb.service cont-librechat-rag-api.service
      Wants=cont-librechat-mongodb.service cont-librechat-rag-api.service

      [Container]
      ContainerName=chat-api
      Image=registry.librechat.ai/danny-avila/librechat-dev:latest

      # Bypass SeLinux or crash
      SecurityLabelDisable=true

      # Podman adds host.docker.internal automatically
      AddHost=host.docker.internal:host-gateway

      # Persistence mapping
      Volume=/opt/librechat/librechat/images:/app/client/public/images:Z
      Volume=/opt/librechat/librechat/uploads:/app/uploads:Z
      Volume=/opt/librechat/librechat/logs:/app/logs:Z
      # Main config file
      Volume=/opt/librechat/librechat.yaml:/app/librechat.yaml:Z

      # Environment
      # Reference the central .env file
      EnvironmentFile=/opt/librechat/.env

      # Network & Exposure
      PublishPort=3080:3080
      Network=cont-librechat-net.network

      [Service]
      Restart=always

      [Install]
      WantedBy=multi-user.target
    ```
