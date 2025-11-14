API 경로 체계

```jsx
/v1/{domain}/{type}/{role}/{resource}

- domain: auth, user, hub, hub-staff, hub-delivery, company...
- type: web (외부용) | internal (내부 서비스간 통신)
- role: public, all, master, hub-manager, drivers, company-user, producer, receiver
- resource: 실제 리소스 경로
```

### **1. Auth Domain** (인증/인가)

- 책임: 순수 인증/인가 처리
    - JWT 토큰 발급/검증
    - Keycloak 연동
    - 권한 검증 (Gateway에서 호출)
- Web APIs
    
    ```jsx
    # Public
    POST /v1/auth/web/public/signup          # 회원가입 요청
    POST /v1/auth/web/public/login           # 로그인
    
    # All (로그인 사용자)
    POST /v1/auth/web/all/refresh            # 토큰 갱신
    POST /v1/auth/web/all/logout             # 로그아웃
    
    # 관리자
    GET  /v1/auth/web/master/pending         # 가입 대기 목록
    PATCH /v1/auth/web/master/approve/{id}   # 가입 승인
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/auth/internal/validate          # 토큰 검증 (Gateway → Auth)
    GET  /v1/auth/internal/user-info/{id}    # 사용자 정보 조회
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - UserApproved (→ User, HubStaff, CompanyUser)
      - UserRejected (→ Notification)
      - UserLoggedIn (→ Analytics)
    ```
    

### **2. User Domain** (사용자 관리)

`책임: 사용자 기본 정보 관리
- 회원가입 승인 프로세스
- 사용자 프로필 (username, email, slack_id)
- 사용자 상태 관리
- 역할(Role) 매핑`

- Web APIs
    
    ```jsx
    # All
    GET  /v1/user/web/all/profile            # 내 정보 조회
    PUT  /v1/user/web/all/profile            # 내 정보 수정
    
    # Master
    GET  /v1/user/web/master/users           # 전체 사용자 목록
    PUT  /v1/user/web/master/users/{id}      # 사용자 수정
    DELETE /v1/user/web/master/users/{id}    # 사용자 삭제
    ```
    
- Internal APIs
    
    ```jsx
    GET  /v1/user/internal/validate/{id}     # 사용자 존재 확인
    GET  /v1/user/internal/bulk              # 다중 사용자 조회
    POST /v1/user/internal/create            # 승인된 사용자 생성
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - UserCreated (→ HubStaff/CompanyUser 도메인별 매핑)
      - UserDeleted (→ 모든 관련 도메인)
      - UserUpdated (→ Notification)
    ```
    

### **3. Hub Domain** (허브 인프라)

`책임: 허브 시설 및 경로 관리
- 17개 허브 정보
- 허브간 이동 경로 (P2P, Hub&Spoke 등)
- 경로 최적화 알고리즘
- Redis 캐싱`

- Web APIs
    
    ```jsx
    # All
    GET  /v1/hub/web/all/hubs                # 허브 목록 (캐시)
    GET  /v1/hub/web/all/hubs/{id}           # 허브 상세 (캐시)
    GET  /v1/hub/web/all/routes              # 경로 조회 (캐시)
    
    # Master
    POST /v1/hub/web/master/hubs             # 허브 생성
    PUT  /v1/hub/web/master/hubs/{id}        # 허브 수정
    DELETE /v1/hub/web/master/hubs/{id}      # 허브 삭제
    POST /v1/hub/web/master/routes           # 경로 등록
    ```
    
- Internal APIs
    
    ```jsx
    GET  /v1/hub/internal/validate/{id}      # 허브 존재 확인
    GET  /v1/hub/internal/calculate-route    # 최적 경로 계산
    GET  /v1/hub/internal/distance           # 거리 조회
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - HubCreated (→ HubStaff)
      - HubDeleted (→ HubStaff, Company, Inventory)
      - RouteUpdated (→ HubDelivery, AI)
    ```
    

### **4. Hub Staff Domain** (허브 직원)

