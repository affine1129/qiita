---
title: APIGateway + Lambdaでの同期・非同期API作成(CDK)
tags:
  - Python
  - AWS
  - 非同期処理
  - aws-cdk
  - CDK
private: false
updated_at: '2023-05-17T09:01:10+09:00'
id: ea5e4c010a229c9ab46f
organization_url_name: null
slide: false
---
# はじめに

GETは同期API、それ以外のメソッドは非同期APIとして実装したいなと思い、CDKを用いて同期APIと非同期APIをそれぞれ実装してみました

AWSコンソールでの作成方法の記事はよくあるのですが、CDKを用いた構築方法はあまり無いように思ったため今回記事にしました

# 同期APIの作成

## AWSコンソールからの設定方法

- 以下の記事が分かり易いです
    - [https://qiita.com/miyuki_samitani/items/f01f1bd49334f97fe84c](https://qiita.com/miyuki_samitani/items/f01f1bd49334f97fe84c)

## CDKを用いた方法

```python
sync_lambda = lambda_.Function(
    self, 'sync-lambda',
    runtime=lambda_.Runtime.PYTHON_3_9,
    handler='[handlerを指定]',
    code=lambda_.Code.from_asset('[デプロイ対象のディレクトリを指定]'),
    function_name='[Lambda Function名を指定]',
    timeout=Duration.minutes(15),
    memory_size=1024,
    environment={
        'TZ': 'Asia/Tokyo',
    },
	# 他必要に応じて、Lambdaの設定を追加
)

apilambda_integration = apigw.LambdaIntegration(
    sync_lambda,
)

api_resource = api.root.add_resource('[リソース名を指定]')
api_resource.add_method('GET', apilambda_integration, api_key_required=True)
```

同期処理の場合は、Lambdaプロキシ統合を使用できるため、比較的シンプルな方法で実装可能です

# 非同期APIの作成

## AWSコンソールからの設定方法

- 以下の記事が分かり易いです
    - [https://qiita.com/so_komiya/items/4d7128a3856d581dbbc8](https://qiita.com/so_komiya/items/4d7128a3856d581dbbc8)

## CDKを用いた方法

```python
async_lambda = lambda_.Function(
    self, 'sync-lambda',
    runtime=lambda_.Runtime.PYTHON_3_9,
    handler='[handlerを指定]',
    code=lambda_.Code.from_asset('[デプロイ対象のディレクトリを指定]'),
    function_name='[Lambda Function名を指定]',
    timeout=Duration.minutes(15),
    memory_size=1024,
    environment={
        'TZ': 'Asia/Tokyo',
    },
	# 他必要に応じて、Lambdaの設定を追加
)

request_templates = """
#set($allParams = $input.params())
{
"body" : $input.json('$'),
"resource": "$context.resourcePath",
"httpMethod": "$context.httpMethod",
"pathParameters": {
    #set($path_params = $allParams.get("path"))
    #foreach($paramName in $path_params.keySet())
    "$paramName" : "$util.escapeJavaScript($path_params.get($paramName))"
        #if($foreach.hasNext),#end
    #end
    },
"queryStringParameters": {
    #set($query_params = $allParams.get("querystring"))
    #foreach($paramName in $query_params.keySet())
    "$paramName" : "$util.escapeJavaScript($query_params.get($paramName))"
        #if($foreach.hasNext),#end
    #end
    }
}
"""

async_integration = apigw.LambdaIntegration(
    async_lambda,
    proxy=False,
    request_templates={'application/json': request_templates},
    request_parameters={
        'integration.request.header.X-Amz-Invocation-Type': "'Event'"
    }
)
```

非同期処理の場合は、少し複雑な記載になっているため`apigw.LambdaIntegration()`の引数に関して解説をつけます

- proxy：
    - プロキシ設定をしないためにFalseを指定する必要があります
- request_templates：
    - APIGatewayが受け取ったリクエスト情報をLambdaを渡す際にどのような値にマッピングするかを定義します
    - 今回の定義では、Lambdaプロキシ統合を用いた場合と同様の形式でLambdaが受け取れるようにマッピングルールを設けています
        
        ※Lambdaプロキシ統合ではbodyは文字列となりますが、この書き方では文字列にならずdictで届きます。。
        
- request_parameters：
    - `X-Amz-Invocation-Type`に'Event'を指定することで非同期でLambdaが動作します
    - ちなみに’RequestResponse’を設定すると同期処理になります

# 最後に

やはりCDKはめっちゃ便利。。

CDKを利用するときは、まずAWSコンソールで作ってみて、それぞれの設定がCDKでどのように表現するのかを確認しながら行うとやりやすく感じました！

また、他にもいろいろな機能を作りたいと思うので、CDKの知識もつけていきたいと思います！
