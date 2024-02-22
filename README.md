
# 서비스 시나리오

기능적 요구사항
1. 사용자는 주차 자리가 있는 주차장을 예약한다.
2. 사용자는 예약을 취소할 수 있다.
3. 주차장이 예약되면 해당 주차장의 자리는 감소한다.
4. 주차장 예약이 취소되면 해당 주차장의 자리는 증가한다.
5. 예약한 사람은 할인쿠폰 발행 대상자가 된다.
6. 예약을 취소하면 할인쿠폰 발행 대상자에서 제외된다.
7. 통합관제실에서 주차장 정보 및 예약현황을 볼 수 있다.

# 분석/설계

![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/69c9fd3a-8fcf-4a6b-97e3-99fe6eac02bc)



# 구현:

각 BC별로 대변되는 마이크로 서비스들은 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다
(각자의 포트넘버는 Reservation:8082, parking:8083, coupon:8084, gateway:8088, kafka:9092 이다)

```
cd reservation
mvn spring-boot:run

cd parking
mvn spring-boot:run 

cd coupon
mvn spring-boot:run  

cd gateway
mvn spring-boot:run

cd kafka
docker-compose up
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 reservation 마이크로 서비스).

```
package ecall.domain;

import ecall.ReservationApplication;
import ecall.domain.Canceled;
import ecall.domain.Reserved;
import java.time.LocalDate;
import java.util.Date;
import java.util.List;
import javax.persistence.*;
import lombok.Data;

@Entity
@Table(name = "Reservation_table")
@Data
public class Reservation {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String parkingId;

    private String customerId;

    private String carNumber;

    private Double amount;

    private String status;

    @PostPersist
    public void onPostPersist() {
        Reserved reserved = new Reserved(this);
        reserved.publishAfterCommit();
    }

    @PrePersist
    public void onPrePersist(){
        ecall.external.Parking parking = 
            ReservationApplication.applicationContext.getBean(ecall.external.ParkingService.class)
            .getParking(Long.valueOf(getParkingId()));

        if(parking.getParkingSpot() < 1){
            throw new RuntimeException("No more Space");
        }

    }
    
    @PreRemove
    public void onPreRemove() {
        Canceled canceled = new Canceled(this);
        canceled.publishAfterCommit();
    }

    public static ReservationRepository repository() {
        ReservationRepository reservationRepository = ReservationApplication.applicationContext.getBean(
            ReservationRepository.class
        );
        return reservationRepository;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package ecall.domain;

import ecall.domain.*;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(
    collectionResourceRel = "reservations",
    path = "reservations"
)
public interface ReservationRepository
    extends PagingAndSortingRepository<Reservation, Long> {}

```
- 적용 후 REST API 의 테스트
```
# reservation 서비스의 Reserve 처리
http POST localhost:8082/reservations parkingId="1" customerId="CUST1" carNumber="10더8612" amount="5000" status="Green" 
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/47f3fca6-0f38-4402-9b8a-727eba9a0463)


```
# Reserve 상태 확인
http localhost:8082/reservations/1
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/fb50fb4f-c227-489b-acaa-b882923b7f66)


```
# reservation 서비스의 Cancel 처리
http DELETE localhost:8082/reservations/1
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/149b4d31-a3ce-4ea1-a44d-263d9867602b)


```
# Cancel 처리 확인
http localhost:8082/reservations/1
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/99bda8b0-2ee4-46c9-9c91-83eac22424de)


# 기능테스트
```
주차장ID:1 주차장은 5개의 자리가 남아있다
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/eaa9981e-4385-44e1-98d1-04fc7c7f92e5)

```
주차장ID:2 주차장은 0개의 자리가 남아있다
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/f008f926-d4b0-42d0-9e9f-a3c98884def3)

```
주차장ID:2 주차장은 자리가 없어 예약이 불가하지만 강제로 예약해보도록 하겠다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/6b47021e-921e-43ef-a87f-12d63f211e84)
```
더 이상 자리가 없어 예약이 불가하다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/75db1e18-54cd-4531-8e70-7329166e6741)

```
주차장ID:1 주차장을 예약 하자.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/233eb81f-edae-43bd-a8e7-d20bd607b277)

```
주차장ID:1 주차장의 남은 자리를 조회하면 예약으로 인해 남은 자리가 4가 된 것을 확인할 수 있다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/a94521e4-c772-4e9a-b5fa-998436100c70)

```
예약됨 이벤트가 발행되고 이어 쿠폰 대상자로 선정되어 쿠폰이 발행됨 이벤트가 발생하였다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/aaf7defa-d5ae-40c9-8e6a-4b1996af93bc)

```
주차장ID:1 주차장의 예약을 취소해보자.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/3972597d-51ec-487a-95c4-c032c38bd0f9)

```
주차장ID:1 주차장의 예약 취소로 남은 자리가 다시 5가 된 것을 확인할 수 있다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/7021647f-432a-4d0c-8f29-db61f2574768)

