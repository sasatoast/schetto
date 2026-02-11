# Rails実装方針

本プロジェクトでは、Service層パターンを採用し、責務を明確に分離します。

## ディレクトリ構造

```
app/
  controllers/api/v1/
    events_controller.rb
    invitations_controller.rb

  services/
    application_service.rb
    events/
      create_event.rb
      update_event.rb
      list_events.rb
    invitations/
      issue_invitation.rb
      accept_invitation.rb

  models/
    event.rb
    invitation.rb
    member.rb
```

## Model

### 責務
- データベースの状態を保持
- バリデーション・関連を定義
- ビジネスロジックは最小化（MVP）

### 例

```ruby
class Event < ApplicationRecord
  belongs_to :user
  has_many :invitations

  validates :name, presence: true
  validates :start_at, presence: true
end
```

## Service

### 責務
- ビジネスロジックをコントローラー、モデルに置かない
- 単一責任の原則に従う
- 仕様変更に強い設計

参考：[Service層パターン](https://techracho.bpsinc.jp/hachi8833/2022_03_17/46482)

### 命名規則
- `CreateUser`や`AuthenticateUser`のように、コマンドアクションを先に書く命名方法を採用
- 責務が明確になるため、`UserCreator`や`UserAuthenticator`のような「〜or」で終わる命名よりも推奨

### 実装方針
- コンストラクタはフィールド変数の定義だけにとどめる
- `call`メソッドの中は処理の流れだけにとどめる
- 詳細な処理はprivateメソッドに切り出す

### 例

#### ベースクラス

```ruby
class ApplicationService
  def self.call(...)
    new(...).call
  end
end
```

#### service/events/create_event.rb

```ruby
module Events
  class CreateService < ApplicationService
    def initialize(current_user:, params:)
      @current_user = current_user
      @params = params
    end

    def call
      validate_privileges!
      build_event
      save_event
      send_notifications
      event
    end

    private

    attr_reader :current_user, :params, :event

    def validate_privileges!
      raise "権限がありません" unless current_user.parent?
    end

    def build_event
      @event = Event.new(params)
    end

    def save_event
      event.save!
    end

    def send_notifications
      Notifications::EventCreatedNotifier.call(event)
    end
  end
end
```

## Controller

### 責務
- HTTPリクエストの受け取り
- Serviceの実行
- JSONレスポンスを返す

### 実装方針
- ビジネスロジックをControllerに書かない
- Serviceクラスのインスタンスを作って`call`実行するだけ
- DI（依存性の注入）をすることで依存性の逆転を実現

参考：[依存性の注入](https://qiita.com/k2491p/items/686ee5dd72b4baf9a81a)

### 例

```ruby
class Api::V1::EventsController < ApplicationController
  def create
    event = Events::CreateService.call(
      current_user: current_user,
      params: event_params
    )
    render json: event, status: :created
  end

  private

  def event_params
    params.require(:event).permit(:name, :start_at, :end_at)
  end
end
```

## 参考記事

- [Service層パターン](https://qiita.com/3062_in_zamud/items/6d3fa4c5dbdf1625e441)
- [RailsのService層実装](https://qiita.com/roll1226/items/937a15898df57af12de6)
- [nfc-sticker-be実装例](https://github.com/sasatoast/nfc-sticker-be)
- [依存性の注入](https://qiita.com/k2491p/items/686ee5dd72b4baf9a81a)
- [Service層パターンの詳細](https://techracho.bpsinc.jp/hachi8833/2022_03_17/46482)
