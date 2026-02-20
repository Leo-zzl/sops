# OpenClaw WSL2 迁移脚本

> 适用于 OpenClaw 环境迁移的完整脚本方案
> 创建日期：2026-02-20

---

## 📦 脚本1：export.sh（源环境导出）

```bash
#!/bin/bash
# openclaw-export.sh - 导出 OpenClaw 完整配置

set -e

BACKUP_NAME="openclaw-backup-$(date +%Y%m%d-%H%M%S).tar.gz"
BACKUP_PATH="$HOME/$BACKUP_NAME"
WINDOWS_DESKTOP="/mnt/c/Users/\${USER:-\$USERNAME}/Desktop"

echo "🐰 OpenClaw 导出工具"
echo "===================="

# 检查 OpenClaw 是否安装
if ! command -v openclaw &> /dev/null; then
    echo "❌ OpenClaw 未安装"
    exit 1
fi

# 停止服务
echo "📍 停止 OpenClaw 服务..."
openclaw gateway stop 2>/dev/null || echo "   服务未运行，继续..."

# 创建备份
echo "📦 打包配置..."
cd ~
tar czvf "$BACKUP_NAME" \
    .openclaw/config/ \
    .openclaw/workspace/ \
    .openclaw/pairing/ \
    .openclaw/secrets/ \
    2>/dev/null || true

# 检查备份大小
BACKUP_SIZE=$(du -h "$BACKUP_PATH" | cut -f1)
echo "✅ 备份完成: $BACKUP_NAME ($BACKUP_SIZE)"

# 复制到 Windows 桌面（如果存在）
if [ -d "$WINDOWS_DESKTOP" ]; then
    echo "📤 复制到 Windows 桌面..."
    cp "$BACKUP_PATH" "$WINDOWS_DESKTOP/"
    echo "✅ 已复制到桌面"
fi

echo ""
echo "🎉 导出完成！"
echo "   文件: $BACKUP_NAME"
echo "   路径: $BACKUP_PATH"
```

---

## 📥 脚本2：import.sh（目标环境导入）

```bash
#!/bin/bash
# openclaw-import.sh - 导入 OpenClaw 配置

set -e

RESET_MODE=false
BACKUP_FILE=""

usage() {
    echo "用法: \$0 [选项] <备份文件>"
    echo ""
    echo "选项:"
    echo "  -r, --reset    完全重置目标环境"
    echo "  -h, --help     显示帮助"
    exit 0
}

# 解析参数
while [[ \$# -gt 0 ]]; do
    case \$1 in
        -r|--reset)
            RESET_MODE=true
            shift
            ;;
        -h|--help)
            usage
            ;;
        *)
            BACKUP_FILE="\$1"
            shift
            ;;
    esac
done

if [ -z "$BACKUP_FILE" ] || [ ! -f "$BACKUP_FILE" ]; then
    echo "❌ 请指定有效的备份文件"
    usage
fi

echo "🐰 OpenClaw 导入工具"
echo "===================="
echo "模式: $([ "$RESET_MODE" = true ] && echo "重置" || echo "合并" )"
echo ""

# 检查 OpenClaw 是否已安装
if ! command -v openclaw &> /dev/null; then
    echo "📥 安装 OpenClaw..."
    curl -fsSL https://openclaw.ai/install.sh | bash
else
    echo "📍 停止现有服务..."
    openclaw gateway stop 2>/dev/null || true
fi

# 重置模式
if [ "$RESET_MODE" = true ]; then
    echo "🧹 重置模式：删除现有配置..."
    rm -rf ~/.openclaw 2>/dev/null || true
fi

# 解压备份
echo "📦 解压备份..."
cd ~
tar xzvf "$BACKUP_FILE"

# 确保目录结构
mkdir -p ~/.openclaw/{config,workspace,pairing}

echo "✅ 导入完成"

# 启动服务
echo ""
echo "🚀 启动 OpenClaw..."
openclaw gateway start
sleep 2

echo ""
echo "📊 状态检查:"
openclaw status

echo ""
echo "🎉 完成！"
```

