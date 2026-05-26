# git-filter-repo 動作検証レポート

## 概要

このリポジトリを検証用の使い捨てリポジトリとして利用し、`git-filter-repo` で以下の 2 つが実現できるかを確認しました。

1. 特定ファイルを Git 履歴から削除する
2. 特定の文字列を Git 履歴全体で置換し、機密情報を履歴から除去する

どちらのシナリオも、検証用履歴の作成、GitHub への push、`git-filter-repo` による履歴書き換え、`--force` push、fresh clone での再確認まで実施しています。

## 検証環境

- リポジトリ: `tamagokakedon/git-filter-repo-test`
- レポート反映先ブランチ: `main`
- 検証用ブランチ:
  - `scenario-file-removal-source`
  - `scenario-string-replace-source`
- `git-filter-repo` の導入方法:
  - `python -m pip install --user git-filter-repo`
- この環境での実行方法:
  - `python -m git_filter_repo ...`

## 実施したこと

### 1. `git-filter-repo` の導入

実行コマンド:

```powershell
python -m pip install --user git-filter-repo
```

結果:

```text
Successfully installed git-filter-repo-2.47.0
WARNING: The script git-filter-repo.exe is installed in
'C:\Users\tamag\AppData\Roaming\Python\Python314\Scripts' which is not on PATH.
```

確認できたこと:

- 初期状態では `git filter-repo` は Git サブコマンドとして使えませんでした。
- `pip` で導入は成功しました。
- 実行ファイルの配置先が `PATH` に入っていなかったため、この検証では `python -m git_filter_repo` で実行しました。

### 2. 検証用の履歴を作成

2 つのシナリオが混ざらないように、`main` から別々のブランチを作って検証しました。

#### A. ファイル削除シナリオ用ブランチ

実行コマンド:

```powershell
git switch -C scenario-file-removal-source main
git add fixtures
git commit -m "Add file-removal verification fixtures"
git add fixtures
git commit -m "Update removable file history"
git push -u origin scenario-file-removal-source
```

作成したファイル:

- `fixtures/file-removal/classified.txt`
- `fixtures/shared/notes.txt`

作成した履歴:

```text
083c8fc Update removable file history
b9ded18 Add file-removal verification fixtures
7b5edc9 Initial commit
```

#### B. 文字列置換シナリオ用ブランチ

実行コマンド:

```powershell
git switch -C scenario-string-replace-source main
git add fixtures
git commit -m "Add string-replace verification fixtures"
git add fixtures
git commit -m "Expand secret exposure history"
git push -u origin scenario-string-replace-source
```

作成したファイル:

- `fixtures/string-replace/app.env`
- `fixtures/string-replace/notes.txt`
- `fixtures/string-replace/audit.log`

検証用に埋め込んだ文字列:

```text
SECRET_TOKEN_ALPHA_12345
```

作成した履歴:

```text
7aece49 Expand secret exposure history
69dab31 Add string-replace verification fixtures
7b5edc9 Initial commit
```

## シナリオ 1: 特定ファイルを履歴から削除する

### 目的

`scenario-file-removal-source` ブランチ上の `fixtures/file-removal/classified.txt` を、到達可能な履歴全体から削除すること。

### 書き換えコマンド

```powershell
python -m git_filter_repo --force --invert-paths --path fixtures/file-removal/classified.txt
```

### 書き換え前の確認

実行コマンド:

```powershell
git --no-pager log --oneline -- fixtures/file-removal/classified.txt
git rev-list --all -- fixtures/file-removal/classified.txt
git grep -n "tracking-id: FILE-REMOVAL-001" $(git rev-list --all)
```

結果:

```text
083c8fc Update removable file history
b9ded18 Add file-removal verification fixtures

before_path_commit_count=2
before_tracking_id_hits=2
```

読み取れること:

- 対象ファイルは 2 commit にまたがって履歴に存在していました。
- 検索用に入れた識別文字列も履歴上で 2 箇所ヒットしました。

### 書き換え時の出力

結果抜粋:

```text
NOTICE: Removing 'origin' remote
HEAD is now at 3a93c29 Update removable file history
```

