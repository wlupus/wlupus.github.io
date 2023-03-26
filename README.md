# About
This repo is a personal blog based on [Hugo](https://gohugo.io/) with theme [FixIt](https://fixit.lruihao.cn/).
You can view the live website [here](https://wlupus.github.io)

# Deployment
First you need to clone this repository
```text
git clone --recurse-submodules https://github.com/wlupus/wlupus.github.io.git
```
If you've already cloned this repository without submodules, you can simply update it.
```text
git submodule update --init --recursive
```

Run `hugo server --disableFastRender` to view live pages on localhost 

# Create a new post
```text
hugo new posts/<pathname>/index.zh-cn.md
```