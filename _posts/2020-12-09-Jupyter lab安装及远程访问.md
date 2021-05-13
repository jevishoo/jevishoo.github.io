---
layout:     post
title:      Jupyter lab安装及远程访问
subtitle:   Python 应用
date:       2020-12-09
author:     JH
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - Python Learning    
---

### 1.Jupyter Lab安装
注意：安装版本最好为 `2.*`
```
pip install jupyterlab==2.2.6
```

### 2.远程配置

#### 1.生成jupyter密码密文
命令行输入 `Python` 后：

```python
from notebook.auth import passwd
passwd()
# 提示输入密码和确认输入，完成后得到密文。
# Enter password: 
# Verify password: 
'sha1:***'
```

#### 2.修改jupyter配置文件
命令行输入 `jupyter notebook --generate-config`生成配置文件

再输入`vim ~/.jupyter/jupyter_notebook_config.py` 修改配置内容：

```shell script
c.NotebookApp.default_url = '/lab'
c.NotebookApp.iopub_data_rate_limit = 1000000000000000000000
c.NotebookApp.ip='*'

# 指定默认目录
c.NotebookApp.notebook_dir = '/home/hoo/Jupyter'

## 修改成将之前生成的密文
c.NotebookApp.password = u'sha1:........'

c.NotebookApp.open_browser = False
c.NotebookApp.port =8888
c.ConnectionFileMixin.ip = '*'
```

#### 3.root配置jupyter自启
命令行输入  `vim /lib/systemd/system/jupyter.service` 后添加：


```shell script
[Unit]
Description=jupyterlab
After=network.target
[Service]
Type=simple
# 这里填用户名，下同
User=hoo
EnvironmentFile=/home/hoo/anaconda3/bin/jupyter-lab
ExecStart=/home/hoo/anaconda3/bin/jupyter-lab --allow-root
ExecStop=/usr/bin/pkill /home/hoo/anaconda3/bin/jupyter-lab
KillMode=process
Restart=on-failure
RestartSec=30s
[Install]
WantedBy=multi-user.target
```

保存后，命令行分别输入：
` systemctl daemon-reload`;

`systemctl enable jupyter.service`;

`service jupyter start`

另外可用 `service jupyter status` 查看进程信息，`service jupyter stop` 停止进程

### 3.Jupyter插件

- 配置nodejs环境

  ```shell
  wget https://cdn.npm.taobao.org/dist/node/v12.18.3/node-v12.18.3-linux-x64.tar.gz
  
  tar -xzvf /home/hoo/package/node-v12.18.3-linux-x64.tar.gz 
  
  mv node-v12.18.3-linux-x64 /usr/lib/nodejs
  
  ln -s /usr/lib/nodejs/bin/npm /usr/local/bin/npm
  ln -s /usr/lib/nodejs/bin/node /usr/local/bin/node
  
  node -v
  npm -v
  ```

- 安装插件

  ```shell
  jupyter labextension install @jupyterlab/toc
  jupyter labextension install @krassowski/jupyterlab-lsp@1.1.2
  	pip install jupyter-lsp==0.9.1
  	pip install python-language-server
  jupyter labextension install @krassowski/jupyterlab_go_to_definition
  jupyter labextension install jupyterlab-drawio
  jupyter labextension install @jupyter-widgets/jupyterlab-manager
  jupyter labextension install @jupyterlab/geojson-extension
  jupyter labextension install jupyterlab-plotly
  jupyter labextension install @mflevine/jupyterlab_html
  jupyter labextension install @jupyterlab/latex
  jupyter labextension install jupyterlab-tailwind-theme
  jupyter labextension install jupyterlab-interactive-dashboard-editor
  ```

- 
