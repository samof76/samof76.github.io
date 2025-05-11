---
layout: post
title: asdf goes go
category: asdf, go, setup, fish
---

I use [asdf](https://asdf-vm.com/) for installing and managing all my package versions, and recently with the realse of 0.16.0 version, the asdf moved to an entirely go with single binary installation. Since I was already using the asdf, and use them on multiple machine, this records how to upgrade from the legacy asdf to the new asdf.

## Download asdf

The moment I was writing this, the current version of asdf was `v0.16.7` and since I use the x86_64 architecture with Linux; downloading the right executable looked like this.

```
pushd /tmp
# Download the asdf tarball (which you already have)
wget https://github.com/asdf-vm/asdf/releases/download/v0.16.7/asdf-v0.16.7-linux-amd64.tar.gz
# Download the MD5 checksum file
wget https://github.com/asdf-vm/asdf/releases/download/v0.16.7/asdf-v0.16.7-linux-amd64.tar.gz.md5
# Compare the calculated MD5 with the expected MD5
echo "$(cat asdf-v0.16.7-linux-amd64.tar.gz.md5)  asdf-v0.16.7-linux-amd64.tar.gz" | md5sum -c
# If `OK` then do the following
tar -xvf asdf-v0.16.7-linux-amd64.tar.gz
cp asdf ~/.local/bin
chmod +x ~/.local/bin
popd
```

That takes care of install it to the `PATH`. But since I have already got legacy installation I would want to reuse already installed tools.

## Setting up the environment

Now that I asdf in my `PATH`, I would need to setup my `fish` shell environment to use the latest asdf but the existing installed and versioned tools. Here we go.

```
echo "set -gx ASDF_DATA_DIR /home/msv/.asdf" >> ~/.config/fish/config.fish
echo "set -gx PATH $ASDF_DATA_DIR/shims $PATH" >> ~/.config/fish/config.fish
```

Now to source it.

```
source ~/.config/fish/config.fish
```

> IMPORTANT: There are many breacking changes expected so we need to be wary about them. [Here](https://asdf-vm.com/guide/upgrading-to-v0-16#breaking-changes) is the complete list.
