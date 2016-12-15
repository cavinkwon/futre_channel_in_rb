# Concurrent-ruby로 비동기 다루기

[concurrent-ruby](https://github.com/ruby-concurrency/concurrent-ruby)란 걸출한 gem이 있다. Thread-safe 객체, 자료구조, 컬렉션에 Atom, TVar, CountDownLatch, Actor, Future, Promise, Channel 등 병행성에 필요한 많은 요소들이 구현되어 있다. concurreut-ruby는 두개의 gem 으로 구성되어 있다.
- [concurrent-ruby](https://rubygems.org/gems/concurrent-ruby) : 안정 버전.
- [concurrent-ruby-edge](https://rubygems.org/gems/concurrent-ruby-edge) : 기능 확장 목적으로 분리된 gem. 최종적으로는 concurrent-ruby로 편입될 예정.

비동기를 처리하는데 간단하고 범용성이 높은 concurrent-ruby-edge 의 Future와 Channel 두가지를 살펴보자.

## 1. 설치
```ruby
gem install concurrent-ruby-edge
```
사용할 때는 require 'concurrent-edge'를 기입한다.

## 2. [Future](http://ruby-concurrency.github.io/concurrent-ruby/Concurrent/Edge/FutureShortcuts.html) 이해
Future는 원자성을 갖는 비동기 처리 기능을 제공한다.
```ruby
require 'concurrent-edge'

future = Concurrent.future { sleep 1; 1 + 1 } # 비동기로 즉시 동작
# some codes...
future.value #=> 2  # .value : future가 완전히 평가될 때 까지 블록. 여기선 1초
future.completed? #=> true
```
직관적이고 간단하다. edge의 Future는 "통합된 비동기 처리"란 개념으로 여러 개념들을 Future에 통합시키고 있다.

#### 1) 다수의 비동기 처리
[여러 API를 반복적으로 호출](https://www.facebook.com/groups/rubykr/permalink/1385284441484246/)한다면 아래와 같이 처리할 수 있다.
```ruby
require 'concurrent-edge'

api_runner = ->url { sleep(1); puts url; url } # api caller
api_urls = %w(url_1 url_2 url_3 url_4 url_5)

# back-ground processing
jobs = api_urls.map {|url| Concurrent.future { api_runner.call(url) } }
Concurrent.zip(*jobs).value  #=> ["url_1, "url_2", "url_3", "url_4", "url_5"]
```
순차적으로 실행하면 5초가 소요되지만, Future로 1초만에 처리가 완료된다.
```ruby
url_1
url_3
url_4
url_5
url_2
["url_1", "url_2", "url_3", "url_4", "url_5"]
```
위와 같이 코드는 비동기, 비순차적으로 실행되지만 처리결과는 입력 순서대로 통합된다는 것이 포인트다.

#### 2) 체이닝
비동기 처리 완료 이후에 수행될 작업을 then 으로 지정할 수 있다.
```ruby
require 'concurrent-edge'

api_runner = ->url { sleep(1); puts url; url } # api caller
api_urls = %w(url_1 url_2 url_3 url_4 url_5)

jobs = api_urls.map do |url|
  Concurrent.future { api_runner.call(url)  }.
    then {|url| url[-1] }.
    then {|num| "task#{num}"}
end
Concurrent.zip(*jobs).value  #=> ["task1", "task2", "task3", "task4", "task5"]
```

#### 3) 에러 핸들링
비동기 처리 실패시 에러를 다룰 수 있다.
```ruby
require 'concurrent-edge'

api_err = ->url { rand(2) > 0 ? api_runner.call(url) : StandardError.new(url) }
jobs = api_urls.map {|url| Concurrent.future { api_err.call(url) } }
Concurrent.zip(*jobs).value  #=> ["url_1", "url_2", #<StandardError: url_3>, "url_4", "url_5"]
```
에러는 순차적으로 then, rescue로 전파된다. 아래 NoMethodError는 전파로 인한 결과다.
```ruby
require 'concurrent-edge'

api_err = ->url { rand(2) > 0 ? api_runner.call(url) : StandardError.new(url) }
jobs = api_urls.map do |url|
  Concurrent.future { api_err.call(url) }.
    then {|url| url[-1] }.
    rescue {|e| e.class }
end
Concurrent.zip(*jobs).value  #=> ["1", "2", "3", NoMethodError, "5"]
```
순차적인 전파로 결과가 왜곡되지 않길 원하면 Either(성공, 실패를 감싸는 객체)로 처리결과를 전파하면 된다.  
관련해서는 다음을 참고하자. - [Refactoring with Monads](http://codon.com/refactoring-ruby-with-monads#monads), [RailWay oriented programming](http://fsharpforfunandprofit.com/posts/recipe-part2/)

#### 4) 기타
여러 요소가 future로 통합되어 스케쥴링, 콜백, 쓰레드 풀, 액터사용, 딜레이 지정, 주기적인 작업을 수월히 구현할 수 있다. [참고](http://ruby-concurrency.github.io/concurrent-ruby/Concurrent/Edge/FutureShortcuts.html).
메인 쓰레드가 종료되면 자식 쓰레드로 동작하는 Future작업 또한 함께 종료된다는 점을 기억하자.


## 3. [Channel과 Goroutines](http://ruby-concurrency.github.io/concurrent-ruby/Concurrent/Edge/Channel.html) 이해
Go 언어의 [goroutine](https://golang.org/doc/effective_go.html#goroutines), [channel](https://golang.org/doc/effective_go.html#channels)과 Clojure언어의 [core.async](https://clojure.github.io/core.async/index.html)에 영감을 받아 CSP(순차 프로세스 통신)기능을 구현하였다.
> "Do not communicate by sharing memory; instead, share memory by communicating."  
> "메모리를 공유하여 통신하기 보다 통신을 통해 메모리를 공유합시다."

쓰레드에 안전한 큐(채널)를 두고 여러 쓰레드들(고루틴)이 자유롭게 데이터를 넣고 빼내는 방법으로 통신을 구현한다.
메모리나 파일 시스템을 통한 공유 방식의 통신도 좋지만, 메세지를 주고 받는 채널에 초점을 두면 직관적으로 통신을 다룰 수 있다.

#### 1) Goroutines
실행중인 함수와 별개로 런타임에 관리되는 쓰레드풀 위에서 동작하는 코드 블록을 말한다.
```ruby
require 'concurrent-edge'

puts "Main thread: #{Thread.current}"
# goroutine은 메인 쓰레드가 아닌 별개의 쓰레드로 동작한다.
Concurrent::Channel.go { puts "Goroutine thread: #{Thread.current}" }
```

#### 2) Channel
값을 넣고, 빼낼 수 있는 쓰레드에 안전한 파이프. 고루틴들간 통신시 채널을 매체로 값을 전송하거나 획득한다.

- 기본 (unbuffered channel)
```ruby
require 'concurrent-edge'

messages = Concurrent::Channel.new # 인자가 없을때는 버퍼크기 0인 채널이 생성된다.
Concurrent::Channel.go { messages.put 'ping' } # 메인쓰레드와 별개로 동작.
msg = messages.take
puts msg
```
고루틴 쓰레드가 채널에 'ping'을 넣으면, 메인 쓰레드는 take로 채널로 부터 값을 획득한다. 버퍼가 없는 채널에 메세지를 보낸다는 것은 수신자에게 메세지를 전달하는 것을 의미한다. 따라서, 수신자가 나타나기 전까지 고루틴은 동작을 마치지 않고 블록상태에 놓인다.

#### 3) 고루틴과 채널의 다양한 사용
- 채널 버퍼링
```ruby
require 'concurrent-edge'

ch = Concurrent::Channel.new(capacity: 2)
ch << 1
ch << 2

puts ~ch, ~ch
```
채널의 버퍼크기가 2로 메인 쓰레드는 블록되지 않고 순차적으로 동작한다.

- 여러 고루틴들의 1개 채널 사용
```ruby
require 'concurrent-edge'

Channel = Concurrent::Channel
channel = Channel.new

positves, negatives = [1, 2, 3], [-1, -2, -3]
sum = ->(nums, chan) { chan.put(nums.reduce(:+)) }

Channel.go { sleep 0.1; sum.call(positves, channel) }
Channel.go { sleep 0.1; sum.call(negatives, channel) }

x, y = channel.take, channel.take
puts [x, y, x+y].join(' ') #=> "6 -6 0" 또는 "-6 6 0"
```
두 개의 고루틴은 비동기로 동작하므로 채널에 6, -6 어떤 값이 먼저들어올 지 확정할 수 없다.

- 채널 동기화
```ruby
require 'concurrent-edge'

def worker(done_channel)
  print "working...\n"
  sleep(1)
  print "done\n"

  done_channel << true # "#<<" is an alias for "put"
end

done = Concurrent::Channel.new(capacity: 1)
Concurrent::Channel.go{ worker(done) }

~done # "#<<" is an alias for "take"
```
메인 쓰레드는 채널에 값이 들어올 때 까지 블록된다. 고루틴과 메인 쓰레드는 동기화 상태라 볼 수 있다.

- 멀티 채널 처리
```ruby
require 'concurrent-edge'

Channel = Concurrent::Channel
c1, c2 = Channel.new, Channel.new

Channel.go { sleep(1); c1 << 'one' }
Channel.go { sleep(2); c1 << 'two' }

2.times do
  Channel.select do |s|
    s.take(c1) {|msg| puts "received #{msg}" }
    s.take(c2) {|msg| puts "received #{msg}" }
  end
end
```
여러 채널에 들어오는 요청은 Channel#select로 위와 같이 처리할 수 있다.

- 멀티 채널 응용 (피보나치 수열)
```ruby
require 'concurrent-edge'

Channel = Concurrent::Channel
def fibonacci(c, quit)
  x, y = 0, 1
  loop do
    Channel.select do |s|
      s.case(c, :<<, x) { x, y = y, x+y; x } # alias for `s.put`
      s.case(quit, :~) do                    # alias for `s.take`
        puts 'quit'
        return
      end
    end
  end
end

c, quit = Channel.new, Channel.new

# 채널로 부터 10번 피보나치 수를 획득해서 출력한 뒤, quit 채널에 종료신호를 보낸다.
# 고루틴은 채널에 값이 없으므로 우선은 블록된 상태에 놓인다.
Channel.go do
  10.times { puts ~c }
  quit << 0
end

# fibonacci가 동작하면 c채널에 값이 들어가면서 고루틴이 동작한다.
fibonacci(c, quit)
```
고루틴과 메인 쓰레드의 fibonacci loop간 동기화가 이루어져 피보나치 수가 계산된다.

- 채널 닫기와 반복하기
```ruby
def fibonacci(n, c)
  x, y = 0, 1
  (1..n).each do
    c << x
    x, y = y, x+y
  end
  c.close
end

chan = Concurrent::Channel.new(capacity: 10)
Concurrent::Channel.go { fibonacci(chan.capacity, c) }
chan.each { |i| puts i }
```
채널은 close로 닫을 수 있고, each로 반복적으로 연산을 수행할 수도 있다.

- 그 밖에도 Timers, Tickers 를 제공한다. 상세한 내용은 [다음](http://ruby-concurrency.github.io/concurrent-ruby/Concurrent/Channel.html)을 참고하길 바란다.


#### 3) 고루틴/채널 응용
응용해서 [개미수열](https://ko.wikipedia.org/wiki/%EC%9D%BD%EA%B3%A0_%EB%A7%90%ED%95%98%EA%B8%B0_%EC%88%98%EC%97%B4)을 작성해 보자.

- 개미수열은 아래와 같다. 100번째 개미수열의 10000번째 수를 구해보자.
```ruby
      1
     11
     21
    1211
   111221
   312211
  13112221
 1113213211
      ?
```

- 리스트 조작으로 풀어보자.
```ruby
next_ants = proc do |str|
  str.chars.chunk{|_|_}.
    map {|ch,grp| [grp.size.to_s, ch]}.join
end
series = ->n { (1..n-1).reduce("1", &next_ants) }

# 1번째 ~ 6번째 개미수열을 계산해 보자.
(1..6).map(&series) #=> %w(1 11 21 1211 111221 312211)

# 100번째 수열은 결과가 나오지 않는다.
series.call(100)
```
100번째 개미수열은 대략 길이가 천억 단위인 문자열이 된다. 순차적으로 100번째 문자열을 계산한다면 너무도 오랜 시간이 걸릴 것이다. 고루틴과 채널을 이용하면 쉽고 빠르게 풀 수 있다.

- 고루틴과 채널 사용
```ruby
# 1. 개미수열을 문자열 덩이가 아닌 문자 1개씩 읽어서 처리하자
# 2. 1~100번째 까지 동시에 수열을 만들어 내자(고루틴 99개, 채널 99개)
# 3. n번째 개미수열은 작업 중인 n-1번째 개미수열을 읽어서 채널에 결과를 넣도록 하자(파이프. 채널 동기화)
# 4. 마지막 채널에서 10000번째 문자열을 출력하자.

require 'concurrent-edge'

Ch = Concurrent::Channel
ants = ->r,w { r ? r.chunk{|e|e}.each {|ch,chunk| w << chunk.size.to_s; w << ch} : w << "1" }
pipe = proc {|r| Ch.new(size: 5).tap {|w| Ch.go { ants[r,w]; w.close }} }
ant_series = ->nth { (1..nth).reduce(nil, &pipe) }

# 6번째 개미수열 : "312211"
puts ant_series[6].map{|e|e}.join

# 100번째 개미수열의 10000번째 문자열. 1.5초 정도 걸린다.
ant_series[100].each_with_index {|ch,idx| (puts ch; break) if idx == 9999 }
```
다양한 [예제](https://github.com/ruby-concurrency/concurrent-ruby/tree/master/examples)들이 있으니 참고하자. [Go By Example : Channels](https://gobyexample.com/channels)를 루비로 옮긴 [예제 코드](https://github.com/ruby-concurrency/concurrent-ruby/tree/master/examples/go-by-example-channels)들도 있다.
## 4. 결론
- Future와 Goroutine, Channel을 살펴봤다.
- 비동기 작업을 Future로 간단히 구현할 수 있었다.
- Goroutine, Channel을 이용한 통신을 살펴봤다.
- Concurrent-ruby gem은 동시성을 다루는 여러 요소들이 구현되어 있고 사용하기도 쉽다. 

#### 첨언
- 개미수열 문제는 lazy로 효율적으로 풀 수 있는 문제다. lazy를 사용해서도 풀어보자.
