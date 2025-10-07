
## conda



```shell

# 路径列表

export CONDA_PATHS=(

    /home/arch/miniconda3/bin/conda

    /data/miniconda3/bin/conda

    $HOME/miniconda3/bin/conda

)

  

# 定义 conda 函数

conda() {

    echo "[Lazy Load] Initializing Conda..."

    unfunction conda  # 移除临时函数，避免重复加载

  

    # 遍历路径

    for conda_path in $CONDA_PATHS; do

        if [[ -f $conda_path ]]; then

            echo "Found Conda at: $conda_path"

            eval "$($conda_path shell.zsh hook)"  # 初始化 Conda

            conda "$@"

            return

        fi

    done

  

    # 未找到 Conda

    echo "Error: No Conda installation found in the following paths:"

    for path in $CONDA_PATHS; do

        echo "  - $path"

    done

    return 1

}

```



## nvm

  

```shell

function nvm() {

    echo "Lazy loading nvm upon first invocation..."

    unfunction nvm

    export NVM_DIR="$HOME/.nvm"

    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

    nvm "$@"

}

```