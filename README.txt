Conceptual Overview
1. Containers vs Host
Think of containers like mini-computers running isolated apps. They run on your laptop, but they don’t mess with your system. Docker makes sure each app (like NiFi or MySQL) runs cleanly and separately.

2. Volumes and Mounts
You said: use /mnt/shared_vol/data/ to persist and share data. That’s good! Mounting this path into containers makes sure:

You don’t lose your data if a container restarts.

Containers can share data between each other.

3. Docker Compose
You’ll use one file (docker-compose.yml) to orchestrate all your services. You define all containers and how they talk to each other.

Project Architecture

| Role          | Container | Tech Stack      | Purpose                         |
| ------------- | --------- | --------------- | ------------------------------- |
| Ingest        | `ingest`  | Apache NiFi     | Pull Excel files, send to MySQL |
| Storage       | `storage` | MySQL           | Persist ingested data           |
| Process       | `process` | Spark + Jupyter | Clean, explore, transform data  |
| Orchestration | `orch`    | Apache Airflow  | Automate workflows              |
| Visual        | `visual`  | Apache Superset | BI and dashboards               |


All containers:

Share the volume: /mnt/shared_vol/data

Use a shared Docker network for name resolution like mysql, nifi, etc.

Next Steps
Here’s how we’ll proceed:

1. Docker Network Setup
We'll use bridge network so all containers can talk to each other by name.

2. Volumes Directory Permissions
We’ll prep /mnt/shared_vol/data/ so no chown errors happen.

3. Docker Compose Setup (Step-by-Step)
We’ll build the docker-compose.yml:

Define services

Attach them to the network

Bind volumes

4. Start with One Container
We’ll start with Apache NiFi (your ingest layer), and test login and data flow to MySQL.

Let’s begin by setting up your Apache NiFi (ingestion) and MySQL (storage) containers. This will allow you to:

Log into NiFi from your browser.

Ingest an Excel file.

Store that data in MySQL.

✅ Step 1: Prepare Your Host Directory
Open your terminal and run this to create and fix permissions for shared data:

sudo mkdir -p /mnt/shared_vol/data
sudo chown -R 1000:1000 /mnt/shared_vol/data

This ensures containers can read/write without permission errors. 1000:1000 is a common UID/GID for containerized apps.

✅ Step 2: Create Your Docker Network
Create a shared bridge network for all containers:

docker network create nifi_net

Now all containers on this network can resolve each other by container name.

✅ Step 3: Docker Compose for NiFi + MySQL
Here’s a minimal but complete docker-compose.yml:


networks:
  nifi_net:
    driver: bridge

volumes:
  nifi_data:
  mysql_data:

services:
  mysql:
    image: mysql:8
    container_name: storage
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: nifi_demo
      MYSQL_USER: nifi_user
      MYSQL_PASSWORD: nifi_pass
    volumes:
      - mysql_data:/var/lib/mysql
      - /mnt/shared_vol/data:/mnt/shared
    networks:
      - nifi_net
    ports:
      - "3306:3306"

  nifi:
    image: apache/nifi:latest
    container_name: ingest
    environment:
      NIFI_WEB_HTTP_PORT: 8080
      SINGLE_USER_CREDENTIALS_USERNAME: admin
      SINGLE_USER_CREDENTIALS_PASSWORD: admin123
    ports:
      - "8080:8080"
    volumes:
      - /mnt/shared_vol/data:/mnt/shared
    networks:
      - nifi_net
    restart: unless-stopped

✅ Step 4: Start Your Environment
Run this from the directory with your docker-compose.yml:

docker compose up -d

Then check:

Visit: http://localhost:8080 and log in

Confirm containers are running: docker ps

Test NiFi > MySQL connection from the GUI.



You're absolutely right — and you’re now making the critical distinction between:

A Dockerfile: Instructions to build a custom image.

A Docker Compose file: Instructions to run containers, possibly using an image built from a Dockerfile.

🧠 What’s Happening in Your Case
You're using the official NiFi image, not building your own. That's fine — but the reason HTTP mode isn't working may be because the base image enforces HTTPS unless you fully configure it to avoid secure mode (and NiFi 2.x has tighter defaults).

This is where a custom Dockerfile gives you more control.

✅ Let's Pivot to a Clean, Custom Solution
We'll now:

Create a custom Dockerfile for NiFi that forces HTTP

Use Docker Compose to build and run it

📁 Step-by-Step Structure
Your project will look like:

data_stack/
├── nifi/
│   └── Dockerfile         👈 Our custom NiFi image
├── docker-compose.yml     👈 Compose file that builds and runs NiFi

🛠️ Step 1: Create the Dockerfile
In a new folder nifi/, create Dockerfile:

mkdir -p ~/docker-lab/data_stack/nifi
nano ~/docker-lab/data_stack/nifi/Dockerfile

Paste this:
FROM apache/nifi:latest

ENV NIFI_WEB_HTTP_PORT=8080
ENV NIFI_WEB_HTTP_HOST=0.0.0.0
ENV NIFI_WEB_HTTPS_PORT=
ENV SINGLE_USER_CREDENTIALS_USERNAME=admin
ENV SINGLE_USER_CREDENTIALS_PASSWORD=admin123

🛠️ Step 2: Modify docker-compose.yml
Update the nifi service like this:
  
  nifi:
    build:
      context: ./nifi
    container_name: ingest
    ports:
      - "8080:8080"
    volumes:
      - /mnt/shared_vol/data:/mnt/shared
    networks:
      - nifi_net
    restart: unless-stopped


This tells Docker Compose:

“Use the custom Dockerfile in ./nifi”

“Expose port 8080”

“Use mounted volumes and the shared network”


🏁 Step 3: Rebuild and Run

cd ~/docker-lab/data_stack
docker compose down
docker compose up -d --build


Then open:

http://localhost:8080/nifi