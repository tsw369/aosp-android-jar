# 编译修改版 android.jar 的完整步骤

## 项目说明

本项目通过修改 AOSP 源码，编译出允许普通应用在编译时访问系统级 API 的 android.jar。这个修改版的 android.jar 主要解决以下问题：

1. 编译时限制：
   - 普通应用在编译时无法直接调用系统级 API
   - 需要通过反射或其他复杂方式访问这些 API
   - 修改后的 android.jar 允许直接调用这些 API，简化开发过程

2. 运行时限制：
   - 即使使用修改后的 android.jar 编译通过
   - 普通应用在运行时仍然会受到系统权限限制
   - 调用系统级 API 时可能会抛出 SecurityException 或其他权限异常

3. 适用场景：
   - 系统应用开发
   - 需要 root 权限的应用
   - 系统定制开发
   - 测试和调试环境

4. 注意事项：
   - 仅用于开发测试目的
   - 不建议在正式发布的应用中使用
   - 需要配合相应的系统权限才能正常运行
   - 使用时需要了解目标系统的权限机制

## 1. 准备 AOSP 编译环境

```bash
# 安装必要的依赖包(以Ubuntu为例)
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
```

## 2. 下载 AOSP 源码

```bash
# 初始化 repo 工具
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo

# 创建 AOSP 目录并初始化仓库
mkdir AOSP
cd AOSP
# 使用 Android 10.0.0_r1 分支
repo init -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r1
repo sync -j8 # 同步代码，这步会花很长时间
```

## 3. 修改编译配置

### 3.1 修改 hiddenapi.go

在 Android 10 中，修改 `build/soong/java/hiddenapi.go` 文件：

```go
// 修改 hiddenAPIGenerateCSVRule 规则
var hiddenAPIGenerateCSVRule = pctx.AndroidStaticRule("hiddenAPIGenerateCSV", blueprint.RuleParams{
    // 删除 --stub-api-flags 参数
    Command:     "${config.Class2Greylist} $in $outFlag $out",
    CommandDeps: []string{"${config.Class2Greylist}"},
}, "outFlag", "stubAPIFlags")

// 修改 hiddenAPIEncodeDexRule 规则
var hiddenAPIEncodeDexRule = pctx.AndroidStaticRule("hiddenAPIEncodeDex", blueprint.RuleParams{
    Command: `rm -rf $tmpDir && mkdir -p $tmpDir && mkdir $tmpDir/dex-input && mkdir $tmpDir/dex-output && ` +
        `unzip -o -q $in 'classes*.dex' -d $tmpDir/dex-input && ` +
        `for INPUT_DEX in $$(find $tmpDir/dex-input -maxdepth 1 -name 'classes*.dex' | sort); do ` +
        `  echo "--input-dex=$${INPUT_DEX}"; ` +
        `  echo "--output-dex=$tmpDir/dex-output/$$(basename $${INPUT_DEX})"; ` +
        `done | xargs ${config.HiddenAPI} encode $hiddenapiFlags && ` +  // 删除 --api-flags=$flagsCsv
        `${config.SoongZipCmd} $soongZipFlags -o $tmpDir/dex.jar -C $tmpDir/dex-output -f "$tmpDir/dex-output/classes*.dex" && ` +
        `${config.MergeZipsCmd} -D -zipToNotStrip $tmpDir/dex.jar -stripFile "classes*.dex" $out $tmpDir/dex.jar $in`,
    CommandDeps: []string{
        "${config.HiddenAPI}",
        "${config.SoongZipCmd}",
        "${config.MergeZipsCmd}",
    },
}, "flagsCsv", "hiddenapiFlags", "tmpDir", "soongZipFlags")
```

### 3.2 修改 hiddenapi_singleton.go

在 Android 10 中，修改 `build/soong/java/hiddenapi_singleton.go` 文件：

```go
// 修改 GenerateBuildActions 方法
func (h *hiddenAPISingleton) GenerateBuildActions(ctx android.SingletonContext) {
    // 删除所有内容，直接返回
    return
}

// 修改 MakeVars 方法
func (h *hiddenAPISingleton) MakeVars(ctx android.MakeVarsContext) {
    // 删除所有内容，直接返回
    return
}

// 修改 flagsRule 方法
func flagsRule(ctx android.SingletonContext) android.Path {
    // 直接返回空文件
    return emptyFlagsRule(ctx)
}
```

## 4. 编译 android.jar

```bash
# 1. 备份原始文件
cp build/soong/java/hiddenapi.go build/soong/java/hiddenapi.go.bak
cp build/soong/java/hiddenapi_singleton.go build/soong/java/hiddenapi_singleton.go.bak

# 2. 设置编译环境
source build/envsetup.sh

# 要兼容多架构，选择 aosp_cf_x86_phone-userdebug 或类似目标
# 该目标会生成与架构无关的 framework.jar
lunch aosp_arm64-eng

# 3. 设置环境变量
export UNSAFE_DISABLE_HIDDENAPI_FLAGS=true

# 4. 编译 framework（Android 10 中使用此命令）
make framework

# 5. 编译结果位置
# out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar
```

## 5. 验证修改

```bash
# 1. 检查生成的 classes.jar
ls -l out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar

# 2. 使用 javap 检查 API 是否可见
javap -classpath out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar android.app.Activity
```

## 6. 将生成的 jar 用于开发

```bash
# 复制生成的 jar 文件到你想要的位置
cp out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar ~/android.jar

# 在 Android 项目中，替换 SDK 中的 android.jar
# 例如替换 ${Android Sdk}/platforms/android-29/android.jar
```

## 注意事项

1. Android 10 (API 29) 特有的考量：
   - Android 10 引入了更严格的非 SDK 接口限制
   - 需要注意 Android 10 中一些类可能被移动或重构
   - 部分 hidden API 可能在 Android 10 中行为有所变化

2. 关于 hidden API 的处理：
   - 需要修改 AOSP 源码中的 API 可见性控制
   - 可能需要修改 `build/soong` 中的相关配置
   - 某些情况下可能需要修改 `frameworks/base` 中的 API 定义

3. 优化建议：
   - 可以只编译必要的模块来加快编译速度
   - 使用足够的 CPU 核心和内存来提升编译效率
   - 建议使用 SSD 存储来加快编译过程

4. 安全提示：
   - 修改后的 android.jar 仅建议用于开发测试
   - 不建议在正式发布的应用中使用修改版 android.jar
   - 需要注意 Google 对非 SDK 接口的政策限制

5. 多架构兼容说明：
   - 编译生成的 android.jar 本身是与架构无关的 Java 代码
   - 不同架构的兼容性主要体现在 native 代码部分
   - 选择 aosp_cf_x86_phone-userdebug 编译目标可以获得通用的 framework 库
   - 对于纯 Java API 调用，此 jar 文件可用于所有架构 

## 参考地址
   - https://github.com/Reginer/aosp-android-jar
