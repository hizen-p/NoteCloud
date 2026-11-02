
```bash
make -j$(nproc) 2>&1 | tee ~/make.log
```

通用日志错误抓取

```bash
grep -Ein '(^|[[:space:]])(error|fatal error):|undefined reference|cannot find|No such file or directory|make\[[0-9]+\]: \*\*\*' make.log
```

please fix package/allwinner/eyesee-mpp/dragonboard/Makefile 

全志流媒体框架报错

```bash
[1] 4106
ERROR: please fix package/allwinner/eyesee-mpp/dragonboard/Makefile - see logs/package/allwinner/eyesee-mpp/dragonboard/dump.txt for details
ERROR: please fix package/allwinner/eyesee-mpp/middleware/Makefile - see logs/package/allwinner/eyesee-mpp/middleware/dump.txt for details
```

由于芯片不支持但是由于别的文件可能依赖该框架导致make的时候会参与编译但是该框架不支持TARGET_CHIP=sun8iw20p1导致报错
将package/allwinner/eyesee-mpp/dragonboard/Makefile下的TARGET_CHIP修改跳过判断

```
ifneq ($(TARGET_CHIP),sun8iw20p1)
ifeq ($(TARGET_CHIP), sun8iw12p1)
    MPPCFG_CHIP := AW1721
else ifeq ($(TARGET_CHIP), sun8iw16p1)
    MPPCFG_CHIP := AW1816
else ifeq ($(TARGET_CHIP), sun8iw21p1)
    MPPCFG_CHIP := AW1886
else
    $(error fatal error! $(TARGET_CHIP) is not support by eyesee-mpp!)
endif
endif
```



将please fix package/allwinner/eyesee-mpp/middleware/Makefile下的TARGET_CHIP修改跳过判断

```bash
#要修改两处地方
SRC_CODE_DIR := $(TOPDIR)/external/eyesee-mpp/middleware
ifneq ($(TARGET_CHIP),sun8iw20p1)
ifeq ($(TARGET_CHIP), sun8iw12p1)
    SRC_CODE_DIR := $(SRC_CODE_DIR)/v5
else ifeq ($(TARGET_CHIP), sun8iw16p1)
    SRC_CODE_DIR := $(SRC_CODE_DIR)/v316
else ifeq ($(TARGET_CHIP), sun8iw19p1)
    SRC_CODE_DIR := $(SRC_CODE_DIR)/sun8iw19p1
else ifeq ($(TARGET_CHIP), sun8iw21p1)
    SRC_CODE_DIR := $(SRC_CODE_DIR)/sun8iw21
else
    $(error fatal error! $(TARGET_CHIP) is not support by eyesee-mpp!)
endif
endif

#config chip version: AW1721
ifneq ($(TARGET_CHIP),sun8iw20p1)
ifeq ($(TARGET_CHIP), sun8iw12p1)
    MPPCFG_CHIP := AW1721
else ifeq ($(TARGET_CHIP), sun8iw16p1)
    MPPCFG_CHIP := AW1816
else ifeq ($(TARGET_CHIP), sun8iw19p1)
    MPPCFG_CHIP := AW1817
else ifeq ($(TARGET_CHIP), sun8iw21p1)
    MPPCFG_CHIP := AW1886
else
    $(error fatal error! $(TARGET_CHIP) is not support by eyesee-mpp!)
endif
endif
```

环境检查脚本

