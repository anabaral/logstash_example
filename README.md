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
* 레이블이 하나만 달라도 다른 메트릭이어서 개별적으로 쌓아 주어야 하고
* 즉 매 scrape 시점마다 가능한 모든 메트릭이 응답이 되어야 한다.
* scrape 시점에 수집되지 않은 메트릭 값은 null로 체크된다. 이들을 내가 원하는 관점으로 sum 해서 그래프로 나타내 주어야 하는데
  null 은 그 메트릭만 볼 때는 단순히 값이 없는 거지만 다른 메트릭들과 sum 할 때 0으로 갖누되므로 결과적으로 sum 값이 원칙적으로는 증가만 해야 하지만 실제로는 내려갔다 올라가게 된다.

![null_value_graph](https://github.com/anabaral/logstash_example/blob/master/prometheus%20null%20value.png?raw=true)