`책임: 허브 소속 직원 관리
- 허브 관리자 정보
- 허브별 업체 배송 담당자 (각 허브 10명)
- 배송 순번 관리
- 담당 허브 매핑`

- Web APIs
    
    ```jsx
    # Hub Manager
    GET  /v1/hub-staff/web/hub-manager/staff          # 담당 허브 직원 목록
    POST /v1/hub-staff/web/hub-manager/staff          # 직원 배정
    PUT  /v1/hub-staff/web/hub-manager/staff/{id}     # 직원 정보 수정
    GET  /v1/hub-staff/web/hub-manager/delivery-sequence # 배송 순번 현황
    
    # Master
    GET  /v1/hub-staff/web/master/all-staff           # 전체 직원 조회
    POST /v1/hub-staff/web/master/hub-manager         # 허브 관리자 지정
    ```
    
- Internal APIs
    
    ```jsx
    GET  /v1/hub-staff/internal/next-driver/{hubId}    # 다음 배송 담당자 조회
    POST /v1/hub-staff/internal/assign-driver          # 담당자 자동 배정
    GET  /v1/hub-staff/internal/validate-manager       # 관리자 권한 확인
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - DriverAssigned (→ LastMileDelivery, Notification)
      - ManagerAssigned (→ User, Notification)
      - SequenceUpdated (→ LastMileDelivery)
    ```
    

### **5. Hub Delivery Domain** (허브간 배송)

`책임: 허브간 물류 이동
- 허브 배송 담당자 (전체 10명)
- 허브간 배송 경로 기록
- 허브 이동 상태 추적
- 실제 이동 거리/시간 기록`

- Web APIs
    
    ```jsx
    # Drivers (허브 배송 담당자)
    GET  /v1/hub-delivery/web/drivers/my-deliveries     # 내 배송 목록
    PATCH /v1/hub-delivery/web/drivers/start/{id}       # 구간 출발
    PATCH /v1/hub-delivery/web/drivers/arrive/{id}      # 구간 도착
    
    # Hub Manager
    GET  /v1/hub-delivery/web/hub-manager/deliveries    # 허브 배송 현황
    GET  /v1/hub-delivery/web/hub-manager/performance   # 배송 성과
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/hub-delivery/internal/create-routes        # 배송 경로 생성
    GET  /v1/hub-delivery/internal/status/{deliveryId}  # 배송 상태 조회
    POST /v1/hub-delivery/internal/assign-driver        # 허브 담당자 배정
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - HubDeliveryStarted (→ Tracking, Notification)
      - ArrivedAtHub (→ LastMileDelivery, Tracking)
      - HubDeliveryCompleted (→ Order, Tracking)
    ```
    

### **6. Company Domain** (업체 관리)

`책임: 업체 정보 관리
- 업체 기본 정보
- 업체 타입 (생산/수령)
- 소속 허브 매핑`

- Web APIs
    
    ```jsx
    # All
    GET  /v1/company/web/all/companies                  # 업체 목록
    GET  /v1/company/web/all/companies/{id}             # 업체 상세
    
    # Company User
    PUT  /v1/company/web/company-user/my-company        # 내 업체 수정
    GET  /v1/company/web/company-user/my-info           # 내 업체 정보
    
    # Hub Manager
    POST /v1/company/web/hub-manager/companies          # 업체 등록
    PUT  /v1/company/web/hub-manager/companies/{id}     # 업체 수정
    DELETE /v1/company/web/hub-manager/companies/{id}   # 업체 삭제
    ```
    
- Internal APIs
    
    ```jsx
    GET  /v1/company/internal/validate/{id}             # 업체 존재 확인
    GET  /v1/company/internal/by-hub/{hubId}            # 허브별 업체 조회
    GET  /v1/company/internal/type/{id}                 # 업체 타입 조회
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - CompanyCreated (→ CompanyUser, Product)
      - CompanyDeleted (→ CompanyUser, Product, Order)
      - CompanyUpdated (→ Notification)
    ```
    

### **7. Company User Domain** (업체 사용자)

