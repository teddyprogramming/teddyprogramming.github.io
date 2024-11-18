---
categories:
    - Spring Boot
    - Kubernetes
    - Docker
date: 2024-11-18
---

# Run appliocation on kubernetes

我們會需要在 kubernets 的 cluster 試跑一下程式是否能夠如預期般的運作。像是特殊的網路環境，或者是內部的 service，我們想確認部署上去的程式是否能打通另一個 service。

<!-- more -->
這裡我們假定使用 Sping Boot 進行開發。為了把 app 放到 kubernetes 上執行，我們需要進行以下動作:


- 將 Spring Boot 的 app 打包成 image
- 將 image 推上 image hub (例如 dockerhub)

## 將 Spring Boot 打包成 image

執行 `gradle build` 後，程式的 jar 會產生在 build/lib 目錄下。透過 jar 就可以將程式跑起來。以下 Dockerfile，將此 jar 打包成 image，當跑起來時，執行該 jar。

```dockerfile title="Dockerfile"
FROM openjdk:21
COPY build/libs/*-SNAPSHOT.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

我們在 `build.gradle.kts` 增加 `buildImage` task，進行打包 image 的動作。

```kotlin title="build.gradle.kts"
val image = "<image>"

tasks.register("buildImage") {
    group = "docker"
    description = "Build the Docker image"
    dependsOn("build")
    doFirst {
        exec {
            commandLine("sh", "-c", "docker buildx build -t $image --platform linux/amd64 .")
        }
    }
}
```

將 `<image>` 替換成目標 image 的名稱。`buildImage` 相依 `build`，在執行 task 之前，會建置產生最新的 jar 檔。為了在 kubernetes 上執行 image，設定 image 的目標平台為 `--platform linux/amd64`。


## 將 image 推上 image hub

在 `build.gradle.kts` 增加 `pushImage` task，將打包的 image 推上 remote repository 中。task 相依前一步的 `buildImage` task，確保使用最新的 image。

```gradle title="build.gradle.kts"
tasks.register("pushImage") {
    group = "docker"
    description = "Push the Docker image"
    dependsOn("buildImage")
    doFirst {
        exec {
            val username = "<username>"
            val password = "<password>"
            commandLine("sh", "-c", "docker login -u $username -p $password")
            commandLine("sh", "-c", "docker push $image")
        }
    }
}
```

## 在 kubernetes 上執行目標程式

執行以下動作，即可執行程式:

```shell
$ kubectl run -it <name> --rm --image=<image> -n <namespace>
```

替換 `<name>`, `<image>`, `<namespace>`
