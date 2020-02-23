+++
title = "Env is Evil: ycmd server SHUT DOWN Unexpected exit code -11"
date = 2018-11-27T07:30:01+08:00
images = []
tags = ["note", "vim"]
categories = ["tool"]
draft = false
+++

## What issue?
The ycmd server SHUT DOWN (restart with ':YcmRestartServer'). Unexpected exit code -11.

## Try fix

```shell
./install.py --clang-completer
cd ~/.vim/bundle/YouCompleteMe/third_party/ycmd
cd ycmd
cp default_settings.json ..
python ycmd --options_file default_settings
python3 ./build.py
```

 Not working.

When reading source code, I found out the `build.py` uses sysconfig.
I try to add pdb.set_trace() in the first line of function GetPossiblePythonLibraryDirectories.
In pdb shell p sysconfig.get_config_var("LIBPL"), result show  path is fault, not  mine.
I try hard code absolute path in GetPossiblePythonLibraryDirectories, but no working.

## Fixed

Anyway, in ycm issues i found out miniconda3 has conflict issue.

so try:

```python
rm -rf miniconda3
python3 install.py
```

Well, it's working.
Environment implicit is evil,  explicit is better than implicit.

