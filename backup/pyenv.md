1. 安装：`curl -fsSL https://pyenv.run | bash`

2. 配置镜像源：

   ```shell
   export PYTHON_BUILD_MIRROR_URL="https://registry.npmmirror.com/-/binary/python"
   export PYTHON_BUILD_MIRROR_URL_SKIP_CHECKSUM=1
   ```

3. 安装指定版本Python：

   ```shell
   pyenv global 3.11.3
   python --version
   python -m venv ~/.venvs/python3.11.3
   source ~/.venvs/python3.11.3/bin/activate
   ```