`책임: 업체 소속 직원 관리
- 업체 담당자 정보
- 업체별 권한 관리
- 업체-사용자 매핑`

- Web APIs
    
    ```jsx
    # Company User
    GET  /v1/company-user/web/company-user/profile      # 내 프로필
    PUT  /v1/company-user/web/company-user/profile      # 프로필 수정
    
    # Hub Manager
    GET  /v1/company-user/web/hub-manager/users         # 허브 업체 사용자
    POST /v1/company-user/web/hub-manager/assign        # 업체 사용자 배정
    
    # Master
    GET  /v1/company-user/web/master/all-users          # 전체 업체 사용자
    POST /v1/company-user/web/master/create             # 사용자 생성
    DELETE /v1/company-user/web/master/delete/{id}      # 사용자 삭제
    ```
    
- Internal APIs
    
    ```jsx
    GET  /v1/company-user/internal/validate/{id}        # 사용자 검증
    GET  /v1/company-user/internal/by-company/{companyId} # 업체별 사용자
    POST /v1/company-user/internal/create-mapping       # 매핑 생성
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - CompanyUserCreated (→ Auth, Notification)
      - CompanyUserDeleted (→ Auth, Order)
      - CompanyUserMapped (→ Company)
    ```
    

### **8. Product Domain** (상품)

`책임: 상품 카탈로그
- 상품 마스터 정보
- 업체별 상품 매핑
- 상품 메타데이터`

- Web APIs
    
    ```jsx
    # All
    GET  /v1/product/web/all/products                   # 상품 목록
    GET  /v1/product/web/all/products/{id}              # 상품 상세
    
    # Company User (생산업체)
    POST /v1/product/web/producer/products              # 상품 등록
    PUT  /v1/product/web/producer/products/{id}         # 상품 수정
    DELETE /v1/product/web/producer/products/{id}       # 상품 삭제
    
    # Hub Manager
    GET  /v1/product/web/hub-manager/hub-products       # 허브 상품 목록
    ```
    
- Internal APIs
    
    ```jsx
    GET  /v1/product/internal/validate/{id}             # 상품 존재 확인
    GET  /v1/product/internal/by-company/{companyId}    # 업체별 상품
    POST /v1/product/internal/bulk-create               # 대량 등록
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - ProductCreated (→ Inventory)
      - ProductDeleted (→ Inventory, Order)
      - ProductUpdated (→ Inventory)
    ```
    

### **9. Inventory Domain** (재고)

`책임: 재고 관리
- 허브별 재고 수량
- 재고 차감/복원 트랜잭션
- 재고 이력 추적
- 재고 부족 알림`

- Web APIs
    
    ```jsx
    # All
    GET  /v1/inventory/web/all/stock/{productId}        # 재고 조회
    
    # Hub Manager
    POST /v1/inventory/web/hub-manager/adjust           # 재고 조정
    GET  /v1/inventory/web/hub-manager/history          # 재고 이력
    
    # Master
    GET  /v1/inventory/web/master/all-inventory         # 전체 재고 현황
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/inventory/internal/check                   # 재고 확인
    POST /v1/inventory/internal/decrease                # 재고 차감
    POST /v1/inventory/internal/restore                 # 재고 복원
    GET  /v1/inventory/internal/available/{productId}   # 가용 재고
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - StockDecreased (→ Order)
      - StockRestored (→ Order)
      - LowStockAlert (→ Notification, Company)
      - OutOfStock (→ Order, Notification)
    ```
    

### **10. Order Domain** (주문)

`책임: 주문 오케스트레이션
- 주문 생성/취소
- Saga 패턴 구현
- 주문 상태 관리
- 타 도메인 호출 조정`

- Web APIs
    
    ```jsx
    # Company User
    POST /v1/order/web/company-user/orders              # 주문 생성
    GET  /v1/order/web/company-user/my-orders           # 내 주문 목록
    POST /v1/order/web/company-user/cancel/{id}         # 주문 취소
    
    # All
    GET  /v1/order/web/all/orders/{id}                  # 주문 상세
    
    # Hub Manager
    PUT  /v1/order/web/hub-manager/orders/{id}          # 주문 수정
    GET  /v1/order/web/hub-manager/hub-orders           # 허브 주문 목록
    ```
    
