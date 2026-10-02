# Introduction to NRP

This presentation was prepared with [sli.dev](https://sli.dev/). Install a few dependencies globally to ensure you can view everything correctly in `slides.md`

Prerequisites.

## 1. Install `npm` and `NodeJS`

Go to the [nodeJS](https://nodejs.org/en/download/) website an download the LTS installer for your machine.

This will install both `npm` and `nodeJS` onto your system.

Verify your installation with the following commands


```bash
node -v

npm -v

```

## 2. Install slidev

The command below will install slidev globally on your system. If you encounter errors during first install, re-run with `sudo`
```bash
npm i -g @slidev/cli
```

Confirm your installation:

```bash
which slidev

```


## 3. Install Liveshell

The Liveshell add-on is required to enable a dynamic shell within the presentation slides. Install with the following commands;

```bash
# 1. Install ttyd (or with apt, pacman or scoop)
brew install ttyd

# 2. Add the addon
npm install slidev-addon-liveshell
```

For additional features, consult the [docs](https://github.com/Moussto/slidev-addon-liveshell) 





To start the slide show:

```bash
slidev
```


Open you browser and visit http://localhost:3030