```
#!/bin/bash
# t113_env_check.sh T113‑S3 Tina‑Linux环境检查脚本
# 使用步骤：
# source build/envsetup.sh
# lunch 7
# bash t113_env_check.sh

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

pass() { echo -e "${GREEN}[PASS] $*${NC}"; }
warn() { echo -e "${YELLOW}[WARN] $*${NC}"; }
fail() { echo -e "${RED}[FAIL] $*${NC}"; }

echo "================ T113‑S3 Tina‑Linux 环境检查 ================"

# 1. TINA_BUILD_TOP
if [ -z "${TINA_BUILD_TOP}" ];then
    fail "TINA_BUILD_TOP 为空，请先 source build/envsetup.sh && lunch 7"
else
    pass "TINA_BUILD_TOP=${TINA_BUILD_TOP}"
fi

# 2. ARCH
if [ "${ARCH}" != "arm" ];then
    fail "ARCH=${ARCH} 期望为 arm"
else
    pass "ARCH=${ARCH}"
fi

# 3. CROSS_COMPILE
if [ -z "${CROSS_COMPILE}" ];then
    fail "CROSS_COMPILE 为空"
else
    pass "CROSS_COMPILE=${CROSS_COMPILE}"
    # 校验末尾是否带横杠
    if [[ "${CROSS_COMPILE}" != *- ]];then
        warn "CROSS_COMPILE 末尾缺少 '-'，编译会失败！"
    fi
fi

# 4. KERNEL_SRC
if [ -z "${KERNEL_SRC}" ];then
    fail "KERNEL_SRC 为空"
else
    pass "KERNEL_SRC=${KERNEL_SRC}"
    if [ ! -d "${KERNEL_SRC}" ];then
        fail "KERNEL_SRC 目录不存在: ${KERNEL_SRC}"
    fi
fi

# 5. 检查交叉编译器是否在PATH
GCC_BIN="${CROSS_COMPILE}gcc"
if command -v ${GCC_BIN} >/dev/null 2>&1 ;then
    pass "交叉编译器 ${GCC_BIN} 找到"
    ${GCC_BIN} --version | head -n1
else
    fail "找不到编译器 ${GCC_BIN}, 检查PATH是否加入工具链bin目录"
fi

# 6. 检查内核源码Makefile
if [ -f "${KERNEL_SRC}/Makefile" ];then
    pass "内核Makefile存在 ${KERNEL_SRC}/Makefile"
else
    fail "内核Makefile不存在，请检查KERNEL_SRC"
fi

# 7. 打印PATH中包含prebuilt的片段
echo -e "\nPATH中prebuilt相关路径："
echo ${PATH} | tr ':' '\n' | grep prebuilt

echo -e "\n===================== 检查完成 ====================="
echo "如果有FAIL项，需要修复；WARN项需要留意风险"
echo "推荐补充脚本（lunch之后执行）："
cat <<'EOF'
TOOLCHAIN_BIN=${TINA_BUILD_TOP}/prebuilt/gcc-linux-arm/arm-linux-gnueabihf/bin
export PATH=${TOOLCHAIN_BIN}:$PATH
export ARCH=arm
export CROSS_COMPILE=arm-openwrt-linux-muslgnueabi-
export KERNEL_SRC=${TINA_BUILD_TOP}/lichee/linux-5.4
EOF
```

环境修复脚本 使用source启动



```
#!/bin/bash
# t113_env_fix.sh 补齐T113环境变量，必须source+ lunch7之后运行
if [ -z "${TINA_BUILD_TOP}" ];then
    echo "ERROR: TINA_BUILD_TOP为空，请先 source build/envsetup.sh && lunch 7"
    return 1
fi

TOOLCHAIN_BIN=${TINA_BUILD_TOP}/prebuilt/gcc/linux-x86/arm/toolchain-sunxi-musl/toolchain/bin

export PATH=${TOOLCHAIN_BIN}:$PATH
export ARCH=arm
export CROSS_COMPILE=arm-openwrt-linux-muslgnueabi-
export KERNEL_SRC=${TINA_BUILD_TOP}/lichee/linux-5.4

echo "环境变量已补齐："
echo "ARCH=${ARCH}"
echo "CROSS_COMPILE=${CROSS_COMPILE}"
echo "KERNEL_SRC=${KERNEL_SRC}"
echo "PATH追加: ${TOOLCHAIN_BIN}"

# 执行自检
bash t113_env_check.sh
```

快速备份脚本放在~/.bashrc 使用方法 bak 文件名

```

bak() {
    if [ $# -ne 1 ];then
        echo "usage: bak filename"
        return 1
    fi
    local f="$1"
    cp -v "$f" "${f}.bak"
}
```

