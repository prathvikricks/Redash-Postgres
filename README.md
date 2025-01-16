# Documentation: PostgreSQL and Redash Setup with Troubleshooting

## Overview
This document outlines the step-by-step process of setting up PostgreSQL 15 and Redash using Docker, along with common errors encountered during the process and their resolutions.

---

### pg_hub.conf file for Main postgres
This is used to enable postgres to listen user named redash and listening to any IP.
```yaml
host    mydb    redash    0.0.0.0/0    md5
```

### postgresql.conf file for Main postgres which makes postgres to listen to every IP again
This is used to enable postgres listening to any IP.
```yaml
listen_addresses = '*'
```

## PostgreSQL Setup
### Docker Compose File for PostgreSQL
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: postgres_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgrespassword
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - redash-network

volumes:
  postgres_data:


networks:
  redash-network:
    name: redash-network
```

### Steps to Set Up PostgreSQL
1. Create the Docker container:
   ```bash
   docker-compose up -d
   ```
2. Verify the PostgreSQL container is running:
   ```bash
   docker ps
   ```
3. Access the database using:
   - **From host:**
     ```bash
     psql -h localhost -p 5432 -U postgres -d mydb
     ```
   - **Directly inside the container:**
     ```bash
     docker exec -it postgres_db psql -U postgres -d mydb
     ```

---

## Sample Data Creation

### Sample Data
```sql
-- Create tables
CREATE TABLE IF NOT EXISTS employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100),
    salary DECIMAL(10,2),
    hire_date DATE
);

CREATE TABLE IF NOT EXISTS departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    budget DECIMAL(15,2)
);

-- Insert sample data
INSERT INTO departments (name, budget) VALUES
    ('Engineering', 1000000),
    ('Marketing', 500000),
    ('Sales', 750000),
    ('HR', 300000);

INSERT INTO employees (name, department, salary, hire_date) VALUES
    ('John Doe', 'Engineering', 85000, '2022-01-15'),
    ('Jane Smith', 'Marketing', 65000, '2022-02-01'),
    ('Bob Johnson', 'Sales', 75000, '2022-03-10'),
    ('Alice Brown', 'Engineering', 90000, '2022-04-05'),
    ('Charlie Wilson', 'HR', 55000, '2022-05-20');
```

## Redash Setup

### Docker Compose File for Redash
```yaml
redash-docker-compose.yml
version: '3.8'

services:
  server:
    image: redash/redash:latest
    container_name: redash_server
    command: server
    depends_on:
      - postgres_redash
      - redis
    ports:
      - "5000:5000"
    environment:
      REDASH_LOG_LEVEL: "INFO"
      REDASH_REDIS_URL: "redis://redis:6379/0"
      REDASH_DATABASE_URL: "postgresql://redash:redashpassword@postgres_redash:5432/redash"
      PYTHONUNBUFFERED: 0
    networks:
      - redash-network

  worker:
    image: redash/redash:latest
    container_name: redash_worker
    command: scheduler
    depends_on:
      - server
    environment:
      REDASH_LOG_LEVEL: "INFO"
      REDASH_REDIS_URL: "redis://redis:6379/0"
      REDASH_DATABASE_URL: "postgresql://redash:redashpassword@postgres_redash:5432/redash"
      PYTHONUNBUFFERED: 0
    networks:
      - redash-network

  redis:
    image: redis:6-alpine
    container_name: redash_redis
    restart: unless-stopped
    networks:
      - redash-network

  postgres_redash:
    image: postgres:15
    container_name: postgres_redash
    environment:
      POSTGRES_USER: redash
      POSTGRES_PASSWORD: redashpassword
      POSTGRES_DB: redash
    volumes:
      - redash_postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - redash-network

volumes:
  redash_postgres_data:

networks:
  redash-network:
    name: redash-network
```

### Steps to Set Up Redash
1. Start the Redash services:
   ```bash
   docker-compose -f redash-docker-compose.yml up -d
   ```
2. Initialize the Redash database:
   ```bash
   docker-compose -f redash-docker-compose.yml run --rm server create_db
   ```
3. Access Redash at: [http://localhost:5000](http://localhost:5000)

### Connect PostgreSQL to Redash
1. Go to **Settings > Data Sources > New Data Source**.
2. Select "PostgreSQL" and enter the following details:
   - **Name:** My PostgreSQL
   - **Host:** postgres_db
   - **Port:** 5432
   - **Database:** mydb
   - **User:** postgres
   - **Password:** postgrespassword

---

## Common Errors and Fixes

### Error: `relation "organizations" does not exist`
#### Cause:
Redash database schema not initialized.
#### Solution:
Run the following command to create the schema:
```bash
docker-compose -f redash-docker-compose.yml run --rm server create_db
```

### Error: Cannot connect to PostgreSQL from Redash
#### Cause:
Incorrect connection details.
#### Solution:
1. Verify the PostgreSQL container is running:
   ```bash
   docker ps
   ```
2. Ensure the hostname (`postgres_db`) and credentials match the main PostgreSQL instance.

---

## Summary
This setup separates the main PostgreSQL database (`postgres_db`) and Redash's internal PostgreSQL instance (`postgres_redash`) for optimal functionality. By following these steps, you can successfully integrate PostgreSQL with Redash to visualize and analyze your data.

