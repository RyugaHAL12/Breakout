# Breakout の Docker セットアップ手順

TypeScript + Canvas API + Vite で作成する Breakout の開発環境を、Docker 内で起動するための手順です。Windows + WSL 2 + Docker Desktop を前提としています。

このドキュメントは設定手順を説明するものです。記載した設定ファイルの作成やコマンド実行は、各手順に従って行ってください。

## 1. 今回の構成

普段は WSL 内のソースコードをエディタで編集し、Vite は Docker コンテナ内で起動します。ブラウザからは `http://localhost:5173/` にアクセスします。

ゲームの処理と Canvas の描画はブラウザで実行されます。コンテナ内の Vite は、HTML・JavaScript・画像などをブラウザに配信する役割です。

| 項目 | 内容 |
| --- | --- |
| プロジェクト | `~/dev/Breakout` |
| 開発環境 | Node.js 24 + Vite + TypeScript |
| コンテナ内の作業場所 | `/app` |
| Compose のサービス名 | `app` |
| ブラウザで開く URL | `http://localhost:5173/` |
| 依存パッケージ | Docker の名前付きボリュームに保存 |

この構成は開発用です。本番配信には、別途ビルドした `dist/` を Nginx などで配信する構成を使用します。

## 2. Docker Desktop と WSL を連携する

### Windows 側の設定

