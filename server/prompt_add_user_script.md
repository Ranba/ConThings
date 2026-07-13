# Prompt

```
帮我写一个linux脚本，用于添加普通权限的用户。要求：
1. 检查当前发行版，以便使用匹配的命令；
2. 使用交互式的方式输入用户名、用户密码，如果输入的用户名已经存在，则提示是否显示该用户信息(y/n)，选择'y'显示，选择'n'退出脚本；
3. 添加用户的同时添加用户组；
4. 将用户自动加入到`sudo`组，并配置在使用`sudo`命令时无需输入密码；
5. 新增的用户使用系统的默认shell（一般是`/bin/bash`)；
请仔细分析我的需求，确认理解后让我向你发送"开始"命令，再开始工作。如果你还需要了解更多信息，请先向我询问。
```

# Script

Generate by `gpt-5-codex-mini`

```bash
#!/usr/bin/env bash
#
# 添加普通权限用户的交互式脚本
#

set -euo pipefail

# 确保以 root 身份运行
if [[ "$(id -u)" -ne 0 ]]; then
  echo "请以 root 身份运行此脚本。"
  exit 1
fi

# 检查发行版信息
if [[ -r /etc/os-release ]]; then
  # shellcheck disable=SC1091
  source /etc/os-release
else
  echo "无法读取 /etc/os-release，无法判断发行版。"
  exit 1
fi

DISTRO_ID="${ID,,}"
DISTRO_LIKE="${ID_LIKE:-}"
DISTRO_LIKE="${DISTRO_LIKE,,}"

# 根据发行版选择 sudo 组
SUDO_GROUP="sudo"
case "$DISTRO_ID" in
  debian|ubuntu|linuxmint|pop|elementary)
    SUDO_GROUP="sudo"
    ;;
  centos|rhel|fedora|rocky|alma|ol|amazonlinux)
    SUDO_GROUP="wheel"
    ;;
  arch)
    SUDO_GROUP="wheel"
    ;;
  *)
    if [[ "$DISTRO_LIKE" =~ (rhel|fedora) ]]; then
      SUDO_GROUP="wheel"
    else
      SUDO_GROUP="sudo"
    fi
    ;;
esac

# 确保 sudo 组存在（wheel 在某些最小化系统上可能不存在）
if ! getent group "$SUDO_GROUP" >/dev/null; then
  groupadd "$SUDO_GROUP"
  echo "已创建缺失的 sudo 组：$SUDO_GROUP"
fi

# 获取系统默认 shell（优先 /etc/default/useradd 中的配置）
DEFAULT_SHELL="/bin/bash"
if [[ -r /etc/default/useradd ]]; then
  SHELL_FROM_CFG=$(awk -F= '/^SHELL=/ {print $2}' /etc/default/useradd | tr -d '"')
  if [[ -n "$SHELL_FROM_CFG" ]]; then
    DEFAULT_SHELL="$SHELL_FROM_CFG"
  fi
fi

# 交互式获取用户名
while true; do
  read -rp "请输入要添加的用户名： " USERNAME
  if [[ -z "$USERNAME" ]]; then
    echo "用户名不能为空，请重新输入。"
    continue
  fi
  if [[ ! "$USERNAME" =~ ^[a-z_][a-z0-9_-]*$ ]]; then
    echo "用户名 \"$USERNAME\" 不符合规范（应以字母或下划线开头，只能包含小写字母、数字、下划线、减号）。"
    continue
  fi
  if id "$USERNAME" &>/dev/null; then
    read -rp "用户 '$USERNAME' 已存在，是否显示该用户信息？(y/n)： " RESPONSE
    case "${RESPONSE,,}" in
      y)
        echo "用户信息："
        id "$USERNAME"
        getent passwd "$USERNAME"
        ;;
    esac
    echo "脚本退出。"
    exit 1
  fi
  break
done

# 交互式获取并确认密码
while true; do
  read -srp "请输入 $USERNAME 的密码： " PASSWORD
  echo
  read -srp "请再次输入密码以确认： " PASSWORD_CONFIRM
  echo
  if [[ "$PASSWORD" != "$PASSWORD_CONFIRM" ]]; then
    echo "两次输入的密码不一致，请重新输入。"
    continue
  fi
  if [[ -z "$PASSWORD" ]]; then
    echo "密码不能为空，重新输入。"
    continue
  fi
  break
done

# 准备用户组（主组与用户名同名）
if ! getent group "$USERNAME" >/dev/null; then
  groupadd "$USERNAME"
fi

# 添加用户
useradd -m -g "$USERNAME" -s "$DEFAULT_SHELL" -G "$SUDO_GROUP" "$USERNAME"

# 设置密码
printf '%s:%s\n' "$USERNAME" "$PASSWORD" | chpasswd

# 确保用户确实属于 sudo/wheel 组
usermod -aG "$SUDO_GROUP" "$USERNAME" >/dev/null

# 设置 sudo 无密码
SUDOERS_FILE="/etc/sudoers.d/$USERNAME"
cat <<EOF > "$SUDOERS_FILE"
$USERNAME ALL=(ALL) NOPASSWD:ALL
EOF
chmod 0440 "$SUDOERS_FILE"

echo
echo "用户 '$USERNAME' 已创建，主组为 '$USERNAME'，默认 shell 为 '$DEFAULT_SHELL'。"
echo "已将其加入 '$SUDO_GROUP' 组并配置 sudo 无需密码。"
```