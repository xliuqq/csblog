# 镜像工具



## [Skopeo](https://github.com/containers/skopeo)

> Work with ***remote*** images registries - retrieving information, images, signing content

`skopeo`是一个命令行实用程序，可以对容器映像和映像存储库执行各种操作。

- 不用 root 权限，不需要守护进程；
- 支持 OCI 和 Docker v2 镜像。

Skopeo与API V2容器映像注册表一起使用，例如docker.io和quay.io注册表、私有注册表、本地目录和本地OCI布局目录。

- 可以作为镜像搬运工具，在不同的存储中进行迁移和管理；



## [Podman](https://github.com/containers/podman)

> Podman: A tool for managing OCI containers and pods.

像 Docker 一样提供基本的容器管理，还可以管理 pod。



## [Buildah](https://github.com/containers/buildah)

 Buildah 相对于 Podman 主要的优势在于可以在没有 Doclerfiles 的情况下创建容器镜像。



## [Kaniko](https://github.com/GoogleContainerTools/kaniko)

> Build Container Images In Kubernetes.

kaniko is a tool to build container images from a Dockerfile, **inside a container** or Kubernetes cluster.

- 不需要守护进程，用于 CI / CD 中构建镜像



示例如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
name: kaniko
spec:
 containers:
 - name: kaniko
   image: gcr.io/kaniko-project/executor:latest
   # command 默认是 /kaniko/executor
   # Dockerfile(--dockerfile)，上下文(--context)，远端镜像仓库(--destination)
   args: ["--dockerfile=/workspace/Dockerfile",
          "--context=/workspace/",
          "--destination=dllhb.docker.io/kaniko-test:v0.4"]
    volumeMounts:
      # 远程仓库的密钥文件
      - name: kaniko-secret
        mountPath: /kaniko/.docker
      - name: dockerfile
        mountPath: /workspace/Dockerfile
        subPath: Dockerfile
    restartPolicy: Never
volumes:
  - name: dockerfile
    configMap:
      name: dockerfile
  - name: kaniko-secret
     projected:
     sources:
     - secret:
        name: regcred
        items:
        - key: .dockerconfigjson
          path: config.json
```

