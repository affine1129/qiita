---
title: 個人的にお気に入りのPythonプロジェクトのファイル構成
tags:
  - Python
  - ポエム
private: false
updated_at: '2023-05-02T08:33:30+09:00'
id: 6d83ca9401da6d7d2ff6
organization_url_name: null
slide: false
---
# 作りたい機能

- PCやスマホから特定のデータベースにデータを蓄積する機能
    
    ※データの例としては収支や勉強時間
    

# 設計上の要望

- AWSを使用してAPIとして使用したい
- コマンドから手軽に使用したい
- Docker上で動作させたい
- 手軽にコードを更新できる仕組みとしたい

# これが現在のファイル構成

設計上の要望を叶えようと作成したプロジェクトのファイル構成は以下の通りです

一部のファイル(折りたたみがあるファイル)に関しては記載例を載せています

- github
    - workflows
        - <details><summary>deploy.yaml：Lambdaへのデプロイを自動化する</summary>
            
            ```yaml
            name: AWS Lambda Deploy 
            on:
              push:
                branches:
                  - [ここに指定したブランチにコードがプッシュされると、このワークフローが発火する]
            jobs:
              deploy:
                runs-on: ubuntu-latest
                permissions:
                  id-token: write
                  contents: read
            
                steps:
                  - name: checkout
                    uses: actions/checkout@v3
            
                  - name: configure-aws-credentials
                    uses: aws-actions/configure-aws-credentials@master
                    with:
                      aws-region: ${{ secrets.AWS_REGION }}
                      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
                      role-session-name: GitHubActions
            
                  - name: get-caller-identity is allowed to run on role.
                    run: aws sts get-caller-identity
            
                  - name: setup-python
                    uses: actions/setup-python@v3
                    with:
                      python-version: '3.x'
            
                  - name: lambda update
                    run: |
                      pip3 install awscli
                      cd [デプロイしたいフォルダを指定] && zip -r package.zip ./*
                      aws lambda update-function-code --function-name [デプロイ先のLambda名を指定] --zip-file fileb://package.zip --publish
            ```

            </details>
            
- docker
    - <details><summary>docker-compose.yaml</summary>
        
        ```yaml
        version: "3"
        services:
          コンテナ名:
            build:
              context: ../
              dockerfile: dockers/dockerfile
            env_file: ../.env # .envの内容がコンテナの環境変数として設定される
            environment:
              - PYTHONPATH=/var/app/src/core:/var/app/src/helper/util/python:/var/app/src/helper/lib/python
              - TZ=Asia/Tokyo
            volumes:
              - ../:/var/app
              - ~/.aws:/root/.aws
            tty: true
            restart: always
        ```
        
        </details>

    - Dockerfile
    - <details><summary>docker-exec.sh：一発でbuildしてexecするシェルスクリプト</summary>
        
        ```bash
        docker compose up -d
        docker exec -it [コンテナ名] bash
        ```

       </details>
        
