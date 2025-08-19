# Ansible Role Plan: Deploy Immich v1.131.3 with community.docker

This plan outlines the steps and structure for an Ansible Role to deploy Immich v1.131.3 and its dependencies using only `community.docker` modules, integrating NVIDIA hardware acceleration.

**Confirmed Configuration:**

*   **Immich Version:** v1.131.3
*   **Hardware Acceleration:** NVIDIA (CUDA for ML, NVENC for Transcoding)
    *   Target host has NVIDIA Container Runtime installed.
    *   ML Service: `immich-machine-learning`
    *   Transcoding Service: `immich-microservices`
    *   GPU Devices: `all`
*   **Paths & Ports:**
    *   Upload Location (Host): `/mnt/data/immich/upload`
    *   Web UI Port (Host): `2283`
*   **Credentials:** Managed via Ansible Vault (e.g., `vault_immich_db_password`).
*   **Task File Naming:** Use `_` prefix for internal task files.

## 1. Ansible Role Structure (`roles/immich`)

```
roles/
└── immich/
    ├── defaults/
    │   └── main.yml       # Default variables (version, ports, paths, GPU, vault placeholders)
    ├── tasks/
    │   ├── main.yml       # Main task entrypoint, includes other files in order
    │   ├── _vars.yml      # Set internal facts (e.g., image tags based on GPU usage)
    │   ├── _networks.yml  # Create Docker network (docker_network)
    │   ├── _volumes.yml   # Create Docker volumes (docker_volume)
    │   ├── _database.yml  # Deploy PostgreSQL (docker_container)
    │   ├── _redis.yml     # Deploy Redis (docker_container)
    │   ├── _machine_learning.yml # Deploy ML (docker_container, with GPU)
    │   ├── _microservices.yml # Deploy Microservices (docker_container, with GPU transcoding)
    │   ├── _server.yml    # Deploy Server (docker_container)
    │   └── _web.yml       # Deploy Web UI (docker_container)
    └── vars/
        └── main.yml       # (Optional) Role internal fixed variables
```

## 2. Key Variables (`defaults/main.yml` - Example)

```yaml
# Immich Configuration
immich_version: "v1.131.3"
immich_timezone: "Asia/Shanghai" # Adjust to your timezone

# Network and Port Configuration
immich_network_name: "immich_network"
immich_web_port: 2283

# Volume and Path Configuration
immich_upload_location: "/mnt/data/immich/upload" # Host path for uploads
immich_pg_volume_name: "immich_pgdata"
immich_redis_volume_name: "immich_redisdata"
immich_model_cache_volume_name: "immich_model_cache"

# Database Configuration (Use Ansible Vault for password)
immich_db_user: "postgres"
immich_db_name: "immich"
immich_db_password: "{{ vault_immich_db_password }}" # Define in Vault

# Image Configuration
immich_server_image: "ghcr.io/immich-app/immich-server"
immich_ml_image: "ghcr.io/immich-app/immich-machine-learning"
immich_microservices_image: "ghcr.io/immich-app/immich-microservices"
immich_web_image: "ghcr.io/immich-app/immich-web"
immich_redis_image: "docker.io/redis:6.2-alpine@sha256:148bb5411c184abd288d9aaed139c98123eeb8824c5d3fce03cf721db58066d8"
immich_db_image: "docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:739cdd626151ff1f796dc95a6591b55a714f341c737e27f045019ceabf8e8c52"

# Hardware Acceleration (NVIDIA)
immich_ml_use_gpu: true
immich_ml_gpu_runtime: "nvidia"
immich_ml_gpu_devices: "all" # Or specify GPU UUID list ['gpu-uuid1', 'gpu-uuid2']
immich_ml_gpu_capabilities: ["gpu"]

immich_transcoding_use_gpu: true
immich_transcoding_gpu_runtime: "nvidia"
immich_transcoding_gpu_devices: "all" # Or specify GPU UUID list
immich_transcoding_gpu_capabilities: ["gpu", "compute", "video"]
```

