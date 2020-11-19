# Docker-compose
### 參考網頁: https://blog.techbridge.cc/2018/09/07/docker-compose-tutorial-intro/
### 參考網頁: https://blog.maxkit.com.tw/2017/03/docker-compose.html


## 使用 Compose 有基本的三個處理步驟：

### 1.使用 Dockerfile 定義你的 app 環境，讓它可以在任何地方都能複製(reproduced)。
### 2.使用 docker-compose.yml 定義你的服務，讓他們可以在獨立環境內一起執行。
### 3.最後，執行 docker-compose up，Compose 將會開始與執行你所有的 app。

docker-compose.yml 也就是組態設定文件，是一種 yaml 格式撰寫的文件，可以上維基看一下 YAML 格式說明，比較需要注意的是他的縮排規定要空白鍵，而不是 tab。

## Docker Compose 是一個工具可以讓你可以透過一個指令就可以控制所有專案（project）中所需要的 services。
### 基礎模板
```
version: '3'  # 目前使用的版本，可以參考官網：
services:  # services 關鍵字後面列出 web, redis 兩項專案中的服務
  web:
    build: . # Build 在同一資料夾的 Dockerfile（描述 Image 要組成的 yaml 檔案）成 container
    ports:
      - "5000:5000" # 外部露出開放的 port 對應到 docker container 的 port
    volumes:
      - .:/code # 要從本地資料夾 mount 掛載進去的資料
    links:
      - redis # 連結到 redis，讓兩個 container 可以互通網路
  redis:
    image: redis # 從 redis image build 出 container
```

### 撰寫yml檔
```
version: '2' #版本
services: #伺服器
  flask-chatbot.pri: #自己建立的images
    build:  
      context: . #放dockerfile路徑
      dockerfile: dockerfile
    container_name: flask-chatbot #容器名 相對於 --name <名字>
    ports: #(host port: continer port)
      - "5000:5000"
    environment: #環境設置 會自動讀取.env文件 將需要的鑰匙放入.env內
      line_channel_access_token: '${line_channel_access_token}'
      line_channel_secret: '${line_channel_secret}'
      TZ: 'Asia/Taipei'
    volumes: # 掛載建立的 資料夾
      - .:/app
    command: python kafka_welcome_linebot.py #執行app目錄下的檔案
    depends_on: #啟動此容器前先啟動容器的先決條件
      - mysql
      - redis
    networks: #每個容器互相訪問的網路
      - es7net

  ngrok-temp: #
    image: wernight/ngrok #網路掛載的image
    container_name: ngrok #容器名 相對於 --name <名字>
    ports: #(host port: continer port)
      - "4040:4040"
    environment: # NGROK_PORT:(continer name:post)
      NGROK_AUTH: '${NGROK_AUTH}'
      NGROK_USERNAME: '${NGROK_USERNAME}'
      NGROK_PASSWORD: '${NGROK_PASSWORD}'
      NGROK_PORT: flask-chatbot.pri:5000 
      NGROK_REGION: 'ap'
      TZ: 'Asia/Taipei'
    depends_on: #啟動此容器前先啟動容器的先決條件
      - flask-chatbot.pri
    networks: #每個容器互相訪問的網路
      - es7net

  mysql:
    image: mysql:8.0 #image:版本號
    container_name: mysql
    hostname: mysql
    command: --default-authentication-plugin=mysql_native_password #新版讀取密碼
    ports:
      - "3307:3306" #防止本地環境互搶port
    environment:
      - TZ=Asia/Taipei
      - MYSQL_ROOT_PASSWORD='${MYSQL_ROOT_PASSWORD}'
    volumes: #掛載 (host 相對路徑: continer 相對路徑)
      - ./data/mysql_db/mysql_data:/var/lib/mysql #資料儲存的路徑
      - ./data/mysql_init:/docker-entrypoint-initdb.d/
    networks:
      - es7net

  zookeeper:
    image: confluentinc/cp-zookeeper:5.2.1
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment: 
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000  #2000ms 
    depends_on:
      - flask-chatbot.pri
    networks:
      - es7net
    
  kafka: #zookeeper是kafka的中控台
    image: confluentinc/cp-kafka:5.2.1
    hostname: kafka
    container_name: kafka
    ports: #9092 對外網 29092 對內網
      - '9092:9092'
      - '29092:29092'
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181 
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      ## for local use
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      ## for public use
      #KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://<host_ip>:9092
      #advertised.host.name: <host_ip>
    networks:
      - es7net

  redis:
    image: 'bitnami/redis:latest'
    container_name: redis
    hostname: redis
    restart: always #重新啟動 :隨時
    ports:
      - 6379:6379
    environment:
      - REDIS_PASSWORD='${REDIS_PASSWORD}'
    volumes:
      # HOST:CONTAINER
      - ./data/redis/data:/usr/share/redis/data
    networks:
      - es7net
networks:
  es7net:

```
# Dockerfile

### 參考網頁: https://ithelp.ithome.com.tw/articles/10191139
### 事實上 Dockerfile 是用來描述映像檔（image）的文件。

```
# 這是一個創建 ubuntu 並安裝 nginx 的 image
FROM ubuntu:16.04 # 從 Docker hub 下載基礎的 image，可能是作業系統環境或是程式語言環境，這邊是 ubuntu 16.04
MAINTAINER demo@gmail.com # 維護者

RUN apt-get update # 執行 CMD 指令跑的指令，更新 apt 套件包資訊
RUN apt-get install –y nginx # 執行 CMD 指令跑的指令，安裝 nginx
CMD ["echo", "Nginx Image created"]
```
