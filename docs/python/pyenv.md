[pyenv](https://github.com/pyenv/pyenv)

**Installation**

```shell
$ brew update
$ brew install pyenv

$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
$ echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(pyenv init - zsh)"' >> ~/.zshrc

$ source ~/.zshrc
```

**查看可以安裝的版本**

```shell
$ pyenv install --list
```

**安裝需要的版本**

```shell
$ pyenv install 3.11.11
```

**查看目前使用的版本**

```shell
$ pyenv versions
```

**設定全域版本**

```shell
$ pyenv glocal 3.11.11
```

**在個別專案切換版本**

進到專案目錄

```shell
$ pyenv local 3.11.1
```
