---
title: githubaction
date created: 2023-08-01
date modified: 2023-08-01
---

## githubaction的妙用

### 编译发布到githubpage中

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - hugo

  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2
      
      - name: Checkout submodules
        run: git submodule update --init --recursive
        
      - name: config1 
        run: rm -rf content/.obsidian content/cedict_ts.u8 content/Extras/Templates  && mv content/*.md content/Atlas && find content/ -name "*.md" | xargs -I file  mv -f file content &&  mv content/AboutTheGarden.md content/_index.md 
      
      - name: config2
        run: "ls content/ && grep -lr --null 'title' content/* | xargs -0 sed -i -E -r 's/title: (.*)/title: \"\\1\"/g'"
      
      - name: config3 
        run: rm -rf content/*.md-E

      
      - name: Build Link Index
        uses: jackyzha0/hugo-obsidian@v2.20
        with:
          index: true
          input: content
          output: assets/indices
          root: .


      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.96.0'
          extended: true

      - name: Build
        run: hugo --minify --debug

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: master  # deploying branch
          cname: clickear.github.io
```

### 打包多个环境

``` yaml
name: Windows

on:
  push:
    tags:
      - '*'           # Push events to every tag not containing /
jobs:
  build:
    runs-on: windows-latest
    permissions: write-all
    steps:
      - uses: actions/checkout@v2

      - name: Add msbuild to PATH
        uses: microsoft/setup-msbuild@v1.0.2

      - uses: actions/checkout@v3
      - name: Set up JDK 8
        # 安装jdk8环境(且要含有javafx)
        uses: actions/setup-java@v1
        with:
          java-version: '1.8'
          java-package: 'jdk+fx'

      - name: Build with Maven
        # 执行打包的mvn命令
        run: mvn jfx:native `-Dmaven.test.skip=true

      # 移动打包文件
      - run: mkdir staging && cp -R target/jfx/native/* staging
      - run: tree target/jfx/native/
      - run: tree staging/
      - run: ls staging/


#      - uses: actions/upload-artifact@v2
#        with:
#          name: windows
#          path: D:\a\spider_client\spider_client\target\jfx\native\trex-stateless-gui
#
##
#
      - name: Archive Release
        uses: thedoctor0/zip-release@0.7.1
        with:
          type: 'zip'
          filename: 'release.zip'
          path: 'staging'
          exclusions: '*.git* /*node_modules/* .editorconfig'
#      - name: Upload Release
#        uses: ncipollo/release-action@v1.12.0
#        with:
#          artifacts: "release.zip"
#          token: ${{ secrets.GITHUB_TOKEN }}


#      - name: Set Release version env variable
#        run: |
#          echo "RELEASE_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)" >> $GITHUB_ENV
#      - name: "Build & test"
#        run: |
#          echo "done!"
      # 创建一个release,并将打包后的文件上传到附件
      - name: Automatic Releases
        # You may pin to the exact commit or the version.
        # uses: marvinpinto/action-automatic-releases@919008cf3f741b179569b7a6fb4d8860689ab7f0
        uses: marvinpinto/action-automatic-releases@latest
        with:
          # GitHub secret token
          repo_token: "${{ secrets.GITHUB_TOKEN }}"
          automatic_release_tag: ${{github.ref_name}}
          prerelease: false
          title: ${{github.ref_name}}
          # Assets to upload to the release
          files: |
            release.zip
```

### 定时触发

## 常用指南

``` yaml
- uses: actions/checkout@v3
      - name: Set up JDK 8
        # 安装jdk8环境(可含有javafx)
        uses: actions/setup-java@v1
        with:
          java-version: '1.8'
          java-package: 'jdk+fx'

      - name: Build with Maven
        # 执行打包的mvn命令
        run: mvn jfx:native `-Dmaven.test.skip=true

      # 移动打包文件
      - run: mkdir staging && cp -R target/jfx/native/* staging
      # 压缩归档成zip包
      - name: Archive Release
        uses: thedoctor0/zip-release@0.7.1
        with:
          type: 'zip'
          filename: 'release.zip'
          path: 'staging'
          exclusions: '*.git* /*node_modules/* .editorconfig'
      # 创建一个release,并将打包后的文件上传到附件
      - name: Automatic Releases
        uses: marvinpinto/action-automatic-releases@latest
        with:
          # GitHub secret token
          repo_token: "${{ secrets.GITHUB_TOKEN }}"
          automatic_release_tag: ${{github.ref_name}}
          prerelease: false
          title: ${{github.ref_name}}
          # Assets to upload to the release
          files: |
            release.zip
```
