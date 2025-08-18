#!/bin/sh

# 颜色定义
METAL="\033[90m"
NEON_BLUE="\033[38;5;45m"
NEON_CYAN="\033[38;5;44m"
ERROR_RED="\033[38;5;196m"
SUCCESS_GREEN="\033[38;5;46m"
HIGHLIGHT="\033[38;5;208m"
DEBUG_YELLOW="\033[33m"
RESET="\033[0m"

# 检查依赖
check_dependency() {
    if ! command -v "$1" >/dev/null 2>&1; then
        echo -e "${ERROR_RED}[ERROR] 缺失组件: $1${RESET}"
        exit 1
    fi
}
check_dependency "xxd"
check_dependency "base64"
check_dependency "basename"
check_dependency "dd"

# 扫描原始脚本
sh_files=$(find . -maxdepth 1 -type f -name "*.sh" ! -name "虚无加密.sh")
if [ -z "$sh_files" ]; then
    echo -e "${METAL}[SYSTEM] 未发现可加密文件${RESET}"
    exit 1
fi

# 标题栏
echo -e "${METAL}============================================"
echo -e "| ${NEON_CYAN}虚无加密系统 v1.0.0${METAL}                    |"
echo -e "| ${NEON_BLUE}https://github.com/void-crypto${METAL}         |"
echo -e "============================================${RESET}"

# 显示文件列表
echo -e "\n${METAL}[SCAN] 已发现以下原始脚本:${RESET}"
echo -e "${METAL}--------------------------------------------${RESET}"
i=1
files=()
while IFS= read -r file; do
    files[$i]="$file"
    echo -e "${NEON_CYAN}[DEVICE-$i]${METAL} $file${RESET}"
    i=$((i + 1))
done <<< "$sh_files"
echo -e "${METAL}--------------------------------------------${RESET}"

# 选择文件
while true; do
    echo -n -e "${NEON_BLUE}[INPUT] 请指定待加密文件编号 [1-$((i-1))]: ${RESET}"
    read num
    if [ "$num" -ge 1 ] && [ "$num" -lt "$i" ] && [ -n "${files[$num]}" ]; then
        original_file="${files[$num]}"
        echo -e "${SUCCESS_GREEN}[SELECTED] 目标: $original_file${RESET}"
        break
    else
        echo -e "${ERROR_RED}[INVALID] 无效编号${RESET}"
    fi
done

# 文件名验证询问
while true; do
    echo -n -e "${NEON_BLUE}[SECURITY] 启用加密后文件名验证? (y/n): ${RESET}"
    read check_name
    case "$check_name" in
        y|Y) 
            echo -e "${SUCCESS_GREEN}[ENABLED] 验证逻辑将写入原始文件（加密前）${RESET}"
            check_name="y"
            break
            ;;
        n|N) 
            echo -e "${METAL}[DISABLED] 不启用验证${RESET}"
            check_name="n"
            break
            ;;
        *) 
            echo -e "${ERROR_RED}[ABORT] 请输入y或n${RESET}"
            ;;
    esac
done

# 生成加密文件名
original_basename=$(basename "$original_file" .sh)
encrypted_file="${original_basename}_加密.sh"
if [ "$(basename "$encrypted_file")" != "${original_basename}_加密.sh" ]; then
    echo -e "${ERROR_RED}[FATAL] 文件名生成错误，强制设置为${original_basename}_加密.sh${RESET}"
    encrypted_file="${original_basename}_加密.sh"
fi
encrypted_filename=$(basename "$encrypted_file")
echo -e "${DEBUG_YELLOW}[DEBUG] 预期加密文件名: $encrypted_filename${RESET}"

# 加密前写入验证逻辑（仅错误时显示调试信息）
if [ "$check_name" = "y" ]; then
    echo -n -e "${METAL}[WRITE] 写入验证逻辑到原始文件..."
    cp "$original_file" "${original_file}.bak"
    temp_verify=$(mktemp)
    cat > "$temp_verify" <<EOL
#!/bin/sh

# 文件名验证（从路径提取纯文件名）
current_path="\$1"
current_name=\$(basename "\$current_path")  # 提取纯文件名
expected_name="$encrypted_filename"

# 仅在验证失败时显示调试信息
if [ "\$current_name" != "\$expected_name" ]; then
    echo -e "${DEBUG_YELLOW}[DEBUG] 实际文件名: \$current_name${RESET}"
    echo -e "${DEBUG_YELLOW}[DEBUG] 预期文件名: \$expected_name${RESET}"
    echo -e "${ERROR_RED}[VERIFY_FAILED] 加密文件名错误，禁止执行${RESET}"
    exit 1
fi

EOL
    cat "$temp_verify" "$original_file" > "$original_file.tmp"
    mv "$original_file.tmp" "$original_file"
    rm -f "$temp_verify"
    echo -e " ${SUCCESS_GREEN}[DONE]${RESET}"
    echo -e "${METAL}[NOTE] 原始文件备份: ${original_file}.bak${RESET}"
fi