重要な注意点:

- `git-filter-repo` 実行後、`origin` remote が自動的に削除されました。
- そのため、push 前に remote の再登録が必要でした。

実行コマンド:

```powershell
git remote add origin https://github.com/tamagokakedon/git-filter-repo-test.git
git push --force origin scenario-file-removal-source
```

### 書き換え後の確認

実行コマンド:

```powershell
git rev-list --all -- fixtures/file-removal/classified.txt
git show HEAD:fixtures/shared/notes.txt
```

結果:

```text
after_path_commit_count=0

verification baseline
this file should remain after history rewrite
updated alongside the removable file
```

確認できたこと:

- 対象ファイルは到達可能な履歴から見えなくなりました。
- 同じブランチ上の残すべきファイルはそのまま保持されました。

### fresh clone での再確認

実行コマンド:

```powershell
git clone --branch scenario-file-removal-source --single-branch https://github.com/tamagokakedon/git-filter-repo-test.git <verify-dir>
git rev-list --all -- fixtures/file-removal/classified.txt
git show HEAD:fixtures/shared/notes.txt
```

結果:

```text
verify_path_commit_count=0
```

### このシナリオで検証できたこと

- 特定パスをブランチ履歴全体から削除できる
- 履歴を書き換えるため commit hash は変化する
  - 書き換え前: `083c8fc`
  - 書き換え後: `3a93c29`
- `--force` push 後、fresh clone でも削除結果を確認できる

## シナリオ 2: 文字列を履歴全体で置換する

### 目的

`scenario-string-replace-source` ブランチ上に含まれている `SECRET_TOKEN_ALPHA_12345` を、履歴全体で `REDACTED_TOKEN` に置換すること。

### 置換定義ファイル

内容:

```text
SECRET_TOKEN_ALPHA_12345==>REDACTED_TOKEN
```

### 書き換えコマンド

```powershell
python -m git_filter_repo --force --replace-text replacements.txt
```

### 書き換え前の確認

実行コマンド:

```powershell
git grep -n "SECRET_TOKEN_ALPHA_12345" $(git rev-list --all)
git grep -n "REDACTED_TOKEN" $(git rev-list --all)
```

結果:

```text
before_old_secret_hits=6
before_new_secret_hits=0
```

読み取れること:

- 元の機密文字列は履歴上で 6 箇所ヒットしていました。
- 置換後文字列は書き換え前には存在していませんでした。

### 書き換え時の出力

結果抜粋:

```text
NOTICE: Removing 'origin' remote
HEAD is now at ae50334 Expand secret exposure history
```

push 前に実行したコマンド:

```powershell
git remote add origin https://github.com/tamagokakedon/git-filter-repo-test.git
git push --force origin scenario-string-replace-source
```

### 書き換え後の確認

実行コマンド:

```powershell
git grep -n "SECRET_TOKEN_ALPHA_12345" $(git rev-list --all)
git grep -n "REDACTED_TOKEN" $(git rev-list --all)
git show HEAD:fixtures/string-replace/app.env
git show HEAD:fixtures/string-replace/audit.log
```

結果:

```text
after_old_secret_hits=0
after_new_secret_hits=6

API_URL=https://example.test
API_TOKEN=REDACTED_TOKEN
FEATURE_FLAG=true
BACKUP_TOKEN=REDACTED_TOKEN

2026-05-27 token seen: REDACTED_TOKEN
2026-05-27 rotation pending
```

確認できたこと:

- 元の機密文字列は履歴から消えました。
- 置換後文字列に置き換わった状態で履歴が再構成されました。

### fresh clone での再確認

実行コマンド:

```powershell
git clone --branch scenario-string-replace-source --single-branch https://github.com/tamagokakedon/git-filter-repo-test.git <verify-dir>
git grep -n "SECRET_TOKEN_ALPHA_12345" $(git rev-list --all)
git grep -n "REDACTED_TOKEN" $(git rev-list --all)
```

結果:

```text
verify_old_secret_hits=0
verify_new_secret_hits=6
```

### このシナリオで検証できたこと