- src
    - core：ここをデプロイ時にLambdaに入れる
        - model：各リソースごとに作成
            - model1.py
        - service：機能ごとに作成する
            - service1.py
        - <details><summary>command.py：コマンドを実装</summary>
            
            ```python
            # このような構成にすると、コマンド名をリソース名で定義した際に、APIと同じような使い方ができるようになる
            
            @click.group(invoke_without_command=True)
            @click.option('--get', '-g', 'method', flag_value='GET')
            @click.option('--post', '-p', 'method', flag_value='POST')
            @click.option('--update', '-u', 'method', flag_value='PATCH')
            @click.option('--delete', '-d', 'method', flag_value='DELETE')
            @click.pass_context
            def cli(ctx, method):
            	# コマンド名を取得する処理
                resource = ctx.info_name
            
                body = {}
                query_parameter = {}
                path_parameter = {}
            
                if method == const.GET_METHOD:
            				pass
                elif method == const.POST_METHOD:
                    pass
                elif method == const.PATCH_METHOD:
                    pass
                elif method == const.DELETE_METHOD:
                    pass
                    
                event = {
                    'resource': f'/{resource}',
                    'body': json.dumps(body),
                    'queryStringParameters': query_parameter,
                    'pathParameters': path_parameter,
                    'httpMethod': method
                }
            		handler(event=event)
            ```

            </details>
            
        - <details><summary>handler.py：各リクエストの入口</summary>
            
            ```python
            def handler(event, context=None):
                resource = event.get('resource', '').replace('/', '')
            
                if resource == 'time':
            		# ここでインポートしないと循環インポートが起きてしまうことがある
                    from service.time import Time
                    return Time(event).main()
                elif resource == 'price':
                    from service.price import Price
                    return Price(event).main()
            	else:
                    return {
                        "isBase64Encoded": False,
                        "statusCode": 404,
                        "headers": {},
            		}
            ```

            </details>
            
    - helper：ここをデプロイ時にLambdaLayerに入れる
        - util
            - 外部連携
                - client.py
                - util.py
            - error.py：自作エラー
            - <details><summary>internal_api.py：内部の機能を使用する際に必要</summary>
                
                ```python
                # 内部の別機能を使用する際はここのメソッドを使用しhandler経由で別機能を呼び出す
                
                def __create_event(method: str, resource: str = '', body: dict = {}, path_parameter: dict = {}, query_parameter: dict = {}):
                    return {
                        'resource': f'/{resource}',
                        'body': json.dumps(body),
                        'queryStringParameters': query_parameter,
                        'pathParameters': path_parameter,
                        'httpMethod': method
                    }
                
                def get(resource, query_parameter=None, path_parameter=None):
                    event = __create_event(
                        method='GET',
                        resource=resource,
                        query_parameter=query_parameter,
                        path_parameter=path_parameter
                    )
                    return handler(event)
                ```

                </details>
                
            - general_util.py：全機能共通のutil
        - lib
            - pip installしたライブラリ
    - cdk_stack.py：AWSリソース構築に必要
- README.md
- app.py：AWSリソース構築に必要
- cdk.json：AWSリソース構築に必要
- <details><summary>setup.py</summary>
    
    ```python
    # ここにライブラリの情報を記載することが可能だが、以下の書き方にしsetup.cfgにまとめている
    
    from setuptools import setup
    
    setup()
    ```

</details>
    
- <details><summary>setup.cfg：このプロジェクトをpip install 可能にする。requirements.txtの役割もあり</summary>
    
    ```
    [metadata]
    name = [このライブラリの名前]
    version = [X.X]
    
    [options]
    install_requires = 
    		[必要なライブラリを記載する] ※製品版requirements.txtの役割
        
    packages = find:
    package_dir =
        = src
    
    [options.packages.find]
    where = src
    
    [options.extras_require]
    develop =
    		[開発時に必要なライブラリを記載する] ※開発用requirements.txtの役割
    
    [options.entry_points]
    console_scripts =
        time = command:cli
        price = command:cli
    ```
    
</details>

- .python-version
- .gitignore
- <details><summary>.env：環境変数を定義するファイル、git管理対象外</summary>
    
    ```
    [環境変数名]=[値]
    ```

</details>
    
- source.bat

# この構成の好きなところ

## マイクロサービスアーキテクチャのメリットが活かせる

ポイントは以下です

- 全ての機能の入口がhandler.pyとなっているため、入口の制御がしやすい
- 各リソースの入口もそれぞれ実装しているため、コードレベルでリソースの制御が可能

今までは機能同士でやり取りしていることがあり、そこまで入口の統一をせず実装してしまっていることが多かったですが、実際今回のアーキにしてみると機能編集時の負担が少なくなりました

各機能の入り口は[Cerberus](https://docs.python-cerberus.org/en/stable/api.html)を使用し、バリデーションすることでインプットの内容の想定が容易のため、ある機能の変更によって別機能が影響を受けるかというところは判断しやすかったです

:::note
マイクロサービス勉強の際に読んだ本
    ・[https://www.oreilly.co.jp/books/9784814400010/](https://www.oreilly.co.jp/books/9784814400010/)
:::

## コマンドを実装しているため手軽に動作させることが可能

setup.cfg, setup.pyによりコマンドでpythonファイルを実行可能にし、[Click](https://click.palletsprojects.com/en/8.1.x/)を用いてコマンドを充実させることにより、各機能をコマンドで実行可能としています

デバッグ時や各機能を使用したいときにコマンドから実行できるとAPIより動作も早いので、PCで使用するときは基本的にコマンドで動作させてます

# 最後に

ファイル構成・アーキテクチャに関しては、正解がないと思います

上記に記載の構成はあくまでも現状で良いと思っているものなので今後より良いファイル構成を見つけられるよう、より精進しようと思います
