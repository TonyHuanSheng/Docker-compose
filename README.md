# Docker-compose
### 參考網頁: https://blog.techbridge.cc/2018/09/07/docker-compose-tutorial-intro/


## Docker Compose 是一個工具可以讓你可以透過一個指令就可以控制所有專案（project）中所需要的 services。
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

### 事實上 Dockerfile 是用來描述映像檔（image）的文件。

```
# 這是一個創建 ubuntu 並安裝 nginx 的 image
FROM ubuntu:16.04 # 從 Docker hub 下載基礎的 image，可能是作業系統環境或是程式語言環境，這邊是 ubuntu 16.04
MAINTAINER demo@gmail.com # 維護者

RUN apt-get update # 執行 CMD 指令跑的指令，更新 apt 套件包資訊
RUN apt-get install –y nginx # 執行 CMD 指令跑的指令，安裝 nginx
CMD ["echo", "Nginx Image created"]
```
