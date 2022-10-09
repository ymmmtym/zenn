---
title: "5 GitOps Best Practices（和訳）"
emoji: "🪁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gitops"]
published: false
---


:::message
この記事は、Intuit 社の Alex Collins 氏による [5 GitOps Best Practices](https://blog.argoproj.io/5-gitops-best-practices-d95cb0cbe9ff) の和訳です。
:::

GitOps が登場してしばらく経ちますが、新旧問わずユーザーをつまずかせるものがあります。
Intuit 社で Argo CD を開発し、本番環境で何千ものアプリケーションを運用する中で発見した、重要なベストプラクティスをいくつか紹介します。

## #その1: 2つのレポ。アプリのソースコード用とマニフェスト用の2つのリポジトリ

多くのエンジニアは、アプリのソースコードとマニフェストの両方を1つのリポジトリで管理することから始めます。
しかし、これにはいくつかの問題があります。

- アプリへのすべてのコミットがデプロイにつながる可能性がある。
- 誰もがリリースできるが、それは必要なこととは限らない。
- Git ログに、アプリの変更とデプロイの変更が混在してしまう。

代わりに 2 つのリポジトリを作成し、1 つにアプリのソースコードを、もう 1 つにデプロイメント設定（つまりマニフェスト）を保存することをオススメします。

## #その2：適切な数のデプロイメントコンフィグレーションレポを選択する

デプロイメント設定をいくつのリポジトリに保存するか検討します。
万能な解決策はないので、いくつかの提案をしておきます。

- 自動化があまり進んでいない小規模な会社で、すべての人を信頼している場合は、単一レポを使用します。
- 中堅企業で、ある程度自動化されている場合は、チームごとのレポ。
- 大企業で、自動化が進んでいて、コントロールが必要な場合は、サービスごとのレポ。

自分たちのレポがあれば、チームは自分たちの面倒を見ることができます。中央のチームがリリースのボトルネックになったり、すべてのチームに書き込み権限を与えたりするのではなく、誰がリリースできるかを決めることができるのです。

## #その3：コミットする前にマニフェストをテストする

多くのエンジニアは、マニフェストの変更をコミットしてプッシュし、GitOpsエージェントにその変更を検証させ、アプリをデプロイできるかどうかを確認します。コミットやプッシュの前に変更をテストしていれば防げたはずの問題が、プリプロダクション環境に紛れ込んでいるのを私たちはよく見かけます。

エージェントは通常、CLIコマンド（"helm template "など）を実行してマニフェストを生成するので、変更をコミットする前にローカルでコマンドを実行すれば、マニフェストをテストできます。

## #第4回：外部からの変更でGit Manifestを変更してはいけない

Kustomize と Helm は、同じコミットで異なるマニフェストをテンプレート化できます。

もしマニフェストが Git にコミットすることなく変更される可能性がある場合。

- 変更を管理または監査することができなくなります。
- おそらく、良いバージョンにロールバックすることができないでしょう。

Kustomize リモートベースを使用している場合、それらを特定のコミットに固定します。

```yaml
# Un-pinned:
bases:
- github.com/argoproj/argo-cd/manifests/cluster-install
# Pinned:
bases:
- github.com/argoproj/argo-cd//manifests/cluster-install?ref=v0.11.1
```

Helmの依存関係を使用している場合は、それらもピン留めしてください。

```yaml
# Un-pinned:
dependencies:
- name: argo-cd
  version: *
  repository: github.com/argoproj/argo-cd/manifests/cluster-install
# Pinned:
dependencies:
- name: argo-cd
  version: 0.6.1
  repository: github.com/argoproj/argo-cd/manifests/cluster-install
```

## #その5：秘密の管理方法を計画する

GitOps では、シークレットの管理に余計な手間がかかるのは周知のとおりです。始める前に、どのようにシークレットを管理するのか計画しましょう。ここでは、私たちがこれまでに成功させてきた方法をいくつか紹介します。

- [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Godaddy Kubernetes External Secrets](https://github.com/godaddy/kubernetes-external-secrets)
- [Hashicorp Vault](https://www.vaultproject.io/)
- [Helm Secrets](https://github.com/futuresimple/helm-secrets)
- [Kustomize secret generator plugins](https://github.com/kubernetes-sigs/kustomize/blob/fd7a353df6cece4629b8e8ad56b71e30636f38fc/examples/kvSourceGoPlugin.md#secret-values-from-anywhere)

## まとめ

何かをするのに1つの方法しかないということはありませんが、これらの一般的なベストプラクティスが正しい方向を示してくれることを願っています。
