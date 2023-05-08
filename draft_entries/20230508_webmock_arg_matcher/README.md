## webmock

RSpec + webmock で多少複雑な stub が必要になったことを切欠に webmock の実装を同僚と読み漁った知見メモ。

例としてテスト対象が 以下の `order` だったとする。

```ruby
def order
  conn = Faraday.new(
    url: "http://localhost:4000",
    headers: {
      "Content-Type" => "application/json"
    }
  )
  res = conn.post('/api/order') do |req|
    req.body = {
      product_ids: ["100", "200"],
      requested_at: Date.current.to_time.to_i,
    }.to_json
  end
  res.body
end
```

これを webmock で stub する場合、body の `requested_at` はマッチングの条件から無視するようにしたい。

Request Body の一部分のみをマッチング条件としたいときは https://github.com/bblimke/webmock#matching-request-body-against-partial-hash にあるとおり、 `hash_including` を使えば良い。

```ruby
stub_request(:post, "http://localhost:4000/api/order").with(
  headers: {
    "Content-Type" => "application/json",
  },
  body: hash_including({
    product_ids: ["100", "200"],
  }),
).to_return(body: "OK")
```

ここまでは webmock のドキュメントをちゃんと読めば書いてあることである。

`product_ids` について、`["100", "200"]` でも `["200", "100"]` でもいいので、順序を問わずマッチングさせるような stub にするようなこともできる。

RSpec の `.with` でいうと以下のようなイメージ。

```ruby
expect(double).to receive(:order).with(array_including(["100", "200"]))
```

https://github.com/rspec/rspec-mocks#argument-matchers

webmock でも RSpec Mocks と同様に `array_including` が利用できる。

```ruby
stub_request(:post, "http://localhost:4000/api/order").with(
  headers: {
    "Content-Type" => "application/json",
  },
  body: hash_including({
    product_ids: array_including(["100", "200"]),
  }),
).to_return(body: "OK")
```

というよりも、 https://github.com/rspec/rspec-mocks#argument-matchers で定義されているような `RSpec::Mocks::ArgumentMatcher` module に含まれるメソッドはどれも利用できる。

仕組みとしては至って単純で、webmock の Matcher 実装が `===` method を見るように実装されていて、 RSpec Mocks の Argument Matcher の各実装も `===` で実装されているために、同じ Matcher が利用できるというだけ。

https://github.com/bblimke/webmock/blob/833291d4ce2d5927a738f929db26b594bf4fa7f5/lib/webmock/request_pattern.rb#L373

したがって、以下のような Custom Argument Matcher をヘルパとして webmock に食わせることも可能。

```ruby
class CustomArgumentMatcher
  def initialize value
    @value = value
  end

  def === args
    @value == args.first
  end
end

def custom value
  CustomArgumentMatcher.new value
end
```

```ruby
.with(
  body: {
    product_ids: custom("100"),
  },
)
```
