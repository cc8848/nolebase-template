- [20个非常实用的 Shell 脚本实例 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/596040029)

1.自动备份文件或目录：

```text
#!/bin/bash

# 设置备份目录
backup_dir="/path/to/backup/dir"

# 设置要备份的文件或目录
files_to_backup="/path/to/files /path/to/dir"

# 创建一个日期时间戳
timestamp=$(date +%F_%T)

# 备份文件
tar -czvf "${backup_dir}/backup_${timestamp}.tar.gz" ${files_to_backup}
```

2.批量重命名文件：

```text
#!/bin/bash

# 设置文件扩展名
extension=".txt"

# 遍历当前目录下的所有文件
for file in *${extension}
do
  # 获取文件名（不包括扩展名）
  filename=$(basename "${file}" "${extension}")
  # 重命名文件
  mv "${file}" "${filename}_new${extension}"
done
```

3.批量删除文件

```text
#!/bin/bash

# 设置文件扩展名
extension=".tmp"

# 遍历当前目录下的所有文件
for file in *${extension}
do
  # 删除文件
  rm "${file}"
done
```

4.查找并删除指定名称的文件：

```text
#!/bin/bash

# 设置文件名
filename="example.txt"

# 查找并删除文件
find . -name "${filename}" -delete
```

5.查找并替换文件内容：

```text
#!/bin/bash

# 设置要查找的字符串
search_string="old_string"

# 设置要替换成的字符串
replace_string="new_string"

# 查找并替换文件内容
find . -type f -exec sed -i "s/${search_string}/${replace_string}/g" {} \;
```

6.批量创建文件：

```text
#!/bin/bash

# 设置文件名前缀
prefix="file"

# 设置文件数量
num_files=10

# 循环创建文件
for i in $(seq 1 ${num_files})
do
  touch "${prefix}${i}.txt"
done
```

7.创建文件夹并移动文件：

```text
#!/bin/bash

# 设置文件夹名称
dir_name="new_dir"

# 设置要移动的文件
files_to_move="file1.txt file2.txt"

# 创建文件夹并移动文件
mkdir ${dir_name} && mv ${files_to_move} ${dir_name}
```

8.在文件夹中查找文件：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 设置文件名
filename="example.txt"

# 在文件夹中查找文件
find ${dir_path} -name "${filename}"
```

9.计算文件夹中文件数量：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 计算文件数量
num_files=$(find ${dir_path} -type f | wc -l)

echo "Number of files in ${dir_path}: ${num_files}"
```

10.计算文件夹大小：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 计算文件夹大小
dir_size=$(du -sh ${dir_path})

echo "Size of ${dir_path}: ${dir_size}"
```

11.定时执行命令：

```text
#!/bin/bash

# 设置命令
command="echo hello"

# 设置执行周期（以秒为单位）
period=10

# 定时执行命令
while true
do
  eval ${command}
  sleep ${period}
done
```

12.发送邮件：

```text
#!/bin/bash

# 设置收件人邮箱
to="recipient@example.com"

# 设置发件人邮箱
from="sender@example.com"

# 设置邮件主题
subject="Test Email"

# 设置邮件内容
body="This is a test email."

# 发送邮件
echo "${body}" | mail -s "${subject}" -r "${from}" "${to}"
```

13.批量解压缩文件：

```text
#!/bin/bash

# 设置文件扩展名
extension=".tar.gz"

# 遍历当前目录下的所有文件
for file in *${extension}
do
  # 解压缩文件
  tar -xvzf ${file}
done
```

14.在文件夹中查找并删除文件：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 设置要删除的文件扩展名
extension=".tmp"

# 在文件夹中查找并删除文件
find ${dir_path} -name "*${extension}" -delete
```

15.批量重命名文件：

```text
#!/bin/bash

# 设置文件扩展名
extension=".old"

# 设置新文件扩展名
new_extension=".new"

# 遍历当前目录下的所有文件
for file in *${extension}
do
  # 获取文件名（不包含扩展名）
  filename=$(basename "${file}" "${extension}")
  
  # 重命名文件
  mv "${file}" "${filename}${new_extension}"
done
```

16.对文件夹中的文件按修改时间排序：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 对文件夹中的文件按修改时间排序
find ${dir_path} -type f -printf "%T@ %p\n" | sort -n
```

17.批量转换文件格式：

```text
#!/bin/bash

# 设置文件扩展名
extension=".txt"

# 设置新文件扩展名
new_extension=".md"

# 遍历当前目录下的所有文件
for file in *${extension}
do
  # 转换文件格
pandoc -s "${file}" -o "${file/${extension}/${new_extension}}"
done

注意：需要先安装Pandoc。
```

18.删除文件夹中的空文件夹：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 删除文件夹中的空文件夹
find ${dir_path} -type d -empty -delete
```

19.删除文件夹中的空文件：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 删除文件夹中的空文件
find ${dir_path} -type f -empty -delete
```

20.批量更改文件权限：

```text
#!/bin/bash

# 设置文件夹路径
dir_path="/path/to/dir"

# 设置文件权限（以八进制表示）
permission="644"

# 批量更改文件权限
find ${dir_path} -type f -exec chmod ${permission} {} \;
```