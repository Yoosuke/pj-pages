# pj-pages
GitHub Actions 学習用REPOSITORY

このYAMLファイルは、Jekyllプロジェクトに対してCI/CD（継続的インテグレーションと継続的デリバリー）を設定するものです。具体的には、以下のような流れで動作します。

### イベントトリガー

- `develop` ブランチにプッシュがあった場合
- リリースタグが `v*`（例：`v1.0`, `v1.2.3` など）の形式でプッシュされた場合

以上のいずれかのイベントが発生すると、GitHub Actionsがトリガーされます。

### ジョブとステップ

1. **Build ジョブ**

    - **環境**: Ubuntu最新版
    - **ステップ**:
        1. リポジトリをチェックアウト
        2. Ruby 2.7 をセットアップ
        3. Jekyll サイトをビルド（依存関係のインストールとビルドの実行）

2. **Merge to main ジョブ**

    - **条件**: リリースタグ（`v*`）がプッシュされた場合のみ実行
    - **ステップ**:
        1. リポジトリをチェックアウト
        2. Gitのユーザー情報を設定
        3. `main` ブランチに `develop` ブランチをマージし、それをリモートの `main` ブランチにプッシュ

3. **Deploy ジョブ**

    - **条件**: リリースタグ（`v*`）がプッシュされた場合のみ実行
    - **ステップ**:
        - GitHub Pagesに `_site` ディレクトリをデプロイ

### 条件付き実行

- `merge-to-main` および `deploy` ジョブは、リリースタグが `v*` の形式でプッシュされた場合のみ実行されます（`if: startsWith(github.ref, 'refs/tags/v')`）。

### 依存関係

- `merge-to-main` ジョブは `build` ジョブが成功した後に実行されます（`needs: build`）。
- `deploy` ジョブは `merge-to-main` ジョブが成功した後に実行されます（`needs: merge-to-main`）。

このように、このYAMLファイルは `develop` ブランチへのプッシュや特定のリリースタグをトリガーとして、ビルド、マージ、デプロイといった一連の作業を自動で行います。

```yaml
name: Jekyll CI/CD

on:
  push:
    branches:
      - develop
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Generate Gemfile
      run: |
        echo "source 'https://rubygems.org'" > Gemfile
        echo "gem 'jekyll', '~> 4.2'" >> Gemfile
        
    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: 2.7
        
    - name: Install dependencies and build site
      run: |
        gem install bundler
        bundle install
        bundle exec jekyll build

  merge-to-main:
    runs-on: ubuntu-latest
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Configure Git
      run: |
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
    - name: Merge to main
      run: |
        git fetch origin main
        git checkout main
        git merge --no-ff "${GITHUB_REF}"
        git push origin main

  deploy:
    runs-on: ubuntu-latest
    needs: merge-to-main
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./_site


```