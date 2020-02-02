# Instructions

## 1. Prepare your repository

### 1.1. Create `upm` branch

If you do not already have `upm` branch, open you repository folder and run:

```bash
git checkout -b upm
git push --set-upstream <remote> upm
```

It will create `upm` branch in your repository and push it to remote

### 1.2. Add version.json to your project

Somewhere in your project place version.json file with folowing content:

```json
{
    "version": "0.0.20"
}
```

This file will be needed later to use as a trigger for pipeline

### 1.3. Add package.json to your project

You'll also need package.josn in order to store there information about your package. It will look something like:

```json
{
    "displayName": "",
    "category": "",
    "description": "",
    "dependencies": {},
    "keywords": [],
    "name": "",
    "unity": "",
    "version": "",
    "type": ""
}
```
## 2. Pipeline configuration

Create new github action, copy and paste following instruction placing your values on commented sections. More on that down below

```yml
name: CI

on:
  push:
    branches:
      - # 1. target branch name
    paths:
      - # 2. path to version.json file
      - ".github/workflows/main.yml"

jobs:
  build:
    env:
      PKG_ROOT: # 3. path to your package root in target branch
      VERSION_JSON: # 4. path to version.json in target branch
      PKG_JSON: # 5. path to package.json in target branch

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: copy package to temp
      run: |
        mkdir temp
        cp -r $PKG_ROOT temp
        cp $VERSION_JSON temp
        cp $PKG_JSON temp

    - name: checkout upm branch
      run: |
        git fetch --no-tags --prune --depth=1 origin +refs/heads/*:refs/remotes/origin/*
        git checkout upm

    - name: get preparation tools
      run: git clone https://github.com/Ujinjinjin/upm-preparator.git

    - name: bump package version
      run: |
        python3 upm-preparator/version_bumpinator.py "temp/${VERSION_JSON}" "temp/${PKG_JSON}"
        rm "temp/${VERSION_JSON}"

    - name: change project structure
      run: python3 upm-preparator/structure_changinator.py "temp/${PKG_ROOT}" "temp/${PKG_JSON}"

    - name: generate meta files
      run: python3 upm-preparator/meta_makinator.py

    - name: remove preparation tools
      run: rm -rf upm-preparator

    # 6. replace 'test@email.com' and 'GitHub Actions' with email and name you wish to be shown on your git tree
    - name: config git data
      run: |
        git config --global user.email "test@email.com"
        git config --global user.name "GitHub Actions"

    - name: commit, tag and push changes
      run: |
        git add -A
        git commit -m "UPM package version ${PKG_VERSION}"
        git push origin upm
        git tag $(echo $PKG_VERSION)
        git push origin $(echo $PKG_VERSION)

    - name: create release
      id: create_release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ env.PKG_VERSION }}
        release_name: "UPM ${{ env.PKG_VERSION }}"
        body: "-"
        draft: false
        prerelease: false

```

### 2.1. Target branch name

It is used to specify branch that will trigger pipeline. E.g. if you use your `master` branch as target, every time you commit on `master`, this pipeline will start if no other constraints are given.

### 2.2. Path to version.json file

But we are going to add one more constraint, because we do not want to publish package 10 times a day. So in this example we are using our created previously `version.json` file as second trigger constraint. So pipeline will run only when changes made to `version.json` on  `master` branch.

### 2.3. Path to package root

Path to root directory of your package. In some scenarios it is the root directory of your `.csproj` file.

### 2.4. Path to version.json

Path to previously created `version.json` in your target branch

### 2.5. Path to package.json

Path to previously created `package.json` in your target branch

### 2.6. git user info

In order to let you commit from machine that running this pipeline you have to specify email and name that git will use.

## 3. Result

If everything configured as it should be, considering, you have following package repository structure:

```
.
├── ArtInFart
│   ├── ArtInFart.asmdef
│   ├── ArtInFart.csproj
│   └── Worker.cs
├── package.json
├── README.md
└── version.json
```

you will see something like this on your upm branch:

```
.
├── ArtInFart.asmdef
├── ArtInFart.asmdef.meta
├── ArtInFart.csproj
├── ArtInFart.csproj.meta
├── package.json
├── package.json.meta
├── Worker.cs
└── Worker.cs.meta
```

Feel free to adjust commit message and release name as you wish, it should not break pipeline. Also, this pipeline depends on my other repository [upm-preparator](https://github.com/Ujinjinjin/upm-preparator) that helps to generate `*.meta` files, change project structure and update version. Feel free to fork both repos and play around with them, through I would recommend to keep `upm-preparator` as it currently is, unless it highly needed.

All guids for `*.meta` files are generated based on path to file or directory, so unless you rename specific file/dir it's guid will remain unchanged.

## 4. Pipeline versions

There are currently two ways to obtain `upm` branch:

1. Recreate branch on every release
2. Reuse branch on every release

### 4.1. Recreation

In order to recreate `upm` branch, you have to refer to pipeline definition on commit [0a89b1f](https://github.com/Ujinjinjin/art-in-fart-package/commit/0a89b1f0c83a1808c6ae866f682af89b429703a3). Do not just copy and paste old version of pipeline definition, some minor changes should be made. This way your repository will look like:

![](https://cdn.discordapp.com/attachments/459727475287654401/673588993727791153/unknown.png)

Benefits of this way are:

1. You use UPM branch as temporary, so you will not drag on all changes form `target` branch also on `upm` branch
2. Using tags and releases people are able to download specific version of pachage

Disadvantages are:

1. If someone wants to add specific version of package to unity from git repo, he will most probably fail, but it is not tested yet

### 4.2. Reusing

In order to reuse `upm` branch jsut refer to pipeline definition from this tutorial. This way your repository will look like:

![](https://cdn.discordapp.com/attachments/459727475287654401/673650465950400512/unknown.png)

Benefits of this way are:

1. People can add specific version of your package to unity using git url


Disadvantages are:

1. Your repo tree will contain and track all changes made to project on both `target` and `upm` branches