- Internal APIs
    
    ```jsx
    PATCH /v1/order/internal/status/{id}                # 상태 변경
    GET   /v1/order/internal/validate/{id}              # 주문 검증
    POST  /v1/order/internal/complete/{id}              # 주문 완료
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      # 주문 생성 Saga
      - OrderCreated → InventoryDomain (재고 확인)
      - StockConfirmed → HubDeliveryDomain (허브 배송 생성)
      - HubDeliveryCreated → LastMileDeliveryDomain (업체 배송 생성)
      - DeliveryCreated → NotificationDomain (알림 발송)
      
      # 주문 취소 Saga (보상 트랜잭션)
      - OrderCancelled → InventoryDomain (재고 복원)
      - OrderCancelled → HubDeliveryDomain (배송 취소)
      - OrderCancelled → NotificationDomain (취소 알림)
      
      # 실패 처리
      - OrderFailed → 모든 관련 도메인 (롤백)
    ```
    

### **11. Last Mile Delivery Domain** (업체 배송)

`책임: 허브→업체 최종 배송
- 업체 배송 담당자 배정
- 업체 배송 경로 기록
- 배송 완료 처리
- 고객 서명/수령 확인`

- Web APIs
    
    ```jsx
    # Company Drivers
    GET  /v1/last-mile/web/drivers/my-deliveries        # 내 배송 목록
    PATCH /v1/last-mile/web/drivers/start/{id}          # 배송 시작
    PATCH /v1/last-mile/web/drivers/complete/{id}       # 배송 완료
    POST /v1/last-mile/web/drivers/signature/{id}       # 서명 등록
    
    # Receiver
    GET  /v1/last-mile/web/receiver/track/{id}          # 배송 추적
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/last-mile/internal/create                  # 업체 배송 생성
    POST /v1/last-mile/internal/assign-driver           # 담당자 배정
    GET  /v1/last-mile/internal/daily-schedule          # 일일 배송 일정
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - LastMileStarted (→ Tracking, Notification)
      - DeliveryCompleted (→ Order, Tracking, Notification)
      - SignatureReceived (→ Order)
    ```
    

### **12. Tracking Domain** (추적)

`책임: 통합 배송 추적
- 실시간 위치 추적
- 배송 이력 조회
- 예상 도착 시간
- 상태 변경 이벤트`

- Web APIs
    
    ```jsx
    # All (모든 로그인 사용자)
    GET  /v1/tracking/web/all/track/{deliveryId}        # 실시간 추적
    GET  /v1/tracking/web/all/history/{orderId}         # 배송 이력
    
    # Company User
    GET  /v1/tracking/web/company-user/my-shipments     # 내 배송 현황
    
    # Receiver (수령업체)
    GET  /v1/tracking/web/receiver/incoming             # 수령 예정 목록
    GET  /v1/tracking/web/receiver/eta/{deliveryId}     # 예상 도착 시간
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/tracking/internal/update-location          # 위치 업데이트
    POST /v1/tracking/internal/update-status            # 상태 업데이트
    GET  /v1/tracking/internal/aggregate/{orderId}      # 통합 추적 정보
    POST /v1/tracking/internal/calculate-eta            # ETA 계산
    ```
    
- 구독 이벤트
    
    ```jsx
    Subscriptions:
      - HubDeliveryStarted → 허브 배송 추적 시작
      - ArrivedAtHub → 허브 도착 기록
      - LastMileStarted → 업체 배송 추적 시작
      - DeliveryCompleted → 배송 완료 기록
      - LocationUpdated → 실시간 위치 갱신
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - ETAUpdated (→ Notification, Order)
      - DelayDetected (→ Notification, AI)
      - MilestoneReached (→ Notification)
    ```
    

### 13**.** **Notification Domain** (알림)

