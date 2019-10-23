# k8s-manifest

EKS 上に Istio をベースとした環境を構築するマニフェストを作成しようと思って作ったプロジェクト。  
基本構成は以下。

- Istio
- Prometheus / Prometheus Operator
- Grafana
- Jeagar
- Kiali

## ローカル環境構築

Homebrew で構築できるものは以下。

```zsh
% brew install awscli
% brew install terraform
% brew install kubernetes-cli
% brew install kubernetes-helm
% brew install helmfile
% helm plugin install https://github.com/databus23/helm-diff --version master
```

Istio のコマンドライン資材の設定は以下。

```zsh
% cd
% curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.3.1 sh -
% cd istio-1.3.1
% export PATH=$PWD/bin:$PATH
% istioctl verify-install
```

## Helm 環境構築

以下の手順で Tiller を導入する。

```zsh
% kubectl apply -f ~/istio-1.3.1/install/kubernetes/helm/helm-service-account.yaml # tiller の Service Account 作成
% helm init --history-max 200 --service-account tiller                             # tiller の作成
% kubectl get all --all-namespaces -o wide | grep tiller | wc -l                   # 確認
       4
```

## Istio 環境構築

### istio-init

```zsh
% helm install ~/istio-1.3.1/install/kubernetes/helm/istio-init --name istio-init --namespace istio-system
% kubectl get crds | grep 'istio.io' | wc -l # 確認１
      23
% helm ls --all                              # 確認２
NAME      	REVISION	UPDATED                 	STATUS  	CHART           	APP VERSION	NAMESPACE
istio-init	1       	Thu Oct  3 21:02:53 2019	DEPLOYED	istio-init-1.3.1	1.3.1      	istio-system
```

### Istio デモ版の構築

```zsh
% helm install ~/istio-1.3.1/install/kubernetes/helm/istio --name istio --namespace istio-system \
    --values ~/istio-1.3.1/install/kubernetes/helm/istio/values-istio-demo.yaml # Istio 構築
% helm install ~/istio-1.3.1/install/kubernetes/helm/istio --name istio --namespace istio-system # Istio 構築
% kubectl label namespace default istio-injection=enabled                       # サイドカーを自動でつける設定
% kubectl get namespace -L istio-injection                                      # 確認
% kubectl apply -f <your-application>.yaml                                      # アプリのデプロイ
```

上記では default namespace に `istio-injection=enabled` ラベルを付与して自動でサイドカー（ envoy ）インジェクションするように設定している。  
なお、自動でサイドカーインジェクションされるための条件は [ここ](https://istio.io/docs/ops/setup/injection-concepts/) 。

Prometheus への接続確認は以下。

```zsh
% kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090
% curl http://localhost:9090
```

Grafana への接続確認は以下。

```zsh
% kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
% curl http://localhost:3000/dashboard/db/istio-mesh-dashboard
```

Jaeger への接続確認は以下。

```zsh
% kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686
% curl http://localhost:16686
```

Kiali への接続確認は以下。

まずはユーザ/パスワードを作成して secret を適用する。  
（ `createDemoSecret: true` に設定している場合、コンソールログインの ID/Pass は admin/admin ）

```zsh
% KIALI_USERNAME=$(read -p 'Kiali Username: ' uval && echo -n $uval | base64)
Kiali Username: # ユーザ名を入力する
% KIALI_PASSPHRASE=$(read -p 'Kiali Passphrase: ' pval && echo -n $pval | base64)
Kiali Passphrase: # パスワードを入力する
% cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: istio-system
  labels:
    app: kiali
type: Opaque
data:
  username: $KIALI_USERNAME
  passphrase: $KIALI_PASSPHRASE
EOF
```

以下でアクセスして、上記で作成したユーザ/パスワードを利用する。

```zsh
% kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001
% curl http://localhost:20001/kiali/console
```

### `istio.io/istio` チャート

helmfile および `istio.io/istio` チャートを利用して構築する。  
なお、 `istio.io/istio` チャートを利用すると Prometheus 、 Grafana 、 Jeager 、 Kiali などもセットで構築できるが、ここでは Jeager 、 Kiali のみ追加で構築している。

```zsh
% helmfile -f helmfile.yaml apply
% helm ls --all
NAME      	REVISION	UPDATED                 	STATUS  	CHART           	APP VERSION	NAMESPACE
istio     	1       	Tue Oct  8 13:48:36 2019	DEPLOYED	istio-1.3.1     	1.3.1      	istio-system
istio-init	1       	Thu Oct  3 21:02:53 2019	DEPLOYED	istio-init-1.3.1	1.3.1      	istio-system
```

Prometheus 、 Grafana は後述の `stable/prometheus-operator` チャートを利用して構築する。

## Prometheus 環境構築

`stable/prometheus-operator` チャートを利用して構築する。  
ここではこのチャートを利用して Grafana 環境も同時に構築する。

### `stable/prometheus-operator` チャート

`helmfile.yaml` に設定済みであれば先の `istio.io/istio` チャート 利用構築時の `helmfile -f helmfile.yaml apply` コマンドにて、 Prometheus Operator の構築も完了している。  
Prometheus の接続は以下。

```zsh
% kubectl -n monitoring port-forward $(kubectl -n monitoring get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090
% curl http://localhost:9090
```

Grafana の接続確認は以下。

```zsh
% kubectl -n monitoring port-forward $(kubectl -n monitoring get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
% curl http://localhost:3000/dashboard/db/istio-mesh-dashboard
```

なお、 ID / Pass は admin / prom-operator 。

## 参考

- Istio 公式
  - [helm 利用](https://istio.io/docs/setup/install/helm/)
  - [Installation Options](https://istio.io/docs/reference/config/installation-options/)
- 参考
  - [マイクロサービスアーキテクチャ向けにサービスメッシュを提供する「Istio」の概要と環境構築、トラフィックルーティング設定](https://knowledge.sakura.ad.jp/20489/)
  - [Istio入門 その1 -Istioとは?-](https://qiita.com/Ladicle/items/979d59ef0303425752c8)
  - [Istio導入のメリットとハマりどころを、実例に学ぶ〜マイクロサービス化の先にある課題を解決する](https://employment.en-japan.com/engineerhub/entry/2019/05/21/103000)
  - [Istio IngressGateway周辺を理解する](https://qiita.com/J_Shell/items/296cd00569b0c7692be7)
  - [IstioをHelmでインストールしてRoutingとTelemetryを行いJaeger/Kialiで確認する](https://www.sambaiz.net/article/185/)
- Prometheus / Grafana
  - [サンプルで学ぶ！PromQLで自在にグラフを描こう (Prometheus + Grafana)](https://qiita.com/nekonok/items/4390a2db8be34da9d238)