---

## 🚀 脚本3：migrate.sh（一键迁移）

```bash
#!/bin/bash
# openclaw-migrate.sh - 一键迁移

set -e

SOURCE_WSL=""
RESET_MODE=false

usage() {
    echo "用法: \$0 -s <源WSL名称> [-r]"
    echo ""
    echo "示例:"
    echo "  \$0 -s Ubuntu-22.04"
    echo "  \$0 -s Ubuntu-22.04 -r"
    exit 0
}

while [[ \$# -gt 0 ]]; do
    case \$1 in
        -s|--source)
            SOURCE_WSL="\$2"
            shift 2
            ;;
        -r|--reset)
            RESET_MODE=true
            shift
            ;;
        *)
            echo "❌ 未知选项: \$1"
            usage
            ;;
    esac
done

if [ -z "$SOURCE_WSL" ]; then
    echo "❌ 请指定源 WSL2 名称"
    wsl -l -v 2>/dev/null || true
    usage
fi

echo "🐰 OpenClaw 一键迁移"
echo "===================="
echo "源环境: $SOURCE_WSL"
echo ""

TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

# 步骤1：导出
echo "📤 步骤1/3: 源环境导出..."
wsl -d "$SOURCE_WSL" -e bash -c "
    cd ~
    if [ -d .openclaw ]; then
        openclaw gateway stop 2>/dev/null || true
        mkdir -p /mnt/c/wsl-temp
        tar czvf /mnt/c/wsl-temp/openclaw-migrate.tar.gz .openclaw/
        echo 'EXPORT_DONE'
    fi
"

if [ -f "/mnt/c/wsl-temp/openclaw-migrate.tar.gz" ]; then
    cp "/mnt/c/wsl-temp/openclaw-migrate.tar.gz" "$TEMP_DIR/"
    BACKUP_FILE="$TEMP_DIR/openclaw-migrate.tar.gz"
else
    echo "❌ 导出失败"
    exit 1
fi

echo "✅ 导出完成"

# 步骤2：导入
echo ""
echo "📥 步骤2/3: 目标环境导入..."

if [ "$RESET_MODE" = true ]; then
    openclaw gateway stop 2>/dev/null || true
    rm -rf ~/.openclaw
fi

if ! command -v openclaw &> /dev/null; then
    echo "📥 安装 OpenClaw..."
    curl -fsSL https://openclaw.ai/install.sh | bash
fi

cd ~
tar xzvf "$BACKUP_FILE"
rm -f /mnt/c/wsl-temp/openclaw-migrate.tar.gz

echo "✅ 导入完成"

# 步骤3：启动
echo ""
echo "🚀 步骤3/3: 启动服务..."
openclaw gateway start
sleep 2

echo ""
echo "📊 最终状态:"
openclaw status

echo ""
echo "🎉 迁移完成！"
```

---

## 📋 使用说明

### 场景1：手动两步迁移
```bash
# 源环境
./openclaw-export.sh

# 目标环境  
./openclaw-import.sh -r openclaw-backup-xxx.tar.gz
```

### 场景2：一键自动迁移
```bash
./openclaw-migrate.sh -s Ubuntu-22.04 -r
```

## 📦 备份包含

- `config/` - Gateway 配置、渠道设置
- `workspace/` - 记忆文件、SOUL.md、AGENTS.md
- `pairing/` - 手机节点配对信息
- `secrets/` - 加密密钥

## ⚠️ 注意事项

- 迁移前停止 Gateway 服务
- 确保目标环境有相同或更新版本的 OpenClaw
- 外部服务配置（Feishu/Telegram 等）会保留
- 手机节点配对可能需要重新授权

---

*本文档为公开版本，所有敏感信息已脱敏处理*