- 特定文字列を履歴全体で一括置換できる
- commit hash は書き換え後に変化する
  - 書き換え前: `7aece49`
  - 書き換え後: `ae50334`
- fresh clone でも置換済みの結果を確認できる

## 今回の検証で分かった `git-filter-repo` の実用性

今回の結果から、チームでは `git-filter-repo` を使って少なくとも次のことが実施できます。

1. 誤って commit したファイルを履歴から除去する
2. 漏えいしたトークンや機密文字列を履歴から置換除去する
3. 履歴を書き換えた上で GitHub に force-push する
4. ローカル作業ツリーだけでなく fresh clone で結果確認する

あわせて、運用上の前提も確認できました。

- 履歴を書き換えると commit hash は変わる
- remote への反映には force-push が必要
- 既存 clone や作業ブランチは rewrite 前後で不整合になる
- `git-filter-repo` 実行後は `origin` が外れることがある

## チーム運用に向けて文書化すべきこと

チームで安全に運用するためには、少なくとも以下をドキュメント化して展開する必要があります。

### 1. どのケースで履歴改変を許可するか

文書化すべき内容:

- どの事故で履歴改変を実施するか
- 誰が実施判断・承認を行うか
- 保護ブランチに対して例外運用が必要か
- どの ref を改変対象に含めるか

### 2. 標準手順書

最低限、以下の流れを runbook として固定するのが望ましいです。

1. 対象ブランチ、対象ファイル、対象文字列を特定する
2. 一時的に merge / push を止める
3. 書き換え前のバックアップ参照を作る
4. fresh clone 上で `git-filter-repo` を実行する
5. 履歴レベルの確認コマンドで結果を検証する
6. 必要に応じて remote を再登録する
7. force-push で反映する
8. チームに clone の更新方法を周知する

### 3. 検証チェックリスト

次のような確認コマンドをチーム標準として文書化すべきです。

```powershell
git rev-list --all -- <path>
git grep -n "<secret>" $(git rev-list --all)
git --no-pager log --oneline -- <path>
git clone --single-branch --branch <branch> <repo> <verify-dir>
```

証跡として残すべき内容:

- 対象パスまたは対象文字列
- 書き換え対象のブランチ / tag / ref
- 実行コマンド
- 実行前後の結果
- 最終的に push した commit hash

### 4. チーム向け通知テンプレート

周知文には少なくとも以下を含めるべきです。

- どのブランチを書き換えたか
- なぜ書き換えたか
- 実際の機密情報ならローテーション済みか
- 開発者は reclone すべきか reset で対応するか
- PR ブランチを作り直す必要があるか

### 5. 事後対応

機密情報漏えい時は、履歴改変だけでは不十分です。以下も手順化すべきです。

- 資格情報のローテーション
- トークン失効
- CI ログ、成果物、release、tag、外部 mirror の確認
- `.gitignore` や secret scanning の見直し
- pre-commit hook / commit hook の導入検討

### 6. 開発者向け復旧手順

チーム向けには、まずは以下のような安全側の案内が実用的です。

1. 手元の未退避変更を保存する
2. リポジトリを fresh clone し直す
3. 必要なら進行中ブランチを新しい履歴から作り直す

既存 clone をそのまま復旧させる運用を採る場合は、どの条件なら安全か、どの reset 手順を使うかを明文化する必要があります。

## チーム向けに用意するとよい文書

最低限、次の 4 種類があると運用しやすくなります。

1. **履歴改変 runbook**: 承認から実施、検証、反映までの標準手順
2. **機密情報漏えい対応手順**: 履歴改変に加えてローテーションと周知を含めた手順
3. **開発者向け復旧ガイド**: rewrite 後に各自が何をすべきか
4. **検証チェックリスト**: 成功判定に必要なコマンドと証跡

## 最終状態

- 現在のレポート掲載ブランチ: `main`
- ファイル削除シナリオの書き換え後先頭 commit: `3a93c29`
- 文字列置換シナリオの書き換え後先頭 commit: `ae50334`

このレポートは、`git-filter-repo` をチームで運用する際の叩き台として利用できます。

