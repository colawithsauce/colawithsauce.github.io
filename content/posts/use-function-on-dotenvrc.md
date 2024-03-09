+++
title = "在 .envrc 中定义函数"
author = ["colawithsauce"]
date = 2024-01-22T11:30:00+08:00
categories = ["article"]
draft = false
+++

参考 <https://github.com/direnv/direnv/issues/73#issuecomment-152284914>

在 `~/.direnvrc` 中写到

```sh
export_function() {
  local name=$1
  local alias_dir=$PWD/.direnv/aliases
  mkdir -p "$alias_dir"
  PATH_add "$alias_dir"
  local target="$alias_dir/$name"
  if declare -f "$name" >/dev/null; then
    echo "#!$SHELL" > "$target"
    declare -f "$name" >> "$target" 2>/dev/null
    # Notice that we add shell variables to the function trigger.
    echo "$name \$*" >> "$target"
    chmod +x "$target"
  fi
}
```

这样，我们就能够在 `.envrc` 中写到

```sh
fun () {
    echo "happy world!"
}

export_function fun
```
