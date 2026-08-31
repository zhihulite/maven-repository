# maven-repository

zhihulite 的自建 Maven 仓库。制品以文件形式直接存在本仓库，通过
`raw.githubusercontent.com` 供 Gradle 解析 —— 不依赖任何仓库服务，
所以更新和删除都只是普通 commit。

## 仓库里有什么

| 坐标 | 内容 | 源码仓库 |
|---|---|---|
| `io.github.zhihulite:luajvm-core` | Lua 5.5.1 的纯 Java 移植：VM、编译器、标准库、Java 绑定层 | [zhihulite/luajvm](https://github.com/zhihulite/luajvm/luajvm-core) |
| `io.github.zhihulite:luajvm-android` | luajvm 的 Android 平台层，以 `api` 依赖 core | 同上 |
| `com.google.android.material:material` | **定制版** Material Components | [zhihulite/material-components-android](https://github.com/zhihulite/material-components-android) |

每个制品都带 `.pom`、`.module`（Gradle Module Metadata）、sources jar，
以及 `.md5`/`.sha1`/`.sha256`/`.sha512` 四种校验和文件，
同级还有 `maven-metadata.xml` —— raw 域名不提供目录列表，版本发现全靠这个文件。

项目里用到这些制品的是 [zhihulite/Hydrogen](https://github.com/zhihulite/Hydrogen)。

## 目录结构

```
repository/
├── releases/    正式版
└── snapshots/   版本号以 -SNAPSHOT 结尾的开发版
```

两者分开放，方便只引用其中一个，避免把开发版误解析进发布构建。

## 怎么导入

`settings.gradle`：

```groovy
dependencyResolutionManagement {
    repositories {
        maven {
            name = 'zhihulite-releases'
            url = 'https://raw.githubusercontent.com/zhihulite/maven-repository/main/repository/releases'
            content {
                includeGroup 'io.github.zhihulite'
                // material 用 includeModule 而不是 includeGroup：
                // 后者会把 com.google.android.material 组下所有模块都拦到这个仓库，
                // 但本仓库只有 material 一个，其余的（如 material-icons）会解析失败
                includeModule 'com.google.android.material', 'material'
            }
        }
        google()
        mavenCentral()
    }
}
```

`build.gradle`：

```groovy
dependencies {
    // luajvm-android 已经 api 依赖了 luajvm-core，不用再单独声明 core
    implementation 'io.github.zhihulite:luajvm-android:VERSION'
    // 定制 material（luajvm-android 也会传递引入它）
    implementation 'com.google.android.material:material:VERSION'
}
```

其中 `VERSION` 是各制品的发布版本号（luajvm 当前为 `1.0.0`，material 当前为 `1.14.0`）。

如果只要 JVM 引擎、不要 Android 平台层：

```groovy
implementation 'io.github.zhihulite:luajvm-core:VERSION'
```

要 snapshot 的话，把 URL 末尾换成 `repository/snapshots`，另加一个 `maven { }` 块。

`raw.githubusercontent.com` 在某些网络环境下可能连不上，可以换镜像域名，
`repository/releases` 后面的路径不变。

### 定制 material 需要确保本地覆盖

material 的坐标和官方**完全一样**。如果不使用本仓库的定制版而换回官方版本，会丢失预测性返回动画支持和相关的性能优化。

要确保用的是定制版，可以用约束强制覆盖：

```groovy
dependencies {
    implementation 'com.google.android.material:material:VERSION'
    constraints {
        implementation('com.google.android.material:material:VERSION') {
            because '定制版：必须覆盖任何传递引入的官方 material'
        }
    }
}
```

验证：

```bash
./gradlew :app:dependencyInsight --configuration debugRuntimeClasspath \
    --dependency com.google.android.material:material
```

生效时输出里会有 `By constraint: 定制版：…`，版本号为 `VERSION`。
注意这个命令**不显示制品来自哪个仓库**，版本号相同也不代表字节相同；
想确认拿到的是定制版，比对 AAR 的 sha1 和本仓库里的那份是否一致。

## 怎么发布

各源码仓库把制品写入本仓库的**本地工作副本**，再由本仓库 `git commit` + `git push` 分发。
三个仓库并排放的时候，默认路径直接能用：

```
GitHub/
├── luajvm/
├── material-components-android/
└── maven-repository/          # 本仓库
```

luajvm：

```bash
cd ../luajvm
./gradlew publish                                  # 写入 ../maven-repository/repository/releases
./gradlew publish -PluajvmVersion=VERSION          # 指定版本号
./gradlew publish -PluajvmVersion=VERSION-SNAPSHOT # 发 snapshot
./gradlew publish -PmavenRepoDir=<本仓库路径>       # 换位置
```

material：

```bash
cd ../material-components-android
./gradlew :lib:publishReleasePublicationToMavenRepository
# 换位置：-PmavenRepoUrl=<本仓库>/repository/releases
# 改版本号：build.gradle 里的 mdcLibraryVersion
```

写入后在本仓库提交推送：

```bash
git add -A && git commit -m "release luajvm VERSION" && git push
```

### 提交前建议核对校验和

制品的 `.module` 里嵌了 AAR 的 size 和 sha512，每个文件还有四种 sidecar 校验和。
任何改动了字节的环节都会让它们对不上，用到这些制品的人按校验和验证时会直接失败，不会静默降级。

```python
import glob, hashlib, json, os

ALGOS = ('md5', 'sha1', 'sha256', 'sha512')
bad = n = 0

# 1) sidecar 校验和 vs 对应文件
for root, _, files in os.walk('repository'):
    for f in files:
        for algo in ALGOS:
            if f.endswith('.' + algo):
                side = os.path.join(root, f)
                base = side[:-(len(algo) + 1)]
                if not os.path.exists(base):
                    print('孤立 sidecar:', side); bad += 1; continue
                want = open(side, encoding='utf-8').read().strip()
                got = hashlib.new(algo, open(base, 'rb').read()).hexdigest()
                n += 1
                if want != got:
                    print('校验和不符:', side); bad += 1

# 2) .module 内嵌的 size/校验和 vs 各自 url 指向的文件
#    注意每个 variant 的 files[] 可能指向 aar/sources/javadoc 中的不同文件，
#    不能一律拿主制品去比。
for mp in glob.glob('repository/**/*.module', recursive=True):
    d = os.path.dirname(mp)
    for v in json.load(open(mp, encoding='utf-8'))['variants']:
        for f in v.get('files', []):
            p = os.path.join(d, f['url'])
            if not os.path.exists(p):
                print('module 指向的文件缺失:', p); bad += 1; continue
            b = open(p, 'rb').read(); n += 1
            if f.get('size') != len(b):
                print('module size 不符:', f['url']); bad += 1
            for algo in ALGOS:
                if algo in f and f[algo] != hashlib.new(algo, b).hexdigest():
                    print('module %s 不符: %s' % (algo, f['url'])); bad += 1

print('核对 %d 项，不符 %d 项' % (n, bad))
```

### 为什么 repository/ 下面禁用了换行归一化

`.gitattributes` 对 `repository/**` 声明了 `-text`。默认的 `* text=auto` 依赖 Git 的
二进制启发式判断，一旦某个文本型元数据被归一化，字节数就变了 —— 实测
`material-VERSION.module` 会从 9101 字节被改写成 8756 字节（345 个 CR 被剥掉），
sidecar 校验和随即全部失效。制品必须逐字节稳定，所以整个 `repository/` 关掉了文本转换。

## License

本仓库自身的说明和配置按 MIT 授权，见 [LICENSE](LICENSE)。
制品各自的许可以其 POM 里的声明为准 —— `luajvm-*` 是 MIT，
`material` 沿用上游的 Apache-2.0。