1. Docker Desktop を起動します。未インストールの場合は、[公式の Windows インストール手順](https://docs.docker.com/desktop/setup/install/windows-install/)に従って導入してください。
2. **Settings → General** で `Use WSL 2 based engine` が有効になっていることを確認します。環境によっては、有効済みで項目が表示されません。
3. **Settings → Resources → WSL Integration** を開きます。
4. 使用しているディストリビューション（Ubuntu など）の連携を有効にします。
5. **Apply / Apply & Restart** で設定を反映します。

使用しているディストリビューションの名前と WSL のバージョンは、Windows の PowerShell で確認できます。

```powershell
wsl -l -v
```

対象のディストリビューションが `VERSION 2` になっていることを確認してください。

### WSL 側の確認

WSL のターミナルで実行します。

```bash
docker version
docker compose version
```

`docker version` に Client と Server の両方が表示され、Compose のバージョンも表示されれば準備完了です。

Docker Desktop は、コンテナを使用している間は起動しておきます。この手順では Docker Desktop の WSL 連携を使用するため、WSL 内への Docker Engine の追加インストールは不要です。

参考：[Docker Desktop の WSL 2 連携](https://docs.docker.com/desktop/features/wsl/)

## 3. プロジェクトの前提条件を確認する

以下のファイルが存在する状態で進めます。

```text
Breakout/
├── package.json
├── package-lock.json
├── index.html
├── tsconfig.json
└── src/
```

`package.json` の `scripts` には、Vite の開発サーバーを起動する設定が必要です。現在のプロジェクトでは、次の構成になっています。

```json
{
  "dev": "vite",
  "build": "tsc && vite build",
  "preview": "vite preview"
}
```

上記は `scripts` 部分の抜粋です。`package.json` 全体を置き換えないでください。

`package-lock.json` がない場合は、WSL 内で次を実行して生成します。

```bash
cd ~/dev/Breakout
npm install
```

`package-lock.json` は Git にコミットします。Docker 内で使用する `npm ci` は、このファイルに基づいて依存パッケージをインストールします。

## 4. `.dockerignore` を作成する

`~/dev/Breakout/.dockerignore` を作成し、次の内容を記載します。

```dockerignore
node_modules
dist
.git
coverage
*.log
*.tsbuildinfo
.env
.env.*
.vscode
.idea
```

| ファイル | 役割 |
| --- | --- |
| `.gitignore` | Git の管理対象から除外する |
| `.dockerignore` | Docker のビルドに送るファイルから除外する |

ローカルの `node_modules/` をコピーせず、コンテナ内で依存パッケージをインストールするための設定です。`package.json` と `package-lock.json` は除外しません。

なお、`.dockerignore` はビルド時に適用されます。後述する `.:/app` のファイル共有には適用されません。

## 5. `Dockerfile` を作成する

`~/dev/Breakout/Dockerfile` を作成し、次の内容を記載します。ファイル名に `.txt` などの拡張子を付けないでください。

```dockerfile
FROM node:24-alpine AS dev

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--strictPort"]
```

| 指定 | 役割 |
| --- | --- |
| `FROM node:24-alpine AS dev` | Node.js 24 の環境を用意し、この段階を `dev` と名付ける |
| `WORKDIR /app` | コンテナ内の作業場所を指定する |
| `COPY package.json package-lock.json ./` | 依存関係の情報を先にコピーする |
| `RUN npm ci` | ロックファイルに基づいて依存パッケージをインストールする |
| `COPY . .` | プロジェクトのファイルをコピーする |
| `EXPOSE 5173` | 使用するポートを明示する。これだけではポートは公開されない |
| `CMD` | コンテナ起動時に Vite を起動する |

`--host 0.0.0.0` は、コンテナ外から Vite に接続するための指定です。`--strictPort` は、5173番が使用できない場合に別のポートへ自動変更せず、エラーで停止させる指定です。

## 6. `compose.yaml` を作成する

`~/dev/Breakout/compose.yaml` を作成し、次の内容を記載します。

```yaml
services:
  app:
    build:
      context: .
      target: dev
    ports:
      - "127.0.0.1:5173:5173"
    volumes:
      - .:/app
      - node_modules:/app/node_modules

volumes:
  node_modules:
```

YAML のインデントにはスペースを使ってください。

| 設定 | 役割 |
| --- | --- |
| `context: .` | 現在のディレクトリをビルド対象にする |
| `target: dev` | Dockerfile の `dev` を使用する |
| `127.0.0.1:5173:5173` | 手元の5173番ポートからコンテナの5173番ポートへ接続する |
| `.:/app` | WSL 内のプロジェクトをコンテナと共有する |
| `node_modules:/app/node_modules` | コンテナ用の依存パッケージを名前付きボリュームに保存する |

ソースコードは WSL 内のファイルをそのまま編集できます。`node_modules` は専用のボリュームを使うため、WSL 側でインストールしたものと混ざりません。

## 7. ビルドして起動する

WSL 内で直接 `npm run dev` を実行している場合は、そのターミナルで `Ctrl+C` を押して停止します。同じ5173番ポートを Docker で使うためです。

プロジェクトのディレクトリで、設定を確認して起動します。

```bash
cd ~/dev/Breakout
docker compose config
docker compose up --build
```

`docker compose config` にエラーがある場合は、設定を修正してから `up` を実行してください。

初回は Docker イメージのダウンロードと依存パッケージのインストールが行われます。

起動後、ブラウザで次の URL を開きます。

```text
http://localhost:5173/
```

### 動作確認

1. ブラウザで初期画面が表示されることを確認します。
2. 別のターミナルで `cd ~/dev/Breakout` に移動し、`docker compose ps` を実行します。
3. `app` が `Up` になっていることを確認します。
4. `src/style.css` などを一時的に編集・保存し、ブラウザに変更が反映されることを確認します。確認用の変更が不要なら元に戻します。

## 8. 普段使うコマンド

以下は、すべて `~/dev/Breakout` 内で実行します。

| 操作 | コマンド |
| --- | --- |
| 起動 | `docker compose up` |
| バックグラウンドで起動 | `docker compose up -d` |
| 起動状態の確認 | `docker compose ps` |
| ログを追跡 | `docker compose logs -f app` |
| 停止してコンテナを削除 | `docker compose down` |
| Dockerfile の変更を反映 | `docker compose up --build` |
| 起動中のコンテナでシェルを開く | `docker compose exec app sh` |

手前で起動している場合は、`Ctrl+C` でも停止できます。ログ追跡中の `Ctrl+C` は、ログ表示だけを終了します。

`docker compose down` は、共有しているソースコードや名前付きボリュームを削除しません。

### 依存パッケージを変更した場合

WSL 側で依存パッケージを追加・更新して `package.json` と `package-lock.json` が変更された場合は、コンテナ側の依存パッケージも更新します。

開発サーバーを停止してから実行してください。

```bash
docker compose down
docker compose run --rm app npm ci
docker compose up --build
```

名前付きボリュームは再ビルド後も保持されるため、イメージの再ビルドだけでは依存パッケージの変更が反映されない場合があります。`npm ci` でボリューム内の依存パッケージを更新します。

## 9. よくある問題と対処

### `docker` が見つからない

Docker Desktop が起動しているか、使用中のディストリビューションに対して WSL Integration が有効か確認してください。設定反映後も変わらない場合は、WSL のターミナルを開き直して再確認します。

### `Cannot connect to the Docker daemon` と表示される

Docker Desktop の起動が完了しているか確認し、`docker version` で Server の情報が表示されるか確認してください。

### ポートが使用中と表示される

WSL 内で直接起動した Vite や、別のコンテナが5173番ポートを使用していないか確認してください。

別のポートを使う場合は、`compose.yaml` の手元側のポートだけ変更できます。

```yaml
ports:
  - "127.0.0.1:5174:5173"
```

コンテナを停止・再作成すると、ブラウザで開く URL は `http://localhost:5174/` になります。

### ブラウザに接続できない

次のコマンドでコンテナの状態とエラーを確認します。

```bash
docker compose ps
docker compose logs app
```

`CMD` に `--host 0.0.0.0` があること、ポート指定とブラウザの URL が一致していることも確認してください。

### `npm ci` が失敗する

`package-lock.json` が存在し、`package.json` と一致していることを確認します。依存関係を手動で変更した場合は、WSL 側で `npm install` を実行してロックファイルを更新してから再試行してください。

通信エラーの場合は、ネットワークや npm レジストリへの接続状況を確認してください。

### 編集しても画面が更新されない

編集しているフォルダが `~/dev/Breakout` であることと、`compose.yaml` に `.:/app` が設定されていることを確認してください。

WSL と Windows のファイル操作の組み合わせによって変更検知が動かない場合があります。まず WSL 側のファイルシステムにあるプロジェクトを、WSL に接続したエディタから編集しているか確認します。

詳細は [Vite のファイル監視に関する説明](https://vite.dev/config/server-options.html#server-watch)を参照してください。

## 10. 設定を Git に保存する

起動と変更反映を確認できたら、必要なファイルをコミットします。

```bash
git status
git add Dockerfile compose.yaml .dockerignore AI-Docs/Docker-Setup/README.md
git diff --cached --stat
git commit -m "chore: add Docker development environment and setup guide"
git push
```

依存関係を変更した場合は、`package.json` と `package-lock.json` も追加対象に含めてください。`node_modules/` と `dist/` はコミットしません。

## 参考資料

- [Docker Desktop の Windows インストール手順](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker Desktop の WSL 2 連携](https://docs.docker.com/desktop/features/wsl/)
- [Docker Compose の入門ガイド](https://docs.docker.com/compose/gettingstarted/)
- [Docker のビルドコンテキストと `.dockerignore`](https://docs.docker.com/build/concepts/context/#dockerignore-files)
- [Vite のサーバー設定](https://vite.dev/config/server-options.html)
- [npm ci](https://docs.npmjs.com/cli/v11/commands/npm-ci/)
