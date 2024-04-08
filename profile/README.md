# Release Sprint 1

## Changes
- Fixed RabbitMQ messaging connection
- Fixed Frontend configuration
- MetaTrader authentication fix
- Fixed MetaTrader startup script
- TSS saves testresults to MongoDB
  
## Pipeline
All images are built and pushed to GHCR (Github Container Registry).

## Prerequisites
- Install and run Docker Desktop
- Create a PAT (Personal Access Token) within [Github](https://github.com/settings/tokens/new) and run the following command to authenticate with GHCR:
```bash
echo [YOUR-PERSONAL-ACCESS-TOKEN] | docker login ghcr.io -u [YOUR-GITHUB-USERNAMES] --password-stdin
```

## How to run
1. Create a `docker-compose.yml` in a directory and paste the following code inside. 
```yml
version: "3.8"

services:
  minio:
    image: minio/minio
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    networks:
      - auto-trader

  mongodb:
    image: mongo
    container_name: mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    networks:
      - auto-trader

  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - auto-trader

  tss:
    image: ghcr.io/lazytrader-automated-trading/tss:b50f24d118f151f702173622222cb5de077c1672
    container_name: tss
    ports:
      - "5000:5000"
    depends_on:
      - rabbitmq
      - minio
      - mongodb
    environment:
      MINIO_ENDPOINT: minio:9000
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
      MONGODB_BASE: mongodb
      RABBITMQ_HOST: rabbitmq
      MT4_API_BASE: http://metatrader:5175
      PYTHONUNBUFFERED: 1 
    networks:
      - auto-trader
    command: ["sh", "-c", "sleep 20 && python3 app.py"] # added to make sure rabbitmq is ready, 20s usually enough

  metatrader:
    image: ghcr.io/lazytrader-automated-trading/metatrader-4:1f79ffaf28f935ea6027f1ff80d520b6d9aab21f
    container_name: metatrader
    ports:
      - "5175:5175"
    environment:
      - BROKER_URL=amqp://guest:guest@rabbitmq:5672
    depends_on:
      - rabbitmq
    networks:
      - auto-trader
    command: ["sh", "-c", "sleep 20 && ./metatrader"] # added to make sure rabbitmq is ready, 20s usually enough

  frontend:
    image: ghcr.io/lazytrader-automated-trading/angular:f7422ed373a11267eead305556e6ec3e119e1516
    container_name: frontend
    ports:
      - "8080:80"
    depends_on:
      - metatrader
    networks:
      - auto-trader

networks:
  auto-trader:
    name: auto-trader
    driver: bridge

volumes:
  mongo_data:
    name: mongo_data
  minio_data:
    name: minio_data
```

2. In the same directory, open a terminal:
![image](https://github.com/LazyTrader-Automated-Trading/.github/assets/33746255/64c17ca5-6ce8-4d4d-88f2-ff4b1e5b91ac)

3. Run the following command:
![image](https://github.com/LazyTrader-Automated-Trading/.github/assets/33746255/103346ef-9b9e-49b2-8d13-884c5bccebf4)

4. To access the application, visit `http://localhost:8080`

5. To stop the app from running, run `docker-compose down`
