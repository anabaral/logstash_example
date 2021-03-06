# logstash_example
Logstash - Pushgateway - Prometheus 연결로 메트릭을 수집하는 과정.
내용은 주로 Logstash 설정을 다룹니다.

현재 닥친 문제:

다음 그림이 구조인데

![Architecture](https://github.com/anabaral/logstash_example/blob/master/Logstash-Pushgateway-AccessLog%EB%A9%94%ED%8A%B8%EB%A6%AD%EC%88%98%EC%A7%91%EA%B5%AC%EC%A1%B0.svg)

Logstash는 로그를 처리하는 데 특화된 유틸리티이기 때문에 로그가 오는 대로 그 정보를 Pushgateway 에 보내는 것이 자연스럽지만
Pushgateway는 Prometheus 가 scrape하는 주기에 맞춰 모인 데이터를 던지는데 이 때 정보가 없는 경우 그 값이 null 이 된다.

그런데 이를테면 아래와 같이 accesslog_bytes_sum 같은 Summary 타입(Counter와 같은 성격임)의 메트릭을 수집할 때 
accesslog_bytes_sum{component="pushgateway",ctxroot="/actuator/info",exported_instance="bff-spring-mobile-0.0.1-7fbf886cfb-qmmc9",exported_job="accesslog",forwarded="-",instance="10.129.75.158:9091",job="kubernetes-service-endpoints",kubernetes_name="pushgateway",kubernetes_namespace="infra",method="GET",msa_app="bff-spring-mobile",msa_namespace="mtw-dev",path="/actuator/info",protocol="HTTP/1.1",respcode="200"} 
이 수집되는 메트릭들의 특성이
* 레이블이 하나만 달라도 다른 메트릭 항목이다. 이들은 개별적으로 쌓아 주어야 한다.
  이를테면 accesslog_bytes_sum{ path="/a", respcode="200" } 과 accesslog_bytes_sum{ path="/b", respcode="200" } 인 메트릭 값을 같은 변수에 담아 관리하게 되면 sum(accesslog_bytes_sum{respcode="200"}) 으로 통합 메트릭 값을 구할 때 의도와 다르게 부풀린 값을 얻을 것이다. 두 값들을 서로 다른 변수로 분리해 저장해야 우리가 원하는 의도대로 얻을 수 있다.
* 매 scrape 시점마다 가능한 모든 메트릭이 응답이 되어야 한다. 이게 중요하다 왜냐면
* scrape 시점에 수집되지 않은 메트릭 값은 null로 체크되기 때문이다. 아래 그래프에서 끊긴 구간은 해당 메트릭이 scrape이 안된 것을 의미한다. 
  ![null_value_graph](https://github.com/anabaral/logstash_example/blob/master/prometheus%20null%20value.png?raw=true)

문제는 이들을 내가 원하는 관점으로 -- 이를테면 respcode="500" 인 것들만 모아서(sum) 그래프로 나타내 주어야 할 때 발생한다. 
counter 타입의 메트릭 여럿을 sum 해도 원칙적으로는 counter여야 하고 따라서 시간에 따라 증가만 해야 한다. 하지만 null 은 그 메트릭만 볼 때는 단순히 값이 없는 거지만 다른 메트릭들과 sum 할 때 0으로 간주되므로 결과적으로 sum 값이 내려갔다 올라가게 된다.

위와 같은 문제를 해결하려면 scrape 시점마다 "수집된 모든" 메트릭들을 다 응답할 수 있도록 계속 준비해 주어야 하는데
logstash는 수집 전달되는 로그 기반으로 동작하므로 그게 안될 수 있다.
즉 우리에게 필요한 것은 input은 file/filebeat/fluentd 등으로 수집하더라도 output은 주기적으로 실행되는 무엇이어야 한다.
이를 구현하기 위해 heartbeat 를 사용했다:

```
    input {
      beats { # 로그 처리는 filebeat 에서 보내주는 걸로
        ...
      }
      heartbeat { # 로그 처리와는 별도로 heartbeat 설정 
        interval => 15
        type => "heartbeat"
      }
    }

    filter { 
      if [type] == "kube-logs" { # 일반 로그 처리
        ... # from beats
      }
      if [log] =~ /^ACC\|.*$/ { # access log 처리
        # 로그 쌓는 prefix 약속을 미리 정했음. 일반적인 포맷은 ipv4+ipv6 등등 고려해서 regex로 수용할 때 굉장히 복잡해져서..
        mutate {
          add_tag => ["accesslog"]
        }
        ... # 내용 파싱해 여러가지 정보 얻고
  
        ruby { # 일반적인 플러그인이 안보여서 루비 코드를 직접 사용
          ## 함수와 전역변수 선언
          init => '
    def add_val(path, namespace, container_name, method, response_code, protocol,
        x_forwarded_for, pod_name, value, x)
    ...
    end
    
    def get_val(path, namespace, container_name, method, response_code, protocol,
        x_forwarded_for, pod_name, x)
    ...
    end
    
    @@bytes_value_hash={} # 해시 형태로 만들어 둔다
    @@usecs_value_hash={}
    @@count_hash={}
    '
          ## 로그 한줄 올 때마다 실행하는 코드. 로그에서 추출한 값들을 해당되는 전역변수에 더함
          code => '
    ... # 필요한 항목을 변수에 넣고
    add_val(path, namespace, container_name, method, response_code, protocol, x_forwarded_for, pod_name, bytes.to_i, @@bytes_value_hash)
    add_val(path, namespace, container_name, method, response_code, protocol, x_forwarded_for, pod_name, usecs.to_i, @@usecs_value_hash)
    add_val(path, namespace, container_name, method, response_code, protocol, x_forwarded_for, pod_name, 1, @@count_hash)
    '
        }
      }
    } 

    filter {  # heartbeat 시점마다 모든 메트릭을 보낼 준비작업을 한다.
      # 해시와 해시용 함수들을 따로 만듬. 낮은 버전의 ruby를 써야 해서 고급진 함수들을 활용 못함 ㅠ_ㅠ
      # 모든 메트릭들을 한데 모아 문자열 메시지로 만들어 logstash event 저장소에 넣는 게 하는 일의 전부.
      if [type] == "heartbeat" {
        ruby {
          code => '
    @@message = ""
    accesslog_resptime_sum = ""
    accesslog_resptime_count = ""
    accesslog_bytes_sum = ""
    accesslog_bytes_count = ""
    # 해시에 등록된 모든 값들을 다 뿌려주기 이해 이런 코드를 사용함.. 버전 낮은 ruby 의 한계도 있고..
    @@count_hash.each { |path, v|
      v.each { |namespace, v|
        v.each { |container_name, v|
          v.each { |method, v|
            v.each {|response_code, v|
              v.each {|protocol, v|
                v.each {|x_forwarded_for, v|
                  v.each{|pod_name, v|
                    shortpath = path.gsub(/[?].*/, "")
                    #puts get_val(path, namespace, container_name, method, response_code, protocol, x_forwarded_for, pod_name, @@usecs_value_hash)
                    accesslog_resptime_sum << "accesslog_resptime_sum{ctxroot=\"#{shortpath}\",path=\"#{path}\",msa_namespace=\"#{namespace}\",msa_app=\"#{container_name}\",method=\"#{method}\",respcode=\"#{response_code}\",protocol=\"#{protocol}\",forwarded=\"#{x_forwarded_for}\",pod=\"#{pod_name}\"} " << get_val(path, namespace, container_name, method, response_code, protocol, x_forwarded_for, pod_name, @@usecs_value_hash).to_s << "\n"
                    accesslog_resptime_count << "accesslog_resptime_count{ctxroot=\"#{shortpath}\",path=\"#{path}\",msa_namespace=\"#{namespace}\",msa_app=\"#{container_name}\",method=\"#{method}\",respcode=\"#{response_code}\",protocol=\"#{protocol}\",forwarded=\"#{x_forwarded_for}\",pod=\"#{pod_name}\"} #{v}" << "\n"
                    accesslog_bytes_sum << "accesslog_bytes_sum{ctxroot=\"#{shortpath}\",path=\"#{path}\",msa_namespace=\"#{namespace}\",msa_app=\"#{container_name}\",method=\"#{method}\",respcode=\"#{response_code}\",protocol=\"#{protocol}\",forwarded=\"#{x_forwarded_for}\",pod=\"#{pod_name}\"} " << get_val(path, namespace, container_name, method, response_code, protocol, x_forwarded_for, pod_name, @@bytes_value_hash).to_s << "\n"
                    accesslog_bytes_count << "accesslog_bytes_count{ctxroot=\"#{shortpath}\",path=\"#{path}\",msa_namespace=\"#{namespace}\",msa_app=\"#{container_name}\",method=\"#{method}\",respcode=\"#{response_code}\",protocol=\"#{protocol}\",forwarded=\"#{x_forwarded_for}\",pod=\"#{pod_name}\"} #{v}" << "\n"
                  }
                }
              }
            }
          }
        }
      }
    }
    @@message << "# TYPE accesslog_resptime summary" << "\n"
    @@message << accesslog_resptime_sum
    @@message << accesslog_resptime_count
    @@message << "# TYPE accesslog_bytes summary" << "\n"
    @@message << accesslog_bytes_sum
    @@message << accesslog_bytes_count
    event.set("metric_msg", @@message)
    '
        }
      }
    }

    output {
      if [type] == "heartbeat" {
        http {
          url => "http://pushgateway.infra:9091/metrics/job/accesslog"
          http_method => "put"
          format => "message"
          message => "%{metric_msg}"
        }
      } else {
        elasticsearch {
          ... # index 처리
        }
      }
    }
```

위 코드는 기본적으로 4-space indent 가 걸려 있는데, 이는 원래 그런것이 아니라 이 내용이 k8s 용 configmap yaml 파일의 일부이기 때문이다.
그냥 logstash configuration 이라면 indent 를 굳이 주지 않아도 된다.

관련된 다른 코드를 섞어 놓아서 보기 힘들 수 있는데, 요약하면 다음과 같다.
* 로그에서 값을 추출해 모으는 로직과 주기적으로 이를 pushgateway로 쏘기 위한 로직이 분리되어 있다
* 값을 저장해 둘을 이어주는 ruby 전역변수가 유용하게 쓰인다.

