---
title: GitHubの草を監視してDiscordに通知するツールを作った話
tags:
  - Python
  - GitHub
  - 自動化
  - discord
  - GitHubActions
private: false
updated_at: '2025-10-09T18:01:00+09:00'
id: 8e4087fee936e7bf080f
organization_url_name: null
slide: false
ignorePublish: false
---

# GitHub の草を監視して Discord に通知するツールを作った話

## はじめに

最近、Github で草（コントリビューション）を継続して生やし始めました。
GitHub の草を継続して生やすのは、一つのモチベーション維持手段になります。でも、私はよく忘れてしまいます。
Obsidian を Github と連携して使ったりしたりしているのですが、やはり忘れてしまい、やってしまったと後悔してしまいます。

以前ネットサーフィンしたときに似たような記事を読んだ覚えがあったのですが、調べてもなかなか出てきませんでした。
そこで、**毎日自動で GitHub の草をチェックして、もし草が生えていなければ Discord に通知してくれるツール**を作りました。

https://github.com/Nyayuta1060/Git-GrassReporter

## なぜ作ったのか

### 問題

- 忙しい日にコミットし忘れることがある
- 草が途切れてしまうとモチベーションが下がる
- 一度スルーすると「まぁ、いいか...」と思ってしまうことが多発する

### 解決したかったこと

- **完全自動化**: PC を起動していなくても動作
- **リアルタイム通知**: 草が生えていない日にすぐに気づける
- **無料**: コストをかけずに運用したい
- **簡単セットアップ**: 複雑な設定なしで使いたい

## 技術構成

### 全体アーキテクチャ

```
GitHub Actions (scheduler)
    ↓
Python Script (grass_checker.py)
    ↓
GitHub GraphQL API (コントリビューション取得)
    ↓
Discord Webhook (通知送信)
```

### 使用技術

- **GitHub Actions**: スケジュール実行とホスティング
- **Python**: メインロジック
- **GitHub GraphQL API**: コントリビューションデータの取得
- **Discord Webhook**: 通知機能

## 実装のポイント

### 1. GitHub GraphQL API でコントリビューションを取得

GitHub の REST API ではなく GraphQL API を使用しました。理由は、コントリビューションデータをより効率的に取得できるからです。

```python
# GraphQLクエリでその日のコントリビューション数を取得
query = '''
query($username: String!, $from: DateTime!, $to: DateTime!) {
  user(login: $username) {
    contributionsCollection(from: $from, to: $to) {
      contributionCalendar {
        totalContributions
      }
    }
  }
}
'''
```

### 2. タイムゾーン処理

日本時間（JST）を基準にして、その日のコントリビューションを正確にチェックします。

```python
# JST基準で今日の日付を取得
jst = timezone(timedelta(hours=9))
today_jst = datetime.now(jst).date()
```

### 3. GitHub Actions での自動実行

毎日 21 時（JST）に自動実行されるよう設定しました。

```yaml
# .github/workflows/check-grass.yml
on:
  schedule:
    - cron: "0 12 * * *" # 12:00 UTC = 21:00 JST
```

### 4. Discord 通知

Webhook URL を使用して Discord に通知を送信します。メンション機能も実装しています。

```python
# Discord通知の送信
message = f"🌱 {today}の草が生えていません！今日もコミットしましょう！"
payload = {
    "content": f"<@{user_id}> {message}",
    "username": "GitHub Grass Checker"
}
```

## 工夫した点

### 1. 完全無料での運用

- GitHub Actions: 月 2,000 分まで無料（このツールは月 30 分程度しか使わない）
- サーバー不要: GitHub Actions が実行環境を提供
- 外部サービス不要: GitHub API と Discord Webhook のみ使用

### 2. セキュリティ

機密情報はすべて GitHub Secrets で管理：

- `GH_TOKEN`: GitHub Personal Access Token
- `DISCORD_WEBHOOK_URL`: Discord Webhook URL
- `DISCORD_USER_ID`: Discord User ID

### 3. エラーハンドリング

- API 呼び出しの失敗に対する適切な例外処理
- ログ出力によるデバッグ支援
- GitHub Actions での実行結果確認

### 4. 設定の簡単さ

環境変数と GitHub Secrets を使用することで、コードを変更せずに設定可能にしました。

## 苦労した点

### 1. GitHub Actions を初めて使用した

今回初めて GitHub Actions を本格的に使用したため、YAML の書き方やワークフローの概念を理解するのに時間がかかりました。

**具体的な困難：**

- **YAML のインデントエラー**: スペースとタブの混在でワークフローが失敗
- **環境変数の渡し方**: secrets から環境変数への設定方法が分からない
- **デバッグ方法**: ローカルでは動くのに Actions 上で失敗する原因の特定が困難

**解決方法：**

- GitHub Actions の公式ドキュメントを熟読
- 他のプロジェクトの workflow ファイルを参考にする
- エラーログを詳しく読み、一つずつ問題を解決

### 2. セキュリティ的な問題

GitHub Token や Discord Webhook URL などの機密情報をどう管理するかで悩みました。

**考慮した点：**

- **Token の権限設定**: 必要最小限の権限（`read:user`のみ）に設定
- **Secrets の管理**: GitHub Secrets を使用してコードに直接書かない

## 実際の動作

### セットアップ

1. リポジトリを Fork またはクローン
2. GitHub Secrets に必要な情報を設定
3. GitHub Actions を有効化

### 動作例

- **草が生えている場合**: 何もしない（静かに監視）
- **草が生えていない場合**: Discord に通知が届く

```
@user 🌱 2025年10月08日の草が生えていません！今日もコミットしましょう！
```

## 成果と効果

### 使ってみた結果

- **継続率向上**: 通知により、コミット忘れが大幅に減少
- **モチベーション維持**: 草が途切れることがなくなった
- **習慣化**: 自然と毎日コードに触れる習慣が身についた

## 今後の改善点

### 機能追加のアイデア

Claude に聞いてみました。

1. **通知方法の選択肢**: Slack、LINE、メールなど
2. **統計機能**: 週間・月間の草生成レポート
3. **目標設定**: 連続日数の目標設定とアチーブメント
4. **チーム機能**: チームメンバーの草をまとめて監視

### 技術的改善

1. **マルチユーザー対応**: 複数の GitHub アカウントを監視
2. **カスタマイズ性向上**: 通知時間や メッセージのカスタマイズ
3. **Web ダッシュボード**: 草の統計を可視化する Web UI

## まとめ

### 学んだこと

- **完全無料での運用**は十分可能
- **GitHub Actions**は想像以上に便利で強力

### 開発者として

このツールを作ることで、以下のスキルが向上しました：

- GitHub Actions の理解
- Python での API 連携
- セキュリティを考慮した設計

---

GitHub リポジトリは[こちら](https://github.com/Nyayuta1060/Git-GrassReporter)です。ぜひ試してみて、フィードバックをいただければ嬉しいです！
