---
title: "Developement Containersのローカル開発環境ではbuild.cacheFromを使うのがよさそう"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "DevelopmentContainers", "remotecontainers"]
published: true
---

## 結論

[devcontainers/ci](https://github.com/devcontainers/ci)で、Dockerレジストリに一度ビルドしたDevelopment Containersのベースイメージをプッシュしておき、[devcontainer.json](https://code.visualstudio.com/docs/remote/devcontainerjson-reference)の`build.cacheFrom`を使って、プッシュしたDockerイメージを使うようにするとローカル開発環境の構築が早くできる。
これにより、特にDockerビルドに時間を要す[Dev container features](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)を導入する場合のこの問題に対応できる。
また、各開発者のローカル開発環境構築のタイミングに依らず、Dockerレジストリに格納された同じDockerイメージを各々の開発者の開発環境で使うことができる。

## 背景・課題

Development Containers(VSCode Remote Containers)のローカル開発環境の再現性の高さやDev Container featuresによる任意の環境やツール導入の容易さを、筆者は気に入っていてこれをよく使っている。その一方で、特に[Dev container features](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)で導入したプログラミング言語の開発環境が多いほど、Dockerビルドに時間がかかるのが難点だと考えていた。

例えば、以下のようにDev container featuresでプログラミング言語の開発環境をたくさん設定した場合、自分のローカル開発環境だと770秒ほどDockerビルドに要した。

```json:.devcontainer.json
{
  "image": "mcr.microsoft.com/vscode/devcontainers/base:debian-11",
  "features": {
    "python": "latest",
    "golang": "latest",
    "ruby": "latest",
    "java": "latest",
    "rust": {
        "version": "latest"
    }
  }
}
```

実際にこれだけの数のfeaturesを設定することはあまりないと思うし、Dockerビルドが一度でも動作すればキャッシュが手元に残るので、2回目以降の起動は早くなる。
とはいえ、開発環境の更新が入ったら、その変更箇所からのDockerビルドが必要となり、これが終わるまで待たされることは発生しうる。これを用いる開発者が増えれば増えるほど、この待ち時間の総和が大きくなり、最終的にコストにはねる。
また、同じdevcontainer.jsonを使用していても、上記のようなベースイメージのタグ指定やDev container featuresのバージョン指定でlatestを使用していると、ローカル開発環境構築時のタイミングによって、ビルドによって作られるDockerイメージが開発者の開発環境毎に変わる可能性もある。

## 解決策 

[devcontainer.json](https://code.visualstudio.com/docs/remote/devcontainerjson-reference)の`build.cacheFrom`を使うと、これに設定したDockerレジストリに保存したDockerイメージをDockerプルしてコンテナ起動することができる。

`build.cacheFrom`を使うように対応したときの差分は以下。

https://github.com/nmemoto/try-devcontainer-ci/commit/b6615f10c395133cf060ffd7abc542b53507fd53?diff=split

`devcontainer.json`で`build.cacheFrom`を指定するためには、Dockerビルドを行うためのDockerfileを作成する必要がある。
ここでは、Dockerfileには使用したいベースイメージのFROM句への指定のみ記述した。

この方法を用いるには`build.cacheFrom`に指定したDockerレジストリにDockerイメージを格納しておく必要がある。
GitHub Actionsを用いたGirHub Container RegistryへのDockerイメージのプッシュについては以下に記載があるが、devcontainers/ci@~のactionsのタスクでimageNameを指定するとイメージのビルドとプッシュが行われる。

https://github.com/devcontainers/ci/blob/main/docs/github-action.md#inputs

今回は行わなかったが、actionsのタスクにimageTagを設定し、タグ含めたDockerイメージの指定を`build.cacheFrom`にすると、CIとローカル開発環境のイメージで全く同じものを使用することができ、開発者のローカル開発環境も同じにすることができる。

### 計測 

`build.cacheFrom`の未利用時と利用時で筆者のローカル開発環境でのコンテナ起動までの時間を比較した。
それぞれ[devcontainer/cli](https://github.com/devcontainers/cli)のv0.8.0を用いてコンテナ起動し、`build.cacheFrom`利用時のdevcontainer.jsonは[こちら](https://github.com/nmemoto/try-devcontainer-ci/blob/b6615f10c395133cf060ffd7abc542b53507fd53/.devcontainer/devcontainer.json)を使い、`build.cacheFrom`未利用時は`build.cacheFrom`の行を削除したものを使った。
両者とも試す前にローカルのDockerイメージは削除してローカルキャッシュを使わないようにした。

結果は、`build.cacheFrom`の未利用時は204秒、利用時は6秒となり、圧倒的に`build.cacheFrom`を使ったほうが早かった。
golangの開発環境をDev container featuresを使って構築しており、下記のログより未利用時はこのDockerビルドに180秒ほどかかっていた。

:::details build.cacheFrom 未利用時
devcontainer up --workspace-folder .
[29 ms] @devcontainers/cli 0.8.0.
[2101 ms] Start: Run: docker buildx build --load --build-arg BUILDKIT_INLINE_CACHE=1 -f /var/folders/yd/5fdt04gj5ll8wnybvr2103r40000gn/T/vsch/container-features/0.8.0-1659524438844/Dockerfile-with-features -t vsc-try-devcontainer-ci-83dd645cc5e4f61846a3ed206be30c1b --build-context dev_containers_feature_content_source=/var/folders/yd/5fdt04gj5ll8wnybvr2103r40000gn/T/vsch/container-features/0.8.0-1659524438844 --build-arg _DEV_CONTAINERS_BASE_IMAGE=dev_container_auto_added_stage_label --build-arg _DEV_CONTAINERS_IMAGE_USER=root --build-arg _DEV_CONTAINERS_FEATURE_CONTENT_SOURCE=dev_container_feature_content_temp /Users/nmemoto/ghq/github.com/nmemoto/try-devcontainer-ci/.devcontainer
[+] Building 202.0s (16/16) FINISHED
 => [internal] load build definition from Dockerfile-with-features         0.0s
 => => transferring dockerfile: 590B                                       0.0s
 => [internal] load .dockerignore                                          0.0s
 => => transferring context: 2B                                            0.0s
 => resolve image config for docker.io/docker/dockerfile:1.4               2.6s
 => [auth] docker/dockerfile:pull token for registry-1.docker.io           0.0s
 => CACHED docker-image://docker.io/docker/dockerfile:1.4@sha256:443aab4c  0.0s
 => [internal] load build definition from Dockerfile-with-features         0.0s
 => [internal] load .dockerignore                                          0.0s
 => [internal] load metadata for mcr.microsoft.com/vscode/devcontainers/b  0.0s
 => [context dev_containers_feature_content_source] load .dockerignore     0.0s
 => => transferring dev_containers_feature_content_source: 2B              0.0s
 => CACHED [dev_container_auto_added_stage_label 1/1] FROM mcr.microsoft.  0.0s
 => [context dev_containers_feature_content_source] load from client       0.0s
 => => transferring dev_containers_feature_content_source: 872.63kB        0.0s
 => [stage-1 1/3] COPY --from=dev_containers_feature_content_source . /tm  0.0s
 => [stage-1 2/3] RUN cd /tmp/build-features/github-cli_2 && chmod +x ./  12.7s
 => [stage-1 3/3] RUN cd /tmp/build-features/golang_1 && chmod +x ./ins  181.4s
 => exporting to image                                                     0.0s
 => => exporting layers                                                    0.0s
 => => writing image sha256:cc0b53c870900076b6119da9153ff029015587214baea  0.0s
 => => naming to docker.io/library/vsc-try-devcontainer-ci-83dd645cc5e4f6  0.0s
 => exporting cache                                                        0.0s
 => => preparing build cache for export                                    0.0s
[204550 ms] Start: Run: docker run --sig-proxy=false -a STDOUT -a STDERR --mount type=bind,source=/Users/nmemoto/ghq/github.com/nmemoto/try-devcontainer-ci,target=/workspaces/try-devcontainer-ci,consistency=cached -l devcontainer.local_folder=/Users/nmemoto/ghq/github.com/nmemoto/try-devcontainer-ci --entrypoint /bin/sh vsc-try-devcontainer-ci-83dd645cc5e4f61846a3ed206be30c1b -c echo Container started
Container started
{"outcome":"success","containerId":"b2552a77d537f2bb9d57621f30cbe092019eba876f3ee83b407fcd0bb90ef05d","remoteUser":"vscode","remoteWorkspaceFolder":"/workspaces/try-devcontainer-ci"}
:::

:::details build.cacheFrom 利用時
devcontainer up --workspace-folder .
[21 ms] @devcontainers/cli 0.8.0.
[1782 ms] Start: Run: docker buildx build --load --build-arg BUILDKIT_INLINE_CACHE=1 -f /var/folders/yd/5fdt04gj5ll8wnybvr2103r40000gn/T/vsch/container-features/0.8.0-1659524991009/Dockerfile-with-features -t vsc-try-devcontainer-ci-83dd645cc5e4f61846a3ed206be30c1b --cache-from ghcr.io/nmemoto/try-devcontainer-ci --build-context dev_containers_feature_content_source=/var/folders/yd/5fdt04gj5ll8wnybvr2103r40000gn/T/vsch/container-features/0.8.0-1659524991009 --build-arg _DEV_CONTAINERS_BASE_IMAGE=dev_container_auto_added_stage_label --build-arg _DEV_CONTAINERS_IMAGE_USER=root --build-arg _DEV_CONTAINERS_FEATURE_CONTENT_SOURCE=dev_container_feature_content_temp /Users/nmemoto/ghq/github.com/nmemoto/try-devcontainer-ci/.devcontainer
[+] Building 4.2s (17/17) FINISHED
 => [internal] load build definition from Dockerfile-with-features         0.0s
 => => transferring dockerfile: 590B                                       0.0s
 => [internal] load .dockerignore                                          0.0s
 => => transferring context: 2B                                            0.0s
 => resolve image config for docker.io/docker/dockerfile:1.4               1.8s
 => [auth] docker/dockerfile:pull token for registry-1.docker.io           0.0s
 => CACHED docker-image://docker.io/docker/dockerfile:1.4@sha256:443aab4c  0.0s
 => [internal] load build definition from Dockerfile-with-features         0.0s
 => [internal] load .dockerignore                                          0.0s
 => [internal] load metadata for mcr.microsoft.com/vscode/devcontainers/b  0.0s
 => [context dev_containers_feature_content_source] load .dockerignore     0.0s
 => => transferring dev_containers_feature_content_source: 2B              0.0s
 => importing cache manifest from ghcr.io/nmemoto/try-devcontainer-ci      2.0s
 => [context dev_containers_feature_content_source] load from client       0.1s
 => => transferring dev_containers_feature_content_source: 604.46kB        0.0s
 => [dev_container_auto_added_stage_label 1/1] FROM mcr.microsoft.com/vsc  0.0s
 => CACHED [stage-1 1/3] COPY --from=dev_containers_feature_content_sourc  0.0s
 => CACHED [stage-1 2/3] RUN cd /tmp/build-features/github-cli_2 && chmod  0.0s
 => CACHED [stage-1 3/3] RUN cd /tmp/build-features/golang_1 && chmod +x   0.0s
 => exporting to image                                                     0.0s
 => => exporting layers                                                    0.0s
 => => writing image sha256:cc0b53c870900076b6119da9153ff029015587214baea  0.0s
 => => naming to docker.io/library/vsc-try-devcontainer-ci-83dd645cc5e4f6  0.0s
 => exporting cache                                                        0.0s
 => => preparing build cache for export                                    0.0s
[6421 ms] Start: Run: docker run --sig-proxy=false -a STDOUT -a STDERR --mount type=bind,source=/Users/nmemoto/ghq/github.com/nmemoto/try-devcontainer-ci,target=/workspaces/try-devcontainer-ci,consistency=cached -l devcontainer.local_folder=/Users/nmemoto/ghq/github.com/nmemoto/try-devcontainer-ci --entrypoint /bin/sh vsc-try-devcontainer-ci-83dd645cc5e4f61846a3ed206be30c1b -c echo Container started
Container started
{"outcome":"success","containerId":"f990fcc871d08572194fa321dc96f17f8413ca73f07d233a9c6f724978012101","remoteUser":"vscode","remoteWorkspaceFolder":"/workspaces/try-devcontainer-ci"}
:::

逆に`build.cacheFrom`を使わないときはDev container featuresでプログラミング言語の環境を用意するべきではなく、以下のようなベースイメージに含まれていたものを使ったほうがよさそうなことも明らかになったと思う。
https://github.com/microsoft/vscode-dev-containers/tree/main/containers/go


## 引用元

`build.cacheFrom`を用いてローカル開発環境構築の高速化については以下で紹介されていた。
https://stuartleeks.com/posts/vscode-dev-containers-continuous-integration/#bonus---speeding-up-local-dev-container-image-builds