# 生成乱码
echo -n -e "${METAL}[OBFUSCATE] 生成混淆矩阵..."
obfuscate_file=$(mktemp)
awk -v lines=30000 -v len=90 'BEGIN{
    srand()
    chars = "abcdefghijklmnopqrstuvwxyz0123456789"
    for(i=1; i<=lines; i++){
        line="#"
        for(j=1; j<=len; j++){
            line=line substr(chars, int(rand()*36)+1, 1)
        }
        print line > "'"$obfuscate_file"'"
    }
}'
echo -e " ${SUCCESS_GREEN}[DONE]${RESET}"

# 加密流程（5层16进制 + 2层Base64）
echo -n -e "${METAL}[ENCRYPT] 执行5层16进制+2层Base64加密..."
temp_content=$(mktemp)
cat "$original_file" > "$temp_content"

# 第一步：2层Base64加密
cat "$temp_content" | base64 -w 0 > "$temp_content.tmp"
mv "$temp_content.tmp" "$temp_content"
cat "$temp_content" | base64 -w 0 > "$temp_content.tmp"
mv "$temp_content.tmp" "$temp_content"

# 第二步：5层16进制加密
count=1
while [ $count -le 5 ]; do
    xxd -p "$temp_content" | tr -d '\n' > "$temp_content.tmp"
    mv "$temp_content.tmp" "$temp_content"
    count=$((count + 1))
done

content=$(cat "$temp_content")
rm -f "$temp_content"
echo -e " ${SUCCESS_GREEN}[COMPLETE]${RESET}"

# 构建解密代码（兼容所有shell的循环语法）
decrypt_temp=$(mktemp)
cat > "$decrypt_temp" <<EOL
#!/bin/sh
# 兼容版解密逻辑：用while循环替代C风格for循环
work_dir=\$(mktemp -d)
stage1="\$work_dir/stage1"
stage2="\$work_dir/stage2"
final="\$work_dir/final"

# 步骤1：写入加密内容
echo -n "$content" > "\$stage1"

# 步骤2：解密5层16进制（用while循环，兼容所有shell）
count=1
while [ \$count -le 5 ]; do
    if ! xxd -p -r "\$stage1" > "\$stage1.tmp"; then
        echo -e "${ERROR_RED}[DECRYPT_FAILED] 第\$count层16进制解密失败${RESET}"
        rm -rf "\$work_dir"
        exit 1
    fi
    mv "\$stage1.tmp" "\$stage1"
    count=\$((count + 1))  # 原生shell算术运算，兼容性更好
done

# 步骤3：解密2层Base64
if ! base64 -d "\$stage1" > "\$stage2"; then
    echo -e "${ERROR_RED}[DECRYPT_FAILED] 第1层Base64解密失败${RESET}"
    rm -rf "\$work_dir"
    exit 1
fi
if ! base64 -d "\$stage2" > "\$final"; then
    echo -e "${ERROR_RED}[DECRYPT_FAILED] 第2层Base64解密失败${RESET}"
    rm -rf "\$work_dir"
    exit 1
fi

# 步骤4：执行解密后的脚本
sh "\$final" "\$0"

# 清理临时目录
rm -rf "\$work_dir"
EOL

# 插入位置计算
insert_pos=$((20000 + RANDOM % 10000))
[ $insert_pos -gt 30000 ] && insert_pos=30000

# 生成加密脚本
echo -e "#==========================================#" > "$encrypted_file"
echo -e "#  SYSTEM: VOID_ENCRYPT v1.0                #" >> "$encrypted_file"
echo -e "#  DATE: $(date +%Y-%m-%d)                  #" >> "$encrypted_file"
echo -e "#  STATUS: ENCRYPTED                        #" >> "$encrypted_file"
echo -e "#  EXPECTED_NAME: $encrypted_filename       #" >> "$encrypted_file"
echo -e "#==========================================#" >> "$encrypted_file"
echo -e "" >> "$encrypted_file"
echo -e "clear" >> "$encrypted_file"

# 写入乱码和解密代码
head -n $((insert_pos - 1)) "$obfuscate_file" >> "$encrypted_file"
cat "$decrypt_temp" >> "$encrypted_file"
tail -n +$insert_pos "$obfuscate_file" >> "$encrypted_file"

# 清理并恢复原始文件
rm -f "$obfuscate_file" "$decrypt_temp"
if [ "$check_name" = "y" ]; then
    mv "${original_file}.bak" "$original_file"
fi
chmod +x "$encrypted_file"

echo -n -e "${METAL}[ASSEMBLE] 生成可执行程序..."
echo -e " ${SUCCESS_GREEN}[FINISHED]${RESET}"

# 输出结果
echo -e "\n${METAL}============================================"
echo -e "| ${SUCCESS_GREEN}加密完成${METAL}                              |"
echo -e "| ${NEON_CYAN}加密文件: $encrypted_file${METAL}             |"
echo -e "| ${NEON_BLUE}执行: ./$encrypted_file${METAL}              |"
echo -e "============================================${RESET}\n"