```
취소됨 이벤트가 발행되고 이어 쿠폰 대상자에서 제외되며 쿠폰 회수됨 이벤트가 발생하였다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/45e1f840-f605-4cd0-9aac-74b65c0106ac)

```
단일 진입점 추가 후 동일한 시나리오 수행 결과
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/4775d625-4246-4b67-a714-51042c04385e)
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/7ef44c96-cb65-4255-8825-1e1541d08ccd)
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/749a1750-88ab-434a-956c-de5ae5d57201)

![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/6e7e434b-8444-4bd9-aef3-a8ec40ece2ad)
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/64f6a6c7-f8ec-44ec-a974-8679d2f51687)
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/7840ab3d-ca1a-4815-a022-c5ca696f698a)


```
ControlCenter 에서는 주차장 정보와 예약현황을 알 수 있다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/34908271-8afe-4ba1-b35e-c22da0edf421)

```
Parking 서비스를 다운시키고
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/c3f05254-31e0-41aa-85c6-767640478fb7)

```
ControlCenter 정보를 조회하여도 ControlCenter 서비스는 정상적으로 동작함을 알 수 있다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/aaa54b3a-edcb-4c4e-8b2e-e2ca58774020)




# 운영

## Container 운영(클라우드 배포)

클라우드 환경에 배포하기 위한 사전 작업 (ex. reservation서비스)
```
cd reservation
mvn package -B -Dmaven.test.skip=true

docker build -t [dockerhub ID]/reservation:latest .     
docker push [dockerhub ID]/reservation:latest

kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

Kafka는 helm으로 설치하였다.
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-kafka bitnami/kafka --version 23.0.5
```

모든 서비스를 컨테이너에 배포하였다.
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/7ec78aba-dac2-450b-9ef9-2f244d26f2e5)


```
로컬 테스트로 진행한 내용을 동일하게 클라우드에 배포 후 진행한다.
```
```
주차장ID:1 주차장을 예약 하자.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/cf2fb98c-a7e9-49c9-86b3-e252ac06ba22)

```
주차장ID:1 주차장 예약으로 인해 주차 자리는 1 감소하였다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/8c80eb32-8e61-4c32-b1d8-ca6ac82e77b6)

```
주차장ID:1 주차장 예약취소
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/ecebb40a-4094-4259-b8ed-97fa13bf77e1)

```
주차장ID:1 주차 자리는 반환되었다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/f01b797b-d993-4f79-a490-61374e690d7e)

```
예약->취소로 인해 예약됨/쿠폰발행됨/취소됨/쿠폰회수됨 이벤트가 발생한 것을 볼 수 있다.
```
![image](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/0c85ebce-0db1-4b17-938e-a2b011b44b48)







## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.

### 오토스케일 아웃
- Reservation 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 50프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy reservation --min=1 --max=10 --cpu-percent=50


* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 50명
- 60초 동안 실시

siege -c50 -t40S -r10 --content-type "application/json" 'ac96fd853fcc74f59a7d21451869ebe5-1642504919.ap-northeast-1.elb.amazonaws.com:8080/reservations POST {"parkingId":"1"}'

```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 스케일 아웃이 벌어지는 것을 확인할 수 있다:
![Screenshot 2024-02-23 at 2 39 32 AM](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/8ef3a495-d457-4199-9d48-673456ba6f73)


### 셀프힐링
- Reservation 서비스가 담긴 컨테이너에 장애 발생 시, 자동으로 감지하여 장애를 복구하도록 한다.

```
aapiVersion: apps/v1
kind: Deployment
metadata:
  name: reservation-liveness
  labels:
    app: reservation-liveness
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reservation-liveness
  template:
    metadata:
      labels:
        app: reservation-liveness
    spec:
      containers:
        - name: reservation-liveness
          image: iure07/reservation:latest
          args:
          - /bin/sh
          - -c
          - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "200m"            
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            exec:
              command:
              - cat
              - /tmp/healthy
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5

```
![Screenshot 2024-02-23 at 2 59 33 AM](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/4fc6d2e3-80fa-4cb3-948d-07453b6002de)


### 무중단배포
- Reservation 서비스 이미지의 새로운 버전을 반영하여 배포를 진행한다.
- 배포 시 seige로 부하를 걸고 배포 완료 시 결과로 무중단 여부를 확인한다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reservation
  labels:
    app: reservation
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reservation
  template:
    metadata:
      labels:
        app: reservation
    spec:
      containers:
        - name: reservation
          image: iure07/reservation:canary
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "200m"            
          readinessProbe:
            httpGet:
              path: '/reservations'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
```
- readinessProbe 주석 처리 후 배포 진행 시 결과
  
![Screenshot 2024-02-23 at 3 38 38 AM](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/45d6385d-efe7-42db-a1ae-eb4ec8142ebd)

- readinessProbe 주석 해제 후 배포 진행 시 결과
  
![Screenshot 2024-02-23 at 3 33 34 AM](https://github.com/Nam-Jae/MSA_FinalProject_Template/assets/34273834/35a03d6c-98ad-4fd0-8aa7-425e537bec76)


--------------------------
--------------------------






## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.
