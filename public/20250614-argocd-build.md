---
title: ArgoCDをhttpでアクセスする
tags:
  - ArgoCD
  - kubernetes
  - ingress
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# 今回のはまりどころ
- ArgoCDはデフォルトでhttpsでしかアクセスできない
- httpでアクセスしたい場合はArgoCDのhttpsを無効化すればいい

# ArgoCDデプロイ

下記の記事を参考にArgoCDをデプロイする

https://qiita.com/masamokkulu/items/e725113706a719175ae6

```
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

このあと、参考記事の通りにPort Forwardしておけばここまではまらなかったと思いますが、「kubernetesといえばingressでしょう！」と、ingressを設定しようとしたためにひとはまりしました

https://qiita.com/tebichi/items/a314888ddd2596adaeee

# httpでingress経由でArgoCDにアクセスする

暗号化なんて面倒だしやってられないということで、httpでアクセスするingressをデプロイしてアクセスする

```ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.hogehoge.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

なんと！ httpsにリダイレクトされるのか。。。

```
$ kubectl apply -f ingress.yaml
ingress.networking.k8s.io/argocd-ingress configured
$ curl -I http://argocd.hogehoge.com
HTTP/1.1 307 Temporary Redirect
Date: Sat, 14 Jun 2025 05:50:43 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Location: https://argocd.hogehoge.com/
```

# ingress → argocd-serverのアクセスだけhttpsにする

後から考えれば、この思考が失敗の根源  
nginx.ingress.kubernetes.io/backend-protocolのannotationを追加して、argocdへのアクセスをhttpsに変更する

```
[アクセス端末] → (http) → [ingress] → (https) → [argocd-server]
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"  ★HTTPSでバックエンドにアクセスするように追加
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.in.vl-style.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443  ★ポートを80から443に変更
```

ブラウザでアクセスすると、ログイン画面が表示される。感無量！  
だが、ログインすると一瞬画面が変わるのだが、すぐにログイン画面に戻ってしまう

![argocd-build-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3761786/84aface8-3d2c-46b5-bb1b-b8fd6aee0813.png)

DevToolsで確認する。途中に出てくる401(Unauthorized)は何？？  

![argocd-build-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3761786/c252ba98-c813-4705-8de9-f611ca3e157c.png)

「no session information」らしい。なんで？

![argocd-build-3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3761786/f14ccf2d-1bba-4319-8428-e63eaab6334c.png)

原因はこれ！ご丁寧にDevToolsでびっくりマークまでついてるし。。。  
tokenのCookieを取得しているが、Secureがセットされているため、https通信してない場合はブラウザはCookieをサーバに送らない仕様。http通信しているので、このtoken Cookieが送信されておらず、「no session information」になってしまう

![argocd-build-4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3761786/146df3ee-cbf4-435b-ad62-df69da3ed216.png)

鍵ペアを作成し、ingressにtlsの設定を追加してhttps通信ができるようにし、無事にArgoCDのログインに成功できた  
後から知ったことだが、NGINX Ingress Controllerはデフォルトで80と443でLISTENしているそうなので、tlsの設定追加は必須ではなかった

# 後日談

会社のエンジニアと雑談（瞬殺）  
- 私：ArgoCD勉強しようと思って・・・(うんぬんかんぬん)・・・ログインできなくて大変だったんだよー  
- 同僚：そうですよね。HTTPS無効化しないといけないんですよねー  
- 私：（ん？？？）そっかー、全然そっちの考えうかんでなかったよー。。。

# まとめ

今度、これやってみようと思います
ingressの理解が深まったし、良しとしよう

https://qiita.com/tuntunkun/items/f593c56aac356c5f1958