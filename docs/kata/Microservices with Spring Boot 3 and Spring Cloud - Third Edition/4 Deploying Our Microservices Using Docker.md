# 4. Deploying Our Microservices Using Docker

[JVM docker container 的資源使用](4.1 jvm docker container.md)

## 新增 docker 環境的設定

- 設定在 `profile` 為 `docker` 時套用的設定
    - 透過 `spring.config.activate.on-profile: docker` 或 `application-docker.yml`
- 使用 `8080` 預設的 port。
- product-composite-service 相依 microservices 的 host 與 port
    - product-service (host: `product`, port: `8080`)
    - recommendation-service (host: `recommendation`, port: `8080`)
    - review-service (host: `review`, port: `8080`)

## 撰寫 `Dockerfile`

每一個 service 的建置腳本基本相同，除了 jar 需要變動以外。將 `Dockerfile` 檔案放在跟目錄，透過帶入 `DIR` 參數來建置不同的 microservice 專案。

```dockerfile title="Dockerfile"
FROM eclipse-temurin:21-jre as builder
WORKDIR /builder
ARG DIR
ADD ${DIR}/build/libs/*.jar application.jar
RUN java -Djarmode=tools -jar application.jar extract --layers --destination extracted

FROM eclipse-temurin:21-jre
WORKDIR application
COPY --from=builder /builder/extracted/dependencies/ ./
COPY --from=builder /builder/extracted/spring-boot-loader/ ./
COPY --from=builder /builder/extracted/snapshot-dependencies/ ./
COPY --from=builder /builder/extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "application.jar"]
```

參考: [Spring - Dockerfiles](https://docs.spring.io/spring-boot/reference/packaging/container-images/dockerfiles.html)

編譯 docker image:

```shell
$ ./gradlew clean build
$ docker-compose build --build-arg DIR=product-service .
$ docker-compose build --build-arg DIR=recommendation-service .
$ docker-compose build --build-arg DIR=review-service .
$ docker-compose build --build-arg DIR=product-composite-service .
```

## 撰寫 `compose.yml`

```yml title="compose.yml"
services:
  product:
    build:
      context: .
      args:
        DIR: product-service
    mem_limit: 512m
    environment:
      - SPRING_PROFILES_ACTIVE=docker
  recommendation:
    build:
      context: .
      args:
        DIR: recommendation-service
    mem_limit: 512m
    environment:
      - SPRING_PROFILES_ACTIVE=docker
  review:
    build:
      context: .
      args:
        DIR: review-service
    mem_limit: 512m
    environment:
      - SPRING_PROFILES_ACTIVE=docker
  product-composite:
    build:
      context: .
      args:
        DIR: product-composite-service
    mem_limit: 512m
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
```

## 使用 docker 執行 microservices

啟動 services

```shell
$ docker-compose up --build -d
```

測試 API

```shell
$ curl "http://localhost:8080/product-composite/1" -s | jq .

{
  "productId": 1,
  ...
  "serviceAddresses": {
    "cmp": "bc46f4a412fd/192.168.48.5:8080",
    "pro": "f99941be89aa/192.168.48.4:8080",
    "rev": "3cc79fa11ae5/192.168.48.3:8080",
    "rec": "7cb5c2bceb5e/192.168.48.2:8080"
  }
}
```

停止 services

```shell
$ docker-dompose down --rmi all
```

## 修改 automation script

- 指令參數有 `start` 時，在腳本開始前，使用 docker compose 啟動 microservices

    ```shell
    for arg in "$@"; do
        if [ "$arg" = "start" ]; then
            docker-compose up -d --build
        fi
    done
    ```

- 指令參數有 `stop` 時，在腳本結束前，將 microservices 結束
- 修改連接的 service 位址: `http://localhost:8080`
- docker 啟動後，到 service 可用前，會需要一段時間。寫一個等待的腳本，確保 request 可用之前，才開始跑測試。

    ```shell
    $ curl http://${HOST}:${PORT}/product-composite/1 --retry 10 --retry-delay 5 --retry-all-errors -s > /dev/null
    ```
