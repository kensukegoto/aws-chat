# SAMと連携

- SAMはCloudFormationのテンプレートを作成するもの
- SAMコマンドでCloudFormationのテンプレート(output.yaml)を作成する。template.yamlを元にSAMコマンドで作成される。その際にLambda関数の実際のARNなどが埋め込まれる
- AWSへのリソースの展開はCloudFormation

# API Gateway + WecSocket

## ルート

クライアントからのリクエストをその種類によって受付場所を変更する。住民票の登録は５番窓口と言うように振り分けるイメージ。

### AWSが用意しているルート

- $connect
    - クライアントがWebSocket APIに最初に接続したとき
- $disconnect
    - クライアントがAPIから切断するとき
- $default
    - 上記とユーザー定義のルートにも一致しないとき

# 統合タイプ（Integration）とは

APIGatewayへのリクエストはLambdaなのかEC2なのか、他のAWSサービスなのか各バックエンドに処理が投げられる。バックエンドへのリクエストを各バックエンドに対して適切な形にフォーマットする。フォーマットのタイプ。

## 種類

APIGatewayでリクエストを受け止めたあと、何処に処理を任せるか

- Lambda関数
    - どのLambda関数を呼ぶかも指定
- HTTP
    - バックエンドのEC2にPOSTを投げるなど。バックエンドのURLも指定
- Mock
- AWSサービス
    - AWSStreamやCloudWatchなど他のサービスを指定
- VPCリンク

# output.yaml解説

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: 'simple-websockets-chat-app'
  simple-websockets-chat-app

Parameters:
// dynamo作成時のオプション。テーブル作成は別。
  TableName:
    Type: String
    Default: simplechat_connections
    Description: (Required) The name of the new DynamoDB to store connection identifiers
      for each connected clients. Minimum 3 characters
    MinLength: 3
    MaxLength: 50
    AllowedPattern: ^[A-Za-z_]+$
    ConstraintDescription: Required. Can be characters and underscore only. No numbers
      or special characters allowed.

Resources:

  SimpleChatWebSocket:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: SimpleChatWebSocket
      ProtocolType: WEBSOCKET
      RouteSelectionExpression: $request.body.message

# 後述のDeploymentの項目で呼ぶための名前
  ConnectRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
      RouteKey: $connect
      AuthorizationType: NONE
      OperationName: ConnectRoute
# TargetはLambda関数を指定するもの
# !Join 文字列の結合。'/'でintegrationsと!Ref ConnectIntegを結合。
# https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-join.html
      Target:
        Fn::Join:
        - /
        - - integrations
          - Ref: ConnectInteg
          # 具体的な内容はConnectIntegに記述されている
  
  ConnectInteg:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
      Description: Connect Integration
      IntegrationType: AWS_PROXY
      IntegrationUri:
# 統合タイプをLambdaで具体的には後述のOnConnectFunctionで指定した関数を呼ぶ
# https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html
        Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${OnConnectFunction.Arn}/invocations
  DisconnectRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
      RouteKey: $disconnect
      AuthorizationType: NONE
      OperationName: DisconnectRoute
      Target:
        Fn::Join:
        - /
        - - integrations
          - Ref: DisconnectInteg
  DisconnectInteg:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
      Description: Disconnect Integration
      IntegrationType: AWS_PROXY
      IntegrationUri:
        Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${OnDisconnectFunction.Arn}/invocations
  SendRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
# $が無いが戸惑わない {"message": "sendmessage","data":"Hello Kensuke" }
# "message"に"sendmessage"があった場合の一連の処理
      RouteKey: sendmessage
      AuthorizationType: NONE
      OperationName: SendRoute
      Target:
        Fn::Join:
        - /
        - - integrations
          - Ref: SendInteg
  SendInteg:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
      Description: Send Integration
      IntegrationType: AWS_PROXY
      IntegrationUri:
        Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${SendMessageFunction.Arn}/invocations
  Deployment:
    Type: AWS::ApiGatewayV2::Deployment
    DependsOn:
    - ConnectRoute
    - SendRoute
    - DisconnectRoute
    Properties:
      ApiId:
        Ref: SimpleChatWebSocket
  Stage:
    Type: AWS::ApiGatewayV2::Stage
    Properties:
      StageName: Prod
      Description: Prod Stage
      DeploymentId:
        Ref: Deployment
      ApiId:
        Ref: SimpleChatWebSocket
  ConnectionsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      AttributeDefinitions:
      - AttributeName: connectionId
        AttributeType: S
      KeySchema:
      - AttributeName: connectionId
        KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 5
        WriteCapacityUnits: 5
