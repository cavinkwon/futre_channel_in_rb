*참고, 루비 초보자가 읽었을 때 15 ~ 20 분 걸립니다*

# Concurrent-ruby로 비동기 다루기

[concurrent-ruby](https://github.com/ruby-concurrency/concurrent-ruby)는 동시성 구현에 필요한 요소들이 구비되어 있다. concurreut-ruby는 두 개의 gem 으로 구성되어있다
- [concurrent-ruby](https://rubygems.org/gems/concurrent-ruby) : 안정 버전
- [concurrent-ruby-edge](https://rubygems.org/gems/concurrent-ruby-edge) : 기능 확장 목적으로 분리된 gem. 안정된 기능은 concurrent-ruby로 편입

비동기 구현에 간편하고 범용성 높은 Promise, Channel을 알아보도록 하자

## 1. 설치
Gem 직접 설치
```ruby
# 2019.04.09 기준
gem install concurrent-ruby -v 1.1.5
gem install concurrent-ruby-edge -v 0.5.0
```
또는, 의존 라이브러리로 설치. Gemfile 작성 후 `bundle install`
```ruby
source 'https://rubygems.org'

ruby '2.6.0'

gem 'concurrent-ruby', '~> 1.1.5'
gem 'concurrent-ruby-edge', '~> 0.5.0'
```
IRB 에서 설치확인
```ruby
2.6.0 :001 > require 'concurrent'
 => true
2.6.0 :002 > require 'concurrent-edge'
 => true
```

## 2. 사용
루비 소스 상단에서 require문 작성
```ruby
require 'concurrent'
require 'concurrent-edge'
```

## 3. Promises
아래 코드를 `understanding.rb`로 저장한 뒤 `ruby understanding.rb`로 실행해보자
```ruby
require 'concurrent'

future = Concurrent::Promises.future { sleep(3); 2 } # 비동기로 즉시 동작

# running
p future 
p future.state #=> :pending
puts future.value #=> 2 .value는 코드가 완전히 평가될 때 까지 블록. 여기선 3초

# completed
p future
p future.state #=> :fulfilled
```
콘솔 출력결과
```ruby
#<Concurrent::Promises::Future:0x00007fffd4dfdd98 pending>
:pending
2
#<Concurrent::Promises::Future:0x00007fffd4dfdd98 fulfilled with 2>
:fulfilled
```
비동기 코드 작성과 실행이 직관적이고 간단하다. Promise 활용방법을 자세히 알아보자

##### 1) 여러 작업의 비동기 처리
[API 반복 호출](https://www.facebook.com/groups/rubykr/permalink/1385284441484246/)은 다음과 같이 처리할 수 있다
```ruby
require 'concurrent'

api_runner = ->(url) { sleep(1); puts url; url }
urls = %w[url_1 url_2 url_3 url_4 url_5]

# back-ground processing
jobs = urls.map { |url| Concurrent::Promises.future { api_runner.call(url) } }
p Concurrent::Promises.zip(*jobs).value
#=> ["url_1, "url_2", "url_3", "url_4", "url_5"]
```
순차적으로 다섯번 실행하면 5초가 소요되지만 동시실행으로 1초 만에 처리가 완료된다
```ruby
url_3
url_4
url_1
url_2
url_5
["url_1", "url_2", "url_3", "url_4", "url_5"]
```
위와 같이 코드는 비동기, 비순차적으로 실행되지만 처리결과는 입력 순서대로 통합된다

##### 2) 체이닝
비동기 처리 완료 이후 수행할 작업을 then 으로 지정할 수 있다
```ruby
require 'concurrent'

api_runner = ->(url) { sleep(1); url }
urls = %w[url_1 url_2 url_3 url_4 url_5]

jobs = urls.map do |url|
  Concurrent::Promises.future { api_runner.call(url)  }.
    then { |url_str| url_str[-1] }.
    then { |num| "task#{num}"}
end
p Concurrent::Promises.zip(*jobs).value
#=> ["task1", "task2", "task3", "task4", "task5"]
```

모듈명이 불편하면 extend 로 모듈 메소드들을 현재 scope에서 사용 가능하게 할 수 있다
```ruby
require 'concurrent'

extend Concurrent::Promises::FactoryMethods

api_runner = ->(url) { sleep(1); url }
urls = %w[url_1 url_2 url_3 url_4 url_5]

jobs = urls.map do |url|
  future { api_runner.call(url)  }.
    then { |url_str| url_str[-1] }.
    then { |num| "task#{num}"}
end
zip(*jobs).value
```

##### 3) 에러 핸들링
비동기 처리 실패시 에러를 다룰 수 있다
```ruby
require 'concurrent'

api_runner = ->(url) { sleep(1); url }
api_err = ->(url) { rand(2) > 0 ? api_runner.call(url) : StandardError.new(url) }

urls = %w[url_1 url_2 url_3 url_4 url_5]
jobs = urls.map { |url| Concurrent::Promises.future { api_err.call(url) } }
p Concurrent::Promises.zip(*jobs).value
#=> ["url_1", #<StandardError: url_2>, "url_3", "url_4", #<StandardError: url_5>]
```
에러는 순차적으로 then, rescue로 전파된다. 아래 NoMethodError는 전파로 인한 결과다
```ruby
require 'concurrent'

api_runner = ->(url) { sleep(1); url }
api_err = ->(url) { rand(2) > 0 ? api_runner.call(url) : StandardError.new(url) }

urls = %w[url_1 url_2 url_3 url_4 url_5]
jobs = urls.map do |url|
  Concurrent::Promises.future { api_err.call(url) }.
    then { |url_str| url_str[-1] }.
    rescue { |err| err.class }
end
Concurrent::Promises.zip(*jobs).value
#=> ["1", NoMethodError, "3", "4", NoMethodError]
```
성공, 실패, 에러를 일관되게 처리하고자 한다면 Either로 처리하는 걸 추천한다
* [dry-monads](https://dry-rb.org/gems/dry-monads/) : Maybe, Either 등을 제공하는 Gem
* [Refactoring with Monads](http://codon.com/refactoring-ruby-with-monads#monads) : 모나드 구현, 리팩토링
* [RailWay oriented programming](https://fsharpforfunandprofit.com/rop/) : FP의 에러핸들링. F#으로 작성되어 있지만 한번씩 읽어보는걸 추천한다


##### 4) 기타
메인 쓰레드가 종료되면 자식 쓰레드로 동작하는 비동기 작업 또한 함께 종료된다는 점을 기억하자
스케쥴링, 콜백, 쓰레드 풀, 액터, 딜레이 지정, 주기적인 작업 등을 수월히 구현할 수 있다 
* [API & Doc](http://ruby-concurrency.github.io/concurrent-ruby/master/Concurrent/Promises.html)
* [Use-cases](https://github.com/ruby-concurrency/concurrent-ruby/blob/master/docs-source/promises.in.md#use-cases)


## 4. Goroutines and Channel
Go 언어의 [goroutine](https://golang.org/doc/effective_go.html#goroutines), [channel](https://golang.org/doc/effective_go.html#channels)과 Clojure언어의 [core.async](https://clojure.github.io/core.async/index.html)에 영감을 받아 CSP(순차 프로세스 통신)기능을 구현하였다
> "Do not communicate by sharing memory; instead, share memory by communicating."  
> "메모리 공유로 소통하기 보다 소통을 통해 메모리를 공유합시다"

쓰레드에 안전한 큐(채널)를 두고 여러 쓰레드들(고루틴)이 자유롭게 큐에 데이터를 넣고 빼내는 방법으로 통신을 구현한다. 
메모리나 파일 시스템 공유 방식의 통신도 좋지만 메세지를 주고 받는 채널에 초점을 두면 직관적으로 통신을 다룰 수 있다

##### 1) Goroutines
실행중인 함수와 별개로 런타임에 관리되는 쓰레드풀 위에서 동작하는 코드 블록을 말한다
```ruby
require 'concurrent-edge'

puts "Main thread: #{Thread.current}"
#=> Main thread: #<Thread:0x00007ffff00c32d8 run>

# goroutine은 별개의 쓰레드풀에서 동작한다.
Concurrent::Channel.go { puts "Goroutine thread: #{Thread.current}" }
#=> Goroutine thread: #<Thread:0x00007ffff079a070@ ...concurrent-ruby-1.1.5/lib/
#                     /concurrent/executor/ruby_thread_pool_executor.rb:317 run>
```

##### 2) Channel
값을 넣고, 빼낼 수 있는 쓰레드에 안전한 파이프. 고루틴간 통신은 채널을 매체로 값을 전송하거나 획득하는 방식으로 이루어진다

* 기본 (unbuffered channel)
  ```ruby
  require 'concurrent-edge'

  msg_channel = Concurrent::Channel.new # 인자가 없으면 버퍼 길이 0인 채널 생성
  Concurrent::Channel.go { msg_channel.put 'ping' } # 메인 쓰레드와 별개로 동작
  msg = msg_channel.take
  puts msg
  ```
  고루틴 쓰레드는 채널에 'ping'을 넣고 메인 쓰레드는 take로 채널로 부터 값을 획득한다. 
  버퍼가 없는 채널에 메세지를 보낸다는 것은 수신자에게 메세지를 전달하는 것을 의미한다. 
  따라서, 수신자가 나타나기 전까지 고루틴은 동작을 마치지 않고 블록상태에 놓인다

  ```ruby
  require 'concurrent-edge'

  Channel = Concurrent::Channel
  channel = Concurrent::Channel.new

  positves, negatives = [1, 2, 3], [-1, -2, -3]
  sum = ->(nums, chan) { chan.put(nums.reduce(:+)) }

  Channel.go { sleep 0.1; sum.call(positves, channel) }
  Channel.go { sleep 0.1; sum.call(negatives, channel) }

  x, y = channel.take, channel.take
  puts [x, y].join(' ') #=> "6 -6" 또는 "-6 6"
  ```
  두 개의 고루틴은 비동기로 동작하므로 채널에 6, -6 어떤 값이 먼저 들어올 지 확정할 수 없다
  

* 버퍼 채널 (buffered channel)
  ```ruby
  require 'concurrent-edge'

  ch = Concurrent::Channel.new(capacity: 2)
  ch << 1 # `<<` is an alias for `put` or `send`
  ch << 2

  puts ~ch, ~ch # ~` is an alias for `take` or `receive`
  ```  
  채널의 버퍼가 2로 메인 쓰레드는 블록되지 않고 순차적으로 동작한다
  

* 채널 동기화
  ```ruby
  require 'concurrent-edge'

  def worker(done_channel)
    print "working...\n"
    sleep(1)
    print "done\n"

    done_channel << true
  end

  done = Concurrent::Channel.new(capacity: 1)
  Concurrent::Channel.go { worker(done) }

  ~done
  ```
  메인 쓰레드는 done 채널에 값이 들어올 때 까지 블록된다. 
  고루틴과 메인 쓰레드는 동기화 상태라 볼 수 있다

* 멀티 채널 처리
  ```ruby
  require 'concurrent-edge'

  Channel = Concurrent::Channel
  c1, c2 = Channel.new, Channel.new

  Channel.go { sleep(1); c1 << 'one' }
  Channel.go { sleep(2); c1 << 'two' }

  2.times do
    Channel.select do |s|
      s.take(c1) { |msg| puts "received #{msg}" }
      s.take(c2) { |msg| puts "received #{msg}" }
    end
  end
  ```
  여러 채널에 들어오는 요청을 선택적으로 처리할 때 Channel#select를 사용한다. 
  select 블록은 내부 작업 중 하나가 처리될 때 까지 블록된다
  
* 멀티 채널 응용 (피보나치 수열)
  ```ruby
  require 'concurrent-edge'

  Channel = Concurrent::Channel
  def fibonacci(c, quit)
    x, y = 0, 1
    loop do
      # c, quit 채널에 값이 들어올 때 까지 블록
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

  # 채널에서 10번 값을 획득해서 출력한 뒤, quit 채널에 종료 신호
  Channel.go do
    # 고루틴은 채널에 전송된 값이 없으므로 일단은 블록 상태
    10.times { puts ~c }
    quit << 0
  end

  # fibonacci가 동작하면 c채널에 값이 들어가면서 고루틴이 동작한다
  fibonacci(c, quit)
  ```
  고루틴과 메인 쓰레드의 fibonacci 메소드간 동기화가 이루어져 피보나치 수가 계산된다
  
* 채널 닫기와 반복하기
  ```ruby
  require 'concurrent-edge'
  
  def fibonacci(channel)
    x, y = 0, 1
    channel.capacity.times do
      channel << x
      x, y = y, x+y
    end
    channel.close
  end

  chan = Concurrent::Channel.new(capacity: 10)
  Concurrent::Channel.go { fibonacci(chan) }
  chan.each { |fib_num| puts fib_num }
  ```
  채널은 close로 닫을 수 있고 each로 버퍼의 값에 대해 연산을 수행할 수도 있다. 
  채널을 닫아도 버퍼에 담긴 값은 사라지지 않는다 

* Timer
  ```ruby
  require 'concurrent-edge'

  # 1. 만료에 의한 트리거
  before = Time.now  
  timer1 = Concurrent::Channel.timer(2)
  ~timer1
  after = Time.now
  puts "Timer 1 expired : #{after - before}" 
  #=> Timer 1 expired : 2.0238324

  # 2. 채널 close 강제 트리거
  timer2 = Concurrent::Channel.timer(10)
  Concurrent::Channel.go do
    before = Time.now  
    ~timer2
    after = Time.now
    puts "Timer 2 expired : #{after - before}"
  end

  stopped = timer2.close # 10초 타이머, 채널 close로 즉시 trigger 발생
  puts "Timer 2 stopped" if stopped
  #=> Timer 2 expired : 0.0112491
  #=> Timer 2 stopped
  ```
  채널로 부터 값을 획득하기 위해 블록중인 쓰레드는 일정 시간후에 채널이 닫히는 것을 신호로 동작할 수 있다. 
  일정 시간 뒤 코드를 동작시키기 위해 사용한다. 
  타이머가 지정된 채널을 강제로 닫는 경우 블록중인 코드에 tigger를 발생시킨 것과 동일한 효과를 갖는다. 
  특정 시간에 코드를 실행시키는 것은 [ScheduledTask](http://ruby-concurrency.github.io/concurrent-ruby/master/Concurrent/ScheduledTask.html) 를 참고하자
  
* Ticker
  ```ruby
  require 'concurrent-edge'
  
  ticker = Concurrent::Channel.ticker(0.5)
  Concurrent::Channel.go do
    ticker.each { |tick| puts "Tick at #{tick}" }
  end

  sleep(1.6)
  ticker.stop # alias for `close`
  print "Ticker stopped\n"
  #=> Tick at 2019-04-09 04:06:18.863728 +0000 UTC
  #=> Tick at 2019-04-09 04:06:19.363718 +0000 UTC
  #=> Tick at 2019-04-09 04:06:19.863719 +0000 UTC
  #=> Ticker stopped
  ```
  일정주기로 채널에 값을 보내며 capacity가 1인 채널을 생성한다. 
  `ticker.each { do_something }`로 주기적으로 작업을 실행하고 채널을 닫아 주기적인 작업을 끝낼 수 있다. 

##### 3) 고루틴 / 채널 응용
[개미 수열 (look-and-say sequence)](https://ko.wikipedia.org/wiki/%EC%9D%BD%EA%B3%A0_%EB%A7%90%ED%95%98%EA%B8%B0_%EC%88%98%EC%97%B4)을 작성해 보자

* 개미 수열은 아래와 같다. 100번째 수열의 10000번째 수를 구해보자
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

* 정규표현으로 풀어보자
  ```ruby
  next_ants = proc { |ants| ants.gsub(/((\d)\2*)/) { $1.size.to_s + $2 } }
  series = ->(n) { (1...n).reduce('1', &next_ants)

  # 1번째 ~ 6번째 개미 수열
  (1..6).map(&series) #=> %w[1 11 21 1211 111221 312211]
  ```

* 리스트 조작으로 풀어보자
  ```ruby
  next_ants = proc do |str|
    str.chars.chunk {|_|_}. map { |ch, grp| [grp.size.to_s, ch] }.join
  end
  series = ->(n) { (1...n).reduce("1", &next_ants) } }
  
  # 1번째 ~ 6번째 개미 수열
  (1..6).map(&series) #=> %w[1 11 21 1211 111221 312211]
  
  # 100번째 수열
  series.call(100) #=> 결과가 나오질 않는다
  ```
  100번째 개미수열은 대략 길이가 천억 단위인 문자열이 된다. 
  순차적으로 문자열을 계산한다면 자원도 모자라고 시간이 오래 걸린다. 

* 고루틴과 채널을 이용해 풀어보자
  ```ruby
  # 1. 개미수열을 문자열 덩이가 아닌 문자 1개씩 읽어서 처리하자
  # 2. 1~100번째 까지 동시에 수열을 만들어 내자(고루틴 99개, 채널 99개)
  # 3. n번째 채널은 작업 중인 n-1번째 채널을 읽어 넣도록 하자
  # 4. 마지막 채널에서 획득한 문자열의 10000번째 문자을 출력하자

  require 'concurrent-edge'

  Ch = Concurrent::Channel
  ants = ->r,w { r ? r.chunk {|_|_}.each {|ch, chunk| w << chunk.size.to_s; w << ch} : w << "1" }
  next_ants = proc { |r| Ch.new(size: 5).tap { |w| Ch.go { ants[r, w]; w.close }} }
  ant_series = ->nth { (1..nth).reduce(nil, &next_ants) }

  # 6번째 수열
  ant_series[6].map(&:itself) #=> ["3", "1", "2", "2", "1", "1"]
  
  # 100번째 개미수열의 10000번째 문자열, 1.5초 정도 걸린다
  ant_series[100].each_with_index { |ch,idx| (puts ch; break) if idx == 9999 }
  ```
  개미수열 문제는 lazy로 효율적으로 풀 수 있는 문제다. lazy를 사용해서도 풀어보자

* [Go By Example : Channels](https://gobyexample.com/channels) 를 [루비로 옮긴 코드](https://github.com/ruby-concurrency/concurrent-ruby/tree/master/examples/go-by-example-channels)

##### 5. 결론
- 비동기 작업을 Concurrent-ruby gem으로 간단히 구현할 수 있다
- [Progress report of "Ruby 3 Concurrency"](http://www.atdot.net/~ko1/activities/2018_RubyElixirConfTaiwan.pdf) : 루비 3에선 동시성 모델에 Guild란 개념을 도입한다. 전망이 밝다
