# Lazytrader

LazyTrader Automated Trading resides in the domain of FinTech. It is created for forex traders who use automated strategies to trade and who would like to backtest and optimize said strategies against past market data in an easy and efficient way. It currently supports strategies built on MQL4, the programming language of one of the biggest trading platforms in the world - MetaTrader 4.

### Term definitions

**Strategy:** A trading strategy is a file containing code designed to achieve a profitable return by going long or short in markets, written in the MQL4 language, the programming language of one of the biggest trading platforms in the world - MetaTrader 4

**Backtest:** A backtest is essentially an automated test, a way to evaluate the effectiveness of a trading strategy by running it against historical data to see how it would have performed.

## Repositories

This project consists of multiple repositories, each containing software which is responsible for a certain part of the process. The repositories this project consists of are:

**angular-app**

This repository contains the front-end application responsible for displaying the UI to the enduser. This application connects to the backend, allowing the user to perform tasks such as: uploading a strategy, runnning backtests, viewing backtest results, etc.

This application was built using the Angular framework, using the Typescript language and Tailwind CSS for the styling.

**TradingStrategyService**

This repository contains the backend which is the intermediary between the front-end application and various other services. It is the application through which the front-end communicates and interacts with the rest of the system. It is a Flask application, which is a micro-webframework for the Python language.

This application connects with the following software:
- MongoDB, which serves as the main database of the application. This database contains all the strategies' references a user has uploaded, the expertparameters a user uploaded and the backtests a user has run.
- Minio, which serves as blobstorage contains the strategies themselves. The data on the strategies in the MongoDB database are used to retrieve the correct strategyfiles from Minio.
- RabbitMQ, a messaging broker which will be used to store queue items and messages to certain topics.
- Metatrader-4, in some cases TradingStrategyService may need to connect to the MetaTrader application directly in order to upload strategy- and/or expertparameterfiles to the container. When a user wants to start a backtest, this service will add an item to the backtest-queue containing the backtest data.

Whenever something needs to happen in one of the above services, it will go through the TradingStrategyService to achieve that.

**Metatrader-4**

This repository contains the MetaTrader installation, which is the software which is responsible for running the backtests. When a backtest is done, it will generate results which can then be used to display the results to the enduser.

In order to communicate with the MetaTrader installation, there is a file containing some endpoints which will run some tasks on the container in which this app is running. This application also contains a consumer to the backtest-queue which will consume the items in the backtest-queue and start a backtest using that data, if any items are present.

When a test is finished it will send the results of the backtest to the results-queue, which the frontend will be listening for. This application will also send status updates to the same queue when the status of a backtest changes, as it can be `pending`, `running`, `succeeded` or `error`.

**QuantDataService**

In order to run a backtest at all, the Metatrader installation will need historical tradedata it can use. The QuantDataService application is responsible for supplying this data. It can be used to download data from certain periods and place it into a directory which the Metatrader installation can access.

This repository contains a Python API with endpoints which will run commands that achieve this on the QuantDataService application. The commands can get all symbols, update a symbol, add a symbol, etc. The repository also contains a cronjob that will run every 12 hours. This cronjob will update the symbols that are present in the database, in order to keep it up-to-date with the symbols given by QuantDataService. The API is built using fastapi which is a webframework for the Python language.

## Features

A range of features have already been implemented thus far. Below is a list of features that are already present in the application:

- **Login and registration**
    - The user can register to the application using an email and password. After registering, the user can log in using the same credentials. This is handled by an external service called Firebase.
- **Uploading a strategy**
    - The user can upload a strategy to the application, which will be stored in the Minio blobstorage. This strategy will be stored in the MongoDB database as a reference.
- **View all strategies**
    - The user can view all of their uploaded strategies in a list, which are loaded in from a database. This view will allow the user to choose which strategy they want to run a backtest with.
- **Uploading an expertparameter**
    - The user can upload an expertparameter for a specific strategy, which will be stored in MongoDB. It will be saved as an object with the `filename` and `settings` properties.
- **Running a backtest**
    - The user can run a backtest with the uploaded strategy and expertparameter. The user can select a period using a start- and enddate and a symbol to run the backtest on. The backtest will be stored in the MongoDB database. After a test has started, the app wil display the status of a backtest.
- **Viewing backtest status**
    - The user can view the status of a backtest they ran. The status can be `pending`, `running`, `succeeded` or `error`. This status will be updated live, as those changes are happening.
- **Viewing backtest results**
    - The user can view the backtest results of a backtest they ran. The backtest results will be displayed, showing the profit, Win Rate and more. It also shows the expertparameter settings the backtest was run with.

## Architecture
The platform is built using a microservices architecture. The platform consists of the following services:

- [TradingStrategyService](https://github.com/LazyTrader-Automated-Trading/TradingStrategyService) - The main Rest API service that allows users to upload their trading strategies and backtest them.
  - **MongoDB** is used to store the trading strategies and backtest results.
  - **Minio** is used to store the trading strategy files (blob storages).
- [Metatrader-4](https://github.com/LazyTrader-Automated-Trading/Metatrader-4) - A service that runs the Metatrader 4 platform and exposes an API for the TradingStrategyService to interact with.
- [Frontend](https://github.com/LazyTrader-Automated-Trading/angular-app) - The frontend application that allows users to interact with the platform (made with Angular).
- [QuantDataService](https://github.com/LazyTrader-Automated-Trading/QuantDataService) - A service that provides historical data for the backtesting process.
- *Rabbitmq* - A message broker that allows the TradingStrategyService to communicate with the Metatrader-4 service.

Architecture Diagram:

![image](./c4-lazytrader-Container%20diagram.drawio.png)


## Pipeline
All images are built and pushed to GHCR (Github Container Registry).

## Prerequisites
- Install and run Docker Desktop (or Docker Engine) on your machine.
- Create a PAT (Personal Access Token) within [Github](https://github.com/settings/tokens/new) with the `read:packages` scope and run the following command to authenticate with GHCR:
```bash
echo [YOUR-PERSONAL-ACCESS-TOKEN] | docker login ghcr.io -u [YOUR-GITHUB-USERNAMES] --password-stdin
```

## How to run

> # IMPORTANT: Update the image tags in the docker-compose file BEFORE sprint release

1. Create a `docker-compose.yml` in a directory and paste the following code inside:

<details>
  <summary>Click me to show the docker-compose content</summary>

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
</details>

2.  In the same directory, open a terminal and Run the following command:
![image](https://github.com/LazyTrader-Automated-Trading/.github/assets/33746255/103346ef-9b9e-49b2-8d13-884c5bccebf4)

3. To access the application, visit `http://localhost:8080`

4. To stop the app from running, run `docker-compose down`
