### tmux 并行运行脚本

  

```shell

tmux new-session -d -s "parallel"

  

# script1

tmux new-window -t "parallel" -n "script1"

tmux send-keys -t "parallel:script1" 'sh script1.sh' C-m

  

# script2

tmux new-window -t "parallel" -n "script2"

tmux send-keys -t "parallel:script2" 'sh script2.sh' C-m

  

# scriptn...

...

  

# result

echo "===== result ====="

echo "窗口列表:"

echo "1. script1 (prefix + 1)"

echo "2. script2 (prefix + 2)"

echo ...

echo "运行tmux attach -t <win_name>附加到对应窗口"

```