## 3. Task Execution Flow (`tasks/`)

1.  **`main.yml`:** Includes tasks in the following order.
2.  **`_vars.yml`:** Sets facts like `immich_ml_image_tag` (e.g., `{{ immich_version }}-cuda` if `immich_ml_use_gpu` is true) and prepares `device_requests` structure for GPU containers.
3.  **`_networks.yml`:** Creates the Docker network `{{ immich_network_name }}`.
4.  **`_volumes.yml`:** Creates named Docker volumes (`pgdata`, `redisdata`, `model-cache`).
5.  **`_database.yml`:** Deploys `immich_postgres` container, mapping `pgdata` volume, setting environment variables (user, password, db name, init args), command, healthcheck, and restart policy.
6.  **`_redis.yml`:** Deploys `immich_redis` container, mapping `redisdata` volume, setting healthcheck, and restart policy.
7.  **`_machine_learning.yml`:** Deploys `immich_machine_learning` container. Uses GPU-specific image tag if enabled. Maps `model-cache` volume. Sets environment variables. Configures `runtime` and `device_requests` for GPU access if `immich_ml_use_gpu` is true. Sets restart policy and healthcheck.
8.  **`_microservices.yml`:** Deploys `immich_microservices` container. Maps upload volume and localtime. Sets environment variables. Configures `runtime` and `device_requests` for GPU access (transcoding) if `immich_transcoding_use_gpu` is true. Sets restart policy and healthcheck.
9.  **`_server.yml`:** Deploys `immich_server` container. Maps web port, upload volume, and localtime. Sets environment variables. Sets restart policy and healthcheck. Depends implicitly on DB and Redis being up (consider adding explicit waits if needed).
10. **`_web.yml`:** Deploys `immich_web` container. Sets environment variables (pointing to server). Sets restart policy and healthcheck.

## 4. Service Dependencies and Startup

The sequential execution in Ansible provides basic dependency management. For stricter guarantees (e.g., ensuring DB/Redis are fully healthy before starting dependent services), `community.docker.docker_container_info` combined with `until` loops can be added before starting `_machine_learning.yml`, `_microservices.yml`, and `_server.yml`.

## 5. Mermaid Diagram (Service Relationships)

```mermaid
graph TD
    subgraph "Ansible Managed Docker Resources"
        direction LR
        subgraph "Network: {{ immich_network_name }}"
            DB(immich_postgres)
            REDIS(immich_redis)
            ML(immich_machine_learning)
            MS(immich_microservices)
            SRV(immich_server)
            WEB(immich_web)
        end

        subgraph "Volumes"
            PG_VOL[({{ immich_pg_volume_name }})]
            REDIS_VOL[({{ immich_redis_volume_name }})]
            CACHE_VOL[({{ immich_model_cache_volume_name }})]
            UPLOAD_PATH[/mnt/data/immich/upload]
        end

        subgraph "Host System"
            GPU[NVIDIA GPU (all)]
            PORT[Port {{ immich_web_port }}]
        end

        DB -- Volume --> PG_VOL
        DB -- Network --> REDIS
        REDIS -- Volume --> REDIS_VOL

        ML -- Network --> DB
        ML -- Network --> REDIS
        ML -- Volume --> CACHE_VOL
        ML -- GPU (CUDA) --> GPU

        MS -- Network --> DB
        MS -- Network --> REDIS
        MS -- Volume --> UPLOAD_PATH
        MS -- GPU (NVENC) --> GPU

        SRV -- Network --> DB
        SRV -- Network --> REDIS
        SRV -- Volume --> UPLOAD_PATH
        SRV -- Port --> PORT

        WEB -- Network --> SRV
    end

    style GPU fill:#77cc77,stroke:#333,stroke-width:2px
    style PORT fill:#77ccff,stroke:#333,stroke-width:2px
    style UPLOAD_PATH fill:#cc77cc,stroke:#333,stroke-width:2px