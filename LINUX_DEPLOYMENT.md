# 医学图像分析系统 Linux 部署指南

本文档详细介绍了如何在 Linux 系统上部署医学图像分析系统。本项目基于 Python 和 Flask 开发，使用深度学习模型进行医学图像处理和分析。通过本指南，您将了解Linux基础操作、项目部署和服务管理等内容。

## 前提条件

在开始之前，请确保您的Linux系统已安装以下软件：
- Python 3.7或更高版本
- pip（Python包管理器）
- git（可选，用于克隆代码）
- SQLite（默认数据库）或MySQL（可选，如需使用）

**注意：** 由于项目使用深度学习库（如TensorFlow），建议系统至少具有4GB内存和支持CUDA的GPU（可选，用于加速处理）。

## 第一步：安装基础依赖

### 更新系统软件包

```bash
sudo apt update && sudo apt upgrade -y
```

### 安装Python和pip

如果您的系统尚未安装Python 3：

```bash
sudo apt install python3 python3-pip python3-venv -y
```

## 第二步：获取项目代码

### 使用git克隆代码（推荐）

```bash
sudo apt install git -y
git clone https://your-repository-url/medical-image-analysis.git
cd medical-image-analysis
```

### 或手动下载和解压

如果没有git，也可以手动下载项目文件并解压：

```bash
# 假设您已下载项目压缩包
tar -xzf medical-image-analysis.tar.gz
cd medical-image-analysis
```

## 第三步：创建虚拟环境（推荐）

创建Python虚拟环境可以避免依赖冲突：

```bash
# 创建虚拟环境
python3 -m venv venv

# 激活虚拟环境
source venv/bin/activate

# 确认已激活虚拟环境
which python  # 输出应指向venv/bin/python
```

## 第四步：安装项目依赖

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

## 第五步：配置环境变量

编辑`.env`文件，根据您的Linux环境进行配置：

```bash
# 使用nano编辑器打开.env文件
nano .env

# 修改配置项
# 例如：在生产环境中关闭DEBUG模式
DEBUG=False
HOST=0.0.0.0  # 允许所有IP访问
PORT=8000

# 如需使用MySQL，请配置相应的数据库连接信息
# DATABASE_URL=mysql+pymysql://username:password@localhost/health_db

# 保存并退出（按Ctrl+O，然后按Enter，最后按Ctrl+X）
```

## 第六步：初始化数据库

首次运行前，需要初始化数据库：

```bash
# 运行初始化命令
python -c "from app import app, init_db; with app.app_context(): init_db()"
```

## 第七步：运行应用

### 开发模式（调试模式）

```bash
python app.py
```

### 生产模式（使用Gunicorn）

对于生产环境，建议使用Gunicorn作为WSGI服务器：

```bash
# 安装Gunicorn
pip install gunicorn

# 运行应用
gunicorn -w 4 -b 0.0.0.0:8000 app:app
```

参数说明：
- `-w 4`：使用4个工作进程
- `-b 0.0.0.0:8000`：绑定到所有IP的8000端口

## 第八步：使用systemd管理服务（高级）

为了让应用能够在系统启动时自动运行，并在意外关闭时自动重启，可以使用systemd创建服务。

### 创建服务文件

```bash
sudo nano /etc/systemd/system/medical-image.service
```

### 编辑服务文件内容

```ini
[Unit]
Description=Medical Image Analysis Service
After=network.target

[Service]
User=your-username
Group=your-groupname
WorkingDirectory=/path/to/medical-image-analysis
ExecStart=/path/to/medical-image-analysis/venv/bin/gunicorn -w 4 -b 0.0.0.0:8000 app:app
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### 启用并启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl start medical-image.service
sudo systemctl enable medical-image.service
```

### 服务管理命令

```bash
# 查看服务状态
sudo systemctl status medical-image.service

# 停止服务
sudo systemctl stop medical-image.service

# 重启服务
sudo systemctl restart medical-image.service

# 查看服务日志
sudo journalctl -u medical-image.service
```

## 第九步：配置Nginx作为反向代理（可选）

如果您希望使用域名访问应用，可以配置Nginx作为反向代理。

### 安装Nginx

```bash
sudo apt install nginx -y
```

### 创建Nginx配置文件

```bash
sudo nano /etc/nginx/sites-available/medical-image
```

### 编辑配置文件内容

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 静态文件配置（如果有）
    location /static {
        alias /path/to/medical-image-analysis/static;
        expires 30d;
    }
}```

### 启用配置并重启Nginx

```bash
sudo ln -s /etc/nginx/sites-available/medical-image /etc/nginx/sites-enabled/
sudo nginx -t  # 测试配置是否正确
sudo systemctl restart nginx
```

## 第十步：Linux基础命令学习

在部署过程中，您可能需要使用以下Linux基础命令：

```bash
# 查看当前目录
pwd

# 列出目录内容
ls -la

# 创建目录
mkdir directory-name

# 切换目录
cd directory-name

# 查看文件内容
cat file-name
tail -f file-name  # 实时查看文件更新

# 复制文件
cp source-file destination-file

# 移动文件
mv source-file destination-file

# 删除文件
rm file-name

# 查看进程
ps aux | grep python

# 终止进程
kill process-id
kill -9 process-id  # 强制终止

# 查看网络连接
netstat -tuln
ss -tuln

# 检查端口占用
lsof -i :port-number
```

## 常见问题解决

### 端口被占用

```bash
# 查找占用指定端口的进程
lsof -i :5000

# 终止该进程
kill -9 process-id
```

### 数据库连接错误

确保您的数据库服务正在运行，并且连接信息正确。

对于MySQL：
```bash
sudo systemctl status mysql
sudo systemctl start mysql
```

### 权限问题

如果遇到权限错误，可以尝试：
```bash
# 更改文件所有者
chown -R your-username:your-groupname /path/to/medical-image-analysis

# 更改文件权限
chmod -R 755 /path/to/medical-image-analysis
```

## 学习资源

通过完成本项目的Linux部署，您已经学习了以下技能：
- Linux系统基础操作
- Python虚拟环境管理
- 项目依赖安装
- 环境变量配置
- 服务管理（使用systemd）
- Web服务器配置（使用Nginx）
- 数据库配置和管理

这些技能对于初级开发者来说是非常宝贵的，特别是在当今云原生和容器化的时代。

祝您学习愉快！