`책임: 모든 알림 처리
- Slack 메시지 발송
- 이벤트 기반 알림
- 스케줄 알림
- 알림 이력 관리`

- Web APIs
    
    ```jsx
    # All
    POST /v1/notification/web/all/send                  # 메시지 발송
    
    # Master
    GET  /v1/notification/web/master/history            # 발송 이력
    POST /v1/notification/web/master/schedule           # 스케줄 등록
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/notification/internal/order-alert          # 주문 알림
    POST /v1/notification/internal/delivery-update      # 배송 상태 알림
    POST /v1/notification/internal/daily-schedule       # 일일 스케줄
    ```
    
- 발행 이벤트
    
    ```jsx
    Subscriptions:
      - OrderCreated → 발송 시한 알림
      - HubDeliveryStarted → 허브 출발 알림
      - DeliveryCompleted → 완료 알림
      - LowStockAlert → 재고 부족 알림
    ```
    

### 14. **AI Integration Domain** (AI 통합)

`책임: 외부 AI 서비스 연동
- Gemini API 호출
- 배송 시간 예측
- 최적 경로 제안
- 네이버 Map API 연동`

- Internal APIs
    
    ```jsx
    POST /v1/ai/internal/predict-delivery-time          # 배송 시간 예측
    POST /v1/ai/internal/optimize-route                 # 경로 최적화
    POST /v1/ai/internal/departure-deadline             # 발송 시한 계산
    POST /v1/ai/internal/geocoding                      # 주소→좌표
    POST /v1/ai/internal/naver-route                    # 네이버 경로
    ```
    
- 발행 이벤트
    
    ```jsx
    Subscriptions:
      - OrderCreated → 발송 시한 계산
      - RouteRequested → 최적 경로 계산
    ```
    

(나중에 해볼 꺼)

### 15. **Analytics Domain** (분석) - 선택

`책임: 비즈니스 인텔리전스
- 배송 성과 분석
- KPI 대시보드
- 리포트 생성`

- Web APIs
    
    ```jsx
    # Master
    GET  /v1/analytics/web/master/dashboard             # 전체 대시보드
    GET  /v1/analytics/web/master/reports/performance   # 성과 리포트
    GET  /v1/analytics/web/master/reports/financial     # 재무 리포트
    POST /v1/analytics/web/master/reports/custom        # 커스텀 리포트
    
    # Hub Manager
    GET  /v1/analytics/web/hub-manager/hub-metrics      # 허브 메트릭스
    GET  /v1/analytics/web/hub-manager/driver-performance # 배송 성과
    GET  /v1/analytics/web/hub-manager/efficiency       # 효율성 분석
    
    # Company User
    GET  /v1/analytics/web/company-user/company-stats   # 업체 통계
    GET  /v1/analytics/web/company-user/order-trends    # 주문 트렌드
    ```
    
- Internal APIs
    
    ```jsx
    POST /v1/analytics/internal/collect-metrics         # 메트릭 수집
    POST /v1/analytics/internal/aggregate-data          # 데이터 집계
    GET  /v1/analytics/internal/kpi/{domain}            # 도메인별 KPI
    POST /v1/analytics/internal/generate-report         # 리포트 생성
    ```
    
- 구독 이벤트
    
    ```jsx
    Subscriptions:
      - OrderCreated → 주문 메트릭
      - OrderCancelled → 취소율 분석
      - DeliveryCompleted → 배송 성과
      - StockDecreased → 재고 회전율
      - UserLoggedIn → 사용자 활동
      - DelayDetected → 지연 분석
      # ... 모든 주요 비즈니스 이벤트
    ```
    
- 발행 이벤트
    
    ```jsx
    Events:
      - ReportGenerated (→ Notification)
      - AnomalyDetected (→ Notification, AI)
      - KPIThresholdReached (→ Notification)
    ```
    

# 송장

## 결제

## 도메인 간 의존 관계

![image.png](attachment:06dde727-c9b6-4ad2-b0a5-0194a24e412c:image.png)

![image.png](attachment:24f4295e-95b3-4015-8c69-0734ddc5b86f:image.png)