# DynamoDBの暗号化
      SSESpecification:
        SSEEnabled: true
      TableName:
# ファイルの先頭で定義したもの
        Ref: TableName

# --------------------------------------------------

  OnConnectFunction:
# $connectで呼ばれるLambda
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: s3://ws-kensukegoto-0825/cc145dced499dacb4b4cbba266ef5b85
      Handler: app.handler
      MemorySize: 256
      Runtime: nodejs10.x
      Environment:
        Variables:
          TABLE_NAME:
            Ref: TableName
# 'simplechat_connections'
# Lambda関数内で使うため
# e.g.) TableName: process.env.TABLE_NAME
      Policies:
      - DynamoDBCrudPolicy:
          TableName:
# 'simplechat_connections'
# 関数に実行権限を与えるため
            Ref: TableName
  OnConnectPermission:
# APIゲートウェイから呼ばれる事を許可？
    Type: AWS::Lambda::Permission
    DependsOn:
# OnConnectPermissionを作る前に下記を先に作る
    - SimpleChatWebSocket
    - OnConnectFunction
    Properties:
      Action: lambda:InvokeFunction
      FunctionName:
        Ref: OnConnectFunction
      Principal: apigateway.amazonaws.com

# --------------------------------------------------

  OnDisconnectFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: s3://ws-kensukegoto-0825/ff49374c142b2eda561170dadbd4ae98
      Handler: app.handler
      MemorySize: 256
      Runtime: nodejs10.x
      Environment:
        Variables:
          TABLE_NAME:
            Ref: TableName
      Policies:
      - DynamoDBCrudPolicy:
          TableName:
            Ref: TableName
  OnDisconnectPermission:
    Type: AWS::Lambda::Permission
    DependsOn:
    - SimpleChatWebSocket
    - OnDisconnectFunction
    Properties:
      Action: lambda:InvokeFunction
      FunctionName:
        Ref: OnDisconnectFunction
      Principal: apigateway.amazonaws.com

# --------------------------------------------------

  SendMessageFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: s3://ws-kensukegoto-0825/5a97206668d6e83e5f8a67b256ebd1c8
      Handler: app.handler
      MemorySize: 256
      Runtime: nodejs10.x
      Environment:
        Variables:
          TABLE_NAME:
            Ref: TableName
      Policies:
      - DynamoDBCrudPolicy:
          TableName:
            Ref: TableName
      - Statement:
        - Effect: Allow
# APIGatewayのWebSocketに接続しているクライアントに対してメッセージを送るなどしたい場合はアプリケーションに新しいアクセス権"execute-api:ManageConnections"が必要。
# これはAWS SAMテンプレートで処理される
          Action:
          - execute-api:ManageConnections
          Resource:
          - Fn::Sub: arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${SimpleChatWebSocket}/*
  SendMessagePermission:
    Type: AWS::Lambda::Permission
    DependsOn:
    - SimpleChatWebSocket
    - SendMessageFunction
    Properties:
      Action: lambda:InvokeFunction
      FunctionName:
        Ref: SendMessageFunction
      Principal: apigateway.amazonaws.com
  
Outputs:
  ConnectionsTableArn:
    Description: Connections table ARN
    Value:
      Fn::GetAtt:
      - ConnectionsTable
      - Arn
  OnConnectFunctionArn:
    Description: OnConnect function ARN
    Value:
      Fn::GetAtt:
      - OnConnectFunction
      - Arn
  OnDisconnectFunctionArn:
    Description: OnDisconnect function ARN
    Value:
      Fn::GetAtt:
      - OnDisconnectFunction
      - Arn
  SendMessageFunctionArn:
    Description: SendMessage function ARN
    Value:
      Fn::GetAtt:
      - SendMessageFunction
      - Arn
  WebSocketURI:
    Description: The WSS Protocol URI to connect to
    Value:
      Fn::Join:
      - ''
      - - wss://
        - Ref: SimpleChatWebSocket
        - .execute-api.
        - Ref: AWS::Region
        - .amazonaws.com/
        - Ref: Stage
```
