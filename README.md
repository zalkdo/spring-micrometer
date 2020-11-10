# Micrometer
가이드 참조 문서 : [www.mokkapps.de](https://www.mokkapps.de/blog/monitoring-spring-boot-application-with-micrometer-prometheus-and-grafana-using-custom-metrics/)
SpringBootAdmin으로도 시각화를 제공하지만 Prometheus가 전반적으로 사용됨.
## Spring Boot Actuator
https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-enabling
SpringBoot에는 애플리케이션을 프로덕션으로 푸시할때 모니터링하고 관리하는 데 도움이 되는 여러 추가 기능이 포함.
### springboot
![](https://www.mokkapps.de/static/00340a28edfffa8fc5b5a1a9d3b285d1/58bb7/spring-initializr.png)
### Actuator 활성화
    #application.properties
    management.endpoints.web.exposure.include=prometheus,health,info,metrics
### custom micrometer 추가
    @Component
    public class Scheduler {
    
        private final AtomicInteger testGauge;
        private final Counter testCounter;
    
        public Scheduler(MeterRegistry meterRegistry){
            // Counter vs. gauge, summary vs. histogram
            // https://prometheus.io/docs/practices/instrumentation/#counter-vs-gauge-summary-vs-histogram
            testGauge = meterRegistry.gauge("custom_gauge", new AtomicInteger(0));
            testCounter = meterRegistry.counter("custom_counter");
        }
    
        @Scheduled(fixedRateString = "1000", initialDelayString = "0")
        public void schedulingTask() {
            testGauge.set(Scheduler.getRandomNumberInRange(0,100));
            testCounter.increment();
        }
    
        private static int getRandomNumberInRange(int min, int max){
            if(min >= max)
                throw new IllegalArgumentException("max must be greater than min");
            Random r = new Random();
            return r.nextInt((max-min)+1)+min;
        }
    }
meterRegistry에 gauge와 counter를 추가, 인스턴스화 할수 있는 유형
- [Counter](https://github.com/micrometer-metrics/micrometer/blob/master/micrometer-core/src/main/java/io/micrometer/core/instrument/Counter.java#L25): 애플리케이션의 지정된 속성에 대한 개수만 보고.
- [Gauge](https://github.com/micrometer-metrics/micrometer/blob/master/micrometer-core/src/main/java/io/micrometer/core/instrument/Gauge.java#L23): 미터의 현재 값을 보여줌.
- [Timers](https://github.com/micrometer-metrics/micrometer/blob/master/micrometer-core/src/main/java/io/micrometer/core/instrument/Timer.java#L34): 지연시간 또는 이벤트 빈도 측정.
- [DistributionSummary](https://github.com/micrometer-metrics/micrometer/blob/master/micrometer-core/src/main/java/io/micrometer/core/instrument/DistributionSummary.java#L29): 이벤트 배포 및 간단한 요약제공
> **Tip**: Scheduler활성화를 위해 **Application.java에 @EnableScheduling 추가.
## Prometheus 연동
### prometheus.yml
    global:
        scrape_interval: 10s # How frequently to scrape targets by default
    scrape_configs:
        - job_name: 'spring_micrometer'         # The job name is assigned to scraped metrics by default.
          metrics_path: '/actuator/prometheus'  # The HTTP resource path on which to fetch metrics from targets.
          scrape_interval: 5s                   # How frequently to scrape targets from this job.
          static_configs:                       # A static_config allows specifying a list of targets and a common label set for them
            - targets: ['<<애플리케이션 ip입력>>:8080']
### prometheus docker 실행
    $ docker run -d -p 9090:9090 -v ~/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
http://localhost:9090 접속하여 custom_guage/counter_total 조회
Status>Targets의 Endpoint상태로 정상 수신인지 확인 가능.
