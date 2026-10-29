# 한강로컬패스 구현 명세

기준 2026-10-29  
문서 상태 설계안  구현 및 실제 API 인증 호출 미실시  
방향과 착수 판단은 [PRD](PRD.md)  외부 사실과 이용권은 [API 근거](API_EVIDENCE.md)  

## 01 기존 자료 분석과 결정 상태

확인 자료 README와 전체 저장소 트리  docs/ideation-worksheet.md  docs/API_EVIDENCE.md  docs/prd-auto-healing.md  references/01-practice-guide.md  references/02-worksheet-template.md  references/system-methodology.md  
현재 README는 프로젝트명만 포함  앱 코드와 테스트 및 배포 파일 없음  
PRD.md는 사용자 삭제 커밋 70b262a 이후 부재  이번 요청에 따라 재생성  
이전 퀴즈 PRD의 모델과 호출 횟수 및 시간 제한은 한강 서비스의 확정값으로 이관 금지  
실습 가이드는 역할 분리와 최소 3개 장애 재현 및 제한 복구와 독립 검증을 안내  
별도 평가표와 Kubernetes 및 커버리지 80% 강제 요건은 저장소에서 미확인  

| 검토 항목 | 기존 누락 | 이번 설계 결론 | 상태 |
|---|---|---|---|
| 고객과 차별 | 지도 대비 기능 부재 단정 근거 부족 | 지도 제공 기능과 중복 인정  통합 제약 검증 효용 인터뷰 | 검증 필요 |
| 여행 경계 | 접근과 대기 및 귀가와 인원 범위 없음 | 선착장 입구 시작  1인 편도  대기 포함  귀가 제외 | 설계 제안 |
| 계산 | 시간 검증만 표기 | 운항 적용기간과 운영시간 및 메뉴 단위 검증 | 설계 제안 |
| 화면 | 결과와 오류 계약 없음 | 입력과 결과 및 상세 패널  상태별 표시 | 설계 제안 |
| 구현 | API와 DB 및 Tool 계약 없음 | F01~F07 및 내부 계약과 최소 4개 테이블 | 설계 제안 |
| 운영 | 재시도와 유효성 한도 없음 | 공유 호출 예산과 유효 캐시 및 중단 조건 | 설계 제안 |
| 한강버스 | 필수 유지 선택 완료 | 시간표 확보 전 서비스 착수 보류 | 확정 사항 |
| 실시간 | 개발자용 명세 없음 | 좌석과 대기 및 결항 자동 반영 P0 제외 | 현 확보 자료로 P0 구현 불가 |

확정 사항은 기존 사용자 결정  설계 제안은 권장 구현안  검증 필요는 확인 전 수용 불가  
구현 불가는 기술 일반의 불가능이 아닌 현재 자료와 3일 범위의 미지원 의미  
예시 문서의 가상 기업 데이터와 수치 및 평가 항목을 실제 서비스 요구로 전용 금지  

## 02 여행 범위와 사용자 확인 항목

다음 기본값은 제안  기존에 사용자가 확정한 사실로 표시 금지  

| 항목 | 권장 기본값 | 사용자 확인과 처리 |
|---|---|---|
| 출발 장소 | 여의도 선착장 입구 | 입력 화면에서 시작 지점 표시  여의도역 출발로 해석 금지 |
| 접근시간 | 경계 밖 접근 0분 | 선착장까지 이동 제외 안내  임의 출발지 P1 |
| 전체 시간 | 180분 | 입력 시각부터 종료까지  대기와 준비 및 승선과 방문 포함 |
| 왕복과 귀가 | 편도  배편 1회  귀가 제외 | 첫 요청에 범위 동의  왕복 요구 시 미지원 응답 |
| 인원 | 1인 | 다인은 미지원  예산을 인원으로 임의 나누지 않음 |
| 예산 | 식비 30000원 | 승선료와 교통비 및 입장료 제외  총여행비 통과 표현 금지 |
| 승선 준비 | 사용자 설정 15분 제안 | 개인 여행 공식 기준 미확인  확정 대기시간으로 표시 금지 |
| 하선 준비 | 사용자 설정 5분 제안 | 평균 실측치가 아닌 계획 여유시간 |
| 시장 체류 | 사용자 설정 30분 제안 | 장소 실제 운영시간과 구분 |
| 식사 체류 | 사용자 설정 40분 제안 | 음식 제공 지연과 대기열 보장 제외 |
| 선호 | 전통시장과 로컬 음식 | 문화시설과 행사는 P1  미적용 선호를 응답에 표시 |
| 혼잡 회피 | P0 미적용 | 지원 장소와 시각 자료 확보 전 수치 및 효과 생성 금지 |
| 음식 가격 누락 | 예산 UNKNOWN | 알려진 가격 소계만 표시  누락을 0원으로 계산 금지 |
| 도보 경로 누락 | 시간 UNKNOWN | 직선거리로 도보시간 대체 금지  검증 통과 후보에서 제외 |

준비와 체류시간은 화면에서 조정 가능  제안값의 출처는 USER_SETTING  
단체 예약의 30분 준비 기준을 개인 여의도 출발에 일반화 금지  
귀가와 다인 및 임의 출발지 지원 여부는 개발 전 사용자 확인 필요  확인 전 위 범위만 개발 견적 기준  
운영사 예약이나 좌석 확보 및 승선 보장은 서비스 범위 밖  

## 03 상태와 사용자 시나리오

PASS FAIL UNKNOWN은 각 검증 항목의 판정  
VALIDATED_PLANNED는 예정 시간표 기준 필수 시간과 예산 및 영업정보 모두 PASS  
NEEDS_CONFIRMATION은 실제 배편과 일부 장소는 있으나 가격 또는 영업 등 필수 항목 UNKNOWN  
NO_FEASIBLE는 확보한 근거에서 운항 부재나 시간 및 가격 충돌 FAIL  
UNAVAILABLE은 운항 자료 사용 불가 또는 핵심 조회 실패로 판정 자체 불가  
어떤 결과에도 현재 출항 보장이라는 문구 사용 금지  

| ID와 입력 | 처리 | 화면 결과와 사유 | 완료 기준 |
|---|---|---|---|
| S01 정상  여의도와 미래 시각 180분 30000원 | 유효 배편과 확인된 장소 및 경로로 계산 | 예정 일정과 배편 및 메뉴 근거 | 필수 판정 모두 PASS  종료가 제한 이내 |
| S02 운항편 없음 | 적용기간과 날짜 및 출발 가능 시각 필터 | NO_FEASIBLE  NO_SAILING  조회 범위 표시 | 가공한 배편 생성 없음  실제 조회된 다른 시각만 제안 |
| S03 시간 부족 | 준비와 운항 및 도보와 체류 합산 | NO_FEASIBLE  TIME_EXCEEDED  초과분 표시 | 줄일 수 있는 사용자 체류 또는 다른 실제 배편만 제안 |
| S04 메뉴 가격 미확인 | 가격 null 유지  시간만 별도 판정 | NEEDS_CONFIRMATION  PRICE_UNKNOWN  식비 미검증 | 가격 합계로 통과하지 않음  실제 메뉴 확인 경로 표시 |
| S05 외부 API 장애 | 제한 재시도 후 유효 캐시 재검증 | 캐시 사용과 취득 시각 또는 UNAVAILABLE | 오래된 자료로 최신 조회 성공 표시 없음 |
| S06 영업시간 미확인 | 휴무와 운영 구간 검증 UNKNOWN | NEEDS_CONFIRMATION  HOURS_UNKNOWN | 방문 가능 확정 표현 없음 |
| S07 도보 경로 없음 | 실패 구간 식별  다른 실제 후보 1회 | 확인 필요 또는 대체 후보와 경로 | 누락 구간의 이동시간 발명 없음 |

## 04 기능 계약

담당자 A 외부 데이터  B 백엔드와 검증  C 프론트엔드  D 오케스트레이션  
난이도는 상대 작업 복잡도  실측 공수 아님  전 기능 P0  

| ID 목적과 담당 | 입력과 필수 데이터 | 처리와 정상 출력 | 예외와 완료 조건 | 선행과 난이도 |
|---|---|---|---|---|
| F01 조건 수집  C와 B | 날짜와 시각  분 단위 시간  식비  선호  준비와 체류 설정 | Pydantic 범위 및 시각 검증  정규화 Request | 지난 시각과 미지원 범위 422  오류 필드 표시 | 계약 확정  낮음 |
| F02 배편 조회  A와 B | 선착장 ID  여행 날짜  권리 확인 운항자료 | 날짜별 유효 배편과 출발 및 도착  출처 반환 | 기간 밖 또는 공휴일 분류 불명 제외  확실한 부재와 조회실패 구분 | 사용권과 적용표  높음 |
| F03 장소 후보  A와 B | 도착 선착장  시장 및 업소와 가격 및 주소 근거 | 주소 정규화와 좌표 매칭  시장 및 음식점 개별 표시 | 인근 음식점과 시장 입점 구분  미확인 관계 추정 금지 | 출처 및 지오코딩 권리  높음 |
| F04 후보 생성  D | Request와 실제 SourceBundle | 허용 ID 안에서 시장과 식사 순서 및 2개 이하 후보 | 알 수 없는 ID와 형식 오류 거부  1회 재계획 후 종료 | F02와 F03 및 도구  중간 |
| F05 검증  B | 후보  경로 초 단위 시간  배편  메뉴가격  영업구간 | 결정적 계산과 항목별 PASS FAIL UNKNOWN | 미확인값 0 대체 금지  경계값 테스트 통과 | 정규화 스키마  높음 |
| F06 결과  C | Result와 Evidence 및 타임라인 | 일정표  경로와 장소  출처와 미확인 안내 | 로딩과 빈 결과 및 오류 상태 재현  채팅만으로 제공 금지 | API 계약과 F05  중간 |
| F07 제한 복구  B와 D | 오류 종류  시도와 기한  유효 캐시 | 재시도 또는 유효 캐시 후 별도 재검증 | 모든 실패는 사유로 종료  복구 행동 로그와 검증 기록 | 공유 한도와 저장 정책  중간 |

## 05 화면 IA와 요청 흐름

독립 페이지 5개 대신 입력 화면과 결과 화면 및 결과 내 지도와 상세 패널 구성 제안  

| 화면 목적과 구성 | 행동과 내부 요청 | 표시 데이터 | 로딩과 없음 및 오류 |
|---|---|---|---|
| UI01 메인  서비스 경계와 입력 폼 | 선착장 선택 GET /api/v1/piers  생성 POST /api/v1/itineraries | 지원 출발지  설정값  예정 운항 안내 | 선착장 로딩  미지원 출발지 차단  422 필드 안내 |
| UI02 결과  후보 목록과 검증 상태 | 후보 선택  재생성 | 운항편과 순서  이동 및 체류  도착시각  식비 상태 | 처리 단계 표시  NO_FEASIBLE와 UNAVAILABLE 분리  재요청 버튼 |
| UI03 지도 및 장소 패널 | 장소 선택 GET /api/v1/places  외부 지도 열기 | 선착장과 경로선  유형  영업과 가격 확인 상태  출처 | 지도 SDK 실패 시 좌표와 주소 및 외부 지도 링크 제공  시간은 경로 API 근거 유지 |
| UI04 상세 타임라인 | 후보 열기  체류 수정 후 POST /api/v1/itineraries/validate | 시작  준비  대기  승선  하선  시장 이동  식사  종료 | 수정 검증 중 이전 통과 표식 해제  최신 검증 실패 이유 표시 |
| UI05 오류 및 대안 패널 | 시각과 시간 예산 수정  재생성 | 원인 코드  실패 구간  실제 확보한 대안 | 가격 미확인을 시간 초과로 바꾸지 않음  다른 실제 후보 없으면 없다고 표시 |

REST 키는 서버 전용  지도 JavaScript 키는 등록 도메인 제한  키 종류별 정책 확인 필요  
장소 맛집 표현과 별점 생성 금지  추천 이유는 사용자 선호와 거리 및 시간 조건  

## 06 결정적 일정 계산

모든 시각 timezone-aware Asia/Seoul  API는 ISO 8601 +09:00  내부 계산은 초 정수  
여행 날짜는 출발 현지 날짜  P0 자정 초과 일정 미지원  
다음 기호는 데이터 필드와 사용자 설정의 계산 정의  실제 운항값 아님  

```text
T0 = user.start_at
deadline = T0 + duration_min * 60
pier_arrival = T0 + access_seconds
ready_at = pier_arrival + boarding_buffer_min * 60
boardable = ready_at <= sailing.depart_at
land_at = sailing.arrive_at + alighting_buffer_min * 60
market_arrival = land_at + walk_pier_market_seconds
market_end = market_arrival + market_stay_min * 60
meal_arrival = market_end + walk_market_restaurant_seconds
end_at = meal_arrival + meal_stay_min * 60
within_time = end_at <= deadline
known_meal_cost = sum(verified_unit_price_krw * selected_quantity)
within_budget = meal_selected and all_prices_known and known_meal_cost <= meal_budget_krw
```

출항 전 남는 시간은 waiting_seconds = depart_at - ready_at  전체 시간에 자동 포함  
도보 초는 제공사 응답 보존  화면 분 표시는 올림  판정은 반올림 전 초 사용  
배편 출발과 도착은 동일 운항편과 순서의 실제 원자료 참조  도착이 출발보다 앞서면 데이터 오류  
정차 선착장 운영기간과 시간표 valid_from 및 valid_to와 요일 및 휴일 규칙 모두 일치 필요  
휴일 캘린더를 임의 구성하지 않음  대상 날짜의 공식 운항 구분 확인 전 HOLIDAY_RULE_UNKNOWN  
개정본 적용기간 겹침 시 공식 개정 우선순위 확인  최신 수집 시각만으로 자동 덮어쓰기 금지  
기간 종료가 없는 공지는 무기한 유효로 추정 금지  대상 날짜 적용과 변경 공지 확인 및 만료시각 별도 기록  
시장과 식당의 방문 시작부터 체류 종료까지 하나의 영업구간 안에 포함 필요  
휴무와 브레이크타임 확인  현장 대기와 임시휴무는 미확인 안내  
인허가 영업상태 정상은 해당 방문 시각의 영업시간 근거가 아님  
식당 가격은 주문할 메뉴명과 수량 및 단가의 출처 필요  상품 ID 연결 불명 시 PRICE_UNKNOWN  
시간 미확인과 예산 미확인은 UNKNOWN  확정 충돌만 FAIL  둘을 NO_FEASIBLE로 합치지 않음  
P0 단일 배편 환승 없음  환승 요구는 UNSUPPORTED_SCOPE  다음 배편 연결을 계산한 것처럼 표시 금지  
귀가 포함 기능 확장 시 end_at에 실제 복귀 구간 추가 필요  현재 계산에 포함했다고 표시 금지  
LLM은 운항시각과 이동시간 및 가격 수정 권한 없음  제안 가능한 값은 허용 장소 ID와 순서  
P0 순서는 승선 → 시장 → 식사로 고정  Candidate.order는 이 순서와 일치해야 하며 서버에서 강제  자유 순서 최적화 P1  
restaurant_id가 없거나 주문 메뉴 및 수량이 없으면 meal_selected=false  budget=UNKNOWN과 PRICE_UNKNOWN  공집합 소계0은 통과 근거 아님  
식사 수량은 1 이상  시간과 가격 및 좌표 값의 음수와 좌표 범위 오류는 데이터 오류  
후보 내 필수 항목 중 FAIL이 하나라도 있으면 해당 후보 제외  FAIL이 없고 필수 항목 모두 PASS이면 통과  나머지는 확인 필요  
결과 집계는 필수 운항자료 불가 우선 UNAVAILABLE  자료가 있으면 통과 후보1개 이상 VALIDATED_PLANNED  통과 없이 확정 FAIL이 없는 미확인 후보가 있으면 NEEDS_CONFIRMATION  전부 제외면 NO_FEASIBLE  

## 07 내부 FastAPI 계약

이 절의 주소는 설계안  실제 외부 공공 API 주소는 API_EVIDENCE만 사용  
내부 API는 동일 출처 웹 호출  회원 인증 없음  서버측 호출량 제한과 입력 길이 제한 적용 제안  
CORS는 배포 프론트 출처만 허용  공개 인터넷 운영 전 한도와 예산 승인 필요  
GET은 Request JSON 없음  표의 query는 URL 파라미터 계약  
JSON 예시는 데이터가 아닌 타입 스키마  운항 및 장소 가상 레코드 없음  

| Method와 Path | Request 계약 | Response 계약 | 오류 | 인증 |
|---|---|---|---|---|
| GET /api/v1/piers | query 없음 | PierList | 503 DATA_UNAVAILABLE | 없음 |
| GET /api/v1/sailings | pier_id str 필수  date YYYY-MM-DD 필수 | SailingList | 422 입력  503 자료  빈 목록 200 | 없음 |
| GET /api/v1/places | arrival_pier_id str 필수  kind MARKET 또는 RESTAURANT | PlaceList | 422 입력  503 조회  빈 목록 200 | 없음 |
| POST /api/v1/itineraries | TripRequest JSON | PlanResult JSON | 422 입력  429 한도  503 핵심 자료  504 전체 기한 | 없음 |
| POST /api/v1/itineraries/validate | ValidateRequest JSON | PlanResult JSON | 409 토큰 만료  422 형식  503 근거 부재 | 없음 |
| GET /health/live | 없음 | HealthResponse | 프로세스 응답 불가 외 200 | 없음 |
| GET /health/ready | 없음 | HealthResponse | DB 또는 필수 운항자료 불가 503 | 없음 |

NO_FEASIBLE와 NEEDS_CONFIRMATION은 처리 완료의 도메인 결과이므로 200  
외부 키 오류를 클라이언트 401로 전달하지 않음  서버 의존성 장애 503으로 매핑  
주요 reason_code는 아래 계약  메시지와 함께 검증 항목별 reason_codes로 전달  

| 코드 | 의미와 판정 |
|---|---|
| NO_SAILING | 정상 조회 범위 내 가능한 배편 없음  FAIL |
| INVALID_PERIOD 또는 HOLIDAY_RULE_UNKNOWN | 전자는 적용기간 충돌 FAIL  후자는 운항일 구분 UNKNOWN |
| TIME_EXCEEDED 또는 BUDGET_EXCEEDED | 근거가 있는 시간 또는 식비 초과 FAIL |
| PRICE_UNKNOWN 또는 HOURS_UNKNOWN 또는 ROUTE_UNKNOWN | 필수 가격과 영업 또는 구간시간 미확인 UNKNOWN |
| SOURCE_NOT_ALLOWED 또는 SOURCE_EXPIRED | 해당 근거 사용 차단  필수 운항 근거 없으면 UNAVAILABLE |
| LLM_ID_INVALID 또는 LIMIT_EXCEEDED | 임의 식별자 거부 또는 공유 한도 소진  재계획 한도 내 수정 후 종료 |

오류 공통 JSON은 아래 계약  details에 키와 외부 원문 응답 포함 금지  

```json
{"request_id":"string","error":{"code":"string","message":"string","field":"string|null","retryable":false}}
```

요청과 응답 JSON의 OpenAPI 연결 설계  ref는 아래 Pydantic 모델 표의 JSON 필드에 대응  실제 생성된 명세나 호출 결과가 아님  

```json
{
  "GET /api/v1/piers": {
    "request_body": null,
    "query": {},
    "response": {"$ref": "#/components/schemas/PierList"}
  },
  "GET /api/v1/sailings": {
    "request_body": null,
    "query_schema": {"type": "object", "required": ["pier_id", "date"], "properties": {"pier_id": {"type": "string"}, "date": {"type": "string", "format": "date"}}},
    "response": {"$ref": "#/components/schemas/SailingList"}
  },
  "GET /api/v1/places": {
    "request_body": null,
    "query_schema": {"type": "object", "required": ["arrival_pier_id"], "properties": {"arrival_pier_id": {"type": "string"}, "kind": {"enum": ["MARKET", "RESTAURANT"]}}},
    "response": {"$ref": "#/components/schemas/PlaceList"}
  },
  "POST /api/v1/itineraries": {
    "request_body": {"$ref": "#/components/schemas/TripRequest"},
    "response": {"$ref": "#/components/schemas/PlanResult"}
  },
  "POST /api/v1/itineraries/validate": {
    "request_body": {"$ref": "#/components/schemas/ValidateRequest"},
    "response": {"$ref": "#/components/schemas/PlanResult"}
  },
  "GET /health/live": {"request_body": null, "response": {"$ref": "#/components/schemas/HealthResponse"}},
  "GET /health/ready": {"request_body": null, "response": {"$ref": "#/components/schemas/HealthResponse"}}
}
```

places의 kind 생략 시 MARKET과 RESTAURANT 둘 다 조회  데이터가 없으면 빈 목록과 sources 반환  

Pydantic v2 모델 제안  extra=forbid  timezone과 지원 범위 검증  

| 모델 | 필수 필드와 타입 |
|---|---|
| TripRequest | start_pier_id:str  start_at:datetime  duration_min:int  meal_budget_krw:int  preferences:list[str]  party_size:Literal[1]  trip_mode:Literal[ONE_WAY]  return_required:Literal[false]  boarding_buffer_min:int  alighting_buffer_min:int  market_stay_min:int  meal_stay_min:int |
| TripRequest 범위 | duration_min 30~360  budget 0~1000000  준비 0~60  체류 5~180  선호 최대5개 각50자  start_at 미래 7일 이내  범위는 정책 제안 |
| Pier | id:str  name:str  address:str  lat:float\|null  lon:float\|null  source_id:str  valid_from:date  valid_to:date\|null |
| Sailing | id:str  from_pier_id:str  to_pier_id:str  depart_at:datetime  arrive_at:datetime  service_date:date  source_id:str  operation_status:Literal[PLANNED] |
| Place | id:str  kind:enum  name:str  address:str  lat:float\|null  lon:float\|null  menu_name:str\|null  unit_price_krw:int\|null  hours:list[TimeWindow]\|null  price_source_id:str\|null  hours_source_id:str\|null  location_source_id:str |
| TimeWindow | opens_at:datetime  closes_at:datetime  휴무는 해당 날짜의 빈 목록  미확인은 null |
| Evidence | id:str  url:HttpUrl  provider:str  retrieved_at:datetime  valid_from:date\|null  valid_to:date\|null  expires_at:datetime\|null  rights_status:enum  license_url:HttpUrl\|null  data_status:enum |
| PierList | items:list[Pier]  sources:list[Evidence] |
| SailingList | items:list[Sailing]  sources:list[Evidence]  queried_date:date |
| PlaceList | items:list[Place]  sources:list[Evidence] |
| Candidate | id:str  sailing_id:str  market_id:str  restaurant_id:str\|null  menu_quantity:int  order:list[str] |
| TimelineStep | kind:enum  entity_id:str\|null  start_at:datetime\|null  end_at:datetime\|null  duration_seconds:int\|null  source_ids:list[str]  value_origin:enum |
| Validation | boarding:enum  time:enum  budget:enum  hours:enum  source_validity:enum  reason_codes:list[str]  exceeded_seconds:int\|null  known_meal_total_krw:int  missing_fields:list[str] |
| PlanResult | request_id:str  status:enum  candidates:list[ValidatedCandidate]  sources:list[Evidence]  warnings:list[str]  unsupported_preferences:list[str]  plan_token:str\|null |
| ValidatedCandidate | candidate:Candidate  timeline:list[TimelineStep]  validation:Validation |
| ValidateRequest | plan_token:str  candidate_id:str  market_stay_min:int  meal_stay_min:int |
| HealthResponse | status:Literal[ok,degraded,unavailable]  dependencies:dict[str,str]  checked_at:datetime |
| DataRequest | request_id:str  travel_date:date  start_pier_id:str  required_kinds:list[str]  required_evidence:list[str] |
| RouteEvidence | origin:LocationRef  destination:LocationRef  duration_seconds:int  distance_m:int  path:list[list[float]]  source_id:str  retrieved_at:datetime |
| LocationRef | kind:Literal[PIER,PLACE]  id:str  서버가 허용된 원천에서 좌표 해석 |
| SourceBundle | sailings:list[Sailing]  places:list[Place]  routes:list[RouteEvidence]  sources:list[Evidence]  missing_fields:list[str] |
| ToolError | tool_name:str  dependency:str  error_code:str  retryable:bool  attempts:int  source_ids:list[str] |
| TripState | request_id:str  request:TripRequest  stage:str  evidence:SourceBundle  candidates:list[Candidate]  validation:list[Validation]  errors:list[ToolError]  model_calls:int  tool_calls:int  http_attempts:int  replan_count:int  deadline_at:datetime |

판정 enum PASS FAIL UNKNOWN  데이터 상태 FRESH CACHED EXPIRED UNKNOWN  이용권 UNKNOWN ALLOWED DENIED  
PlanResult.status는 03절의 네 상태  error 응답은 별도 공통 계약  
첫 미확인 이동구간 이후의 계산 시각은 null  해당 구간은 시간 미확인 표시  null 대신 임의 시각을 채우지 않음  
plan_token은 서버 서명된 요청과 후보 ID 및 허용 후보 내용과 근거 ID 및 버전 해시  10분 유효 제안  인증 수단 아님  
candidate_id는 토큰 안의 후보 중 하나만 허용  미존재 후보 422  만료된 토큰 409  
클라이언트가 보낸 가격과 이동시간을 신뢰하지 않음  재검증 시 출처 레코드와 버전 및 유효성 재조회  
토큰의 개별 근거 만료가 먼저면 10분 이내여도 재계획 필요  
근거가 저장 불가한 제공사는 허용된 요청 내 메모리 결과만 사용  토큰 재검증은 새 외부 조회와 원자료 대조  

## 08 최소 저장 모델과 캐시

PostgreSQL은 허용된 운항과 장소 근거의 지속 보관 용도  외부 원본을 무조건 복제하는 DB 아님  

| 테이블 | 필드와 관계 | 저장 이유와 갱신 |
|---|---|---|
| SourceEvidence | id PK  provider  source_url  license_url  allow_display  allow_store  allow_transform  rights_checked_at  retrieved_at  expires_at  valid_from  valid_to  revision  hash  note | 출처와 사용 허용 및 버전 추적  해당 권한 미확인은 상품 데이터 적재 차단 |
| Pier | id PK  name  address  lat  lon  source_id FK  stop_valid_from  stop_valid_to | 허용된 지원 선착장만 조회  변경 확인 시 갱신 |
| SailingSchedule | id PK  from_pier_id FK  to_pier_id FK  service_date  depart_at  arrive_at  source_id FK  revision | 대상 날짜별 배편 저장  고유키 source와 revision 및 배편구간  날짜와 개정 충돌 수동 확인 |
| Place | id PK  kind  name  address  lat  lon  menu_name nullable  unit_price_krw nullable  hours_json nullable  closed_dates  location_source_id FK  price_source_id FK nullable  hours_source_id FK nullable | 확인한 한 메뉴와 운영구간만 저장  시장 입점 관계는 별도 확인 전 없음 |

식당 좌표 근접은 인근 의미  시장 입점 관계의 증거로 사용 금지  
DB 타입 제안  앱 내부 id와 FK는 UUID  원천 upstream_id는 TEXT nullable  이름과 주소 및 URL과 개정은 TEXT  
좌표 DOUBLE PRECISION  금액과 수량 INTEGER CHECK >=0  날짜 DATE  시각 TIMESTAMPTZ  운영구간 JSONB  
이용 허용 플래그 BOOLEAN nullable  null은 미확인  권한 확인 전 true 기본값 금지  
SourceEvidence.expires_at nullable  만료 미설정은 캐시 재사용 불가  Place 가격과 시간은 null 가능  
필수 FK는 원천 레코드 존재 검사  가격 데이터는 업소 연결키 또는 담당자 대조 근거 없이 상호명 유사성만으로 연결 금지  
인덱스 SailingSchedule의 service_date와 from_pier_id  Place의 kind  P0 공간 DB 확장 도입 없음  
배편 갱신은 새 revision 적재와 품질 검사를 한 트랜잭션으로 수행  검증 실패 시 이전 유효 버전 유지  
과거 버전이 만료됐으면 유지 여부와 관계없이 서비스 사용 차단  
Itinerary 테이블은 생성하지 않음  결과는 브라우저 세션과 서명 토큰  새로고침 시 재조회  
RequestLog 테이블은 생성하지 않음  stdout JSON 로그  비밀키와 여행 입력 원문 제외  
외부 장소와 경로의 저장권이 불명확하면 장기 적재 금지  위치 및 가격 캐시 권한 확보가 선행조건  
정적 내부 ID와 출처 메타데이터만으로 검증에 필요한 실제 값이 확보됐다고 간주 금지  

| 방안 | 장점 | 한계와 선택 |
|---|---|---|
| PostgreSQL | 운항 개정과 근거 재현 및 재시작 보존 | 관리 비용 있음  기존 기술 선택 유지  위4개만 사용 |
| 요청 메모리 캐시 | 간단하고 저장 제한 대응 쉬움 | 프로세스 종료 소실  근거 재현 제한  저장 불허 제공사에 적용 |
| 파일 캐시 | 단순 배포 | 이용권과 갱신 충돌 및 동시 쓰기 관리  P0 추가 저장소로 도입하지 않음 |

유효성은 expires_at과 운항 적용기간 및 권한 모두 확인  expires_at 미정이면 유효 캐시 폴백 금지  
TTL을 모든 데이터에 일괄 24시간으로 정하지 않음  제공사 약관과 원천 갱신 및 취득시점 확인 후 A와 B가 데이터별 설정  
운항표는 일자별 적용 확인과 개정 공지 확인이 필요  취득 날짜가 오늘이라는 이유만으로 최신 보장 금지  

## 09 Multi-Agent와 Tool 계약

최소 실제 LLM 역할 2개 제안  Planner와 Executor  별도 시스템 지시와 상태 및 도구 권한  
Validator와 Recovery는 Python 서비스  함수 이름만 바꿔 LLM 에이전트로 주장 금지  
LangGraph는 실행 상태와 한도 관리 후보  도입 여부 Day 1 확정  단순 직접 상태기계도 동일 계약 충족 가능  
LLM 공급자와 모델은 미결정  Tool Calling과 JSON 스키마 지원 및 키 확보 후 선택  이전 퀴즈 모델 임의 계승 금지  
Planner는 선호 의미 해석과 장소 조합 후보 제안에 사용  검색과 산술은 Python 담당  
Executor는 필요한 증거 유형과 조회 도구의 선택 및 후속 누락 확인을 담당  실제 HTTP와 정규화는 Python Tool 수행  
이 역할의 추가 호출비와 지연이 3일 범위를 초과하면 멀티 에이전트 요건 변경을 사용자와 재합의  이름만 유지하는 우회 금지  

| 구성 | 입력 Schema | 출력 Schema | 도구와 실제 작업 | 실패와 전달 |
|---|---|---|---|---|
| Planner prepare | TripRequest | DataRequest  날짜와 선착장 및 요구 증거 유형 | request_evidence handoff | 미지원 범위는 오류  Executor로 전달 |
| Executor | DataRequest와 사용 가능 도구 목록 | SourceBundle  sailings와 places 및 routes와 sources | get_sailings  find_places  get_walk_route  실제 호출 로그 필수 | 오류는 Recovery  부분 결과와 missing_fields 전달 |
| Planner propose | TripRequest와 SourceBundle | Candidate 최대2개 | submit_candidates  허용 ID와 순서만 | 모르는 ID나 직접 만든 값 거부 |
| Validator | Request와 Candidate 및 SourceBundle | Validation과 TimelineStep | validate_candidate  06절 Python 계산 | FAIL 또는 UNKNOWN 사유를 Planner에 1회 전달 |
| Recovery | ToolError와 retry_budget 및 source_refs | RecoveryResult  action과 result 및 verified | retry_tool  read_allowed_cache  schema_check | 한도 소진은 실패  복구 후 Validator 또는 별도 조회 확인 |

Tool 입력과 출력 계약  서비스 운영 데이터 예시가 아닌 타입 정의  

```json
{
  "get_sailings": {"input": {"pier_id": "str", "date": "date"}, "output": "SailingList"},
  "find_places": {"input": {"arrival_pier_id": "str", "kinds": "list[enum]"}, "output": "PlaceList"},
  "get_walk_route": {"input": {"origin": "LocationRef", "destination": "LocationRef"}, "output": {"duration_seconds": "int", "distance_m": "int", "path": "list[coordinate]", "source_id": "str", "retrieved_at": "datetime"}},
  "validate_candidate": {"input": {"request": "TripRequest", "candidate": "Candidate", "sources": "SourceBundle"}, "output": "Validation"},
  "RecoveryResult": {"action": "RETRY|CACHE|STOP", "attempts": "int", "verified": "bool", "source_ids": "list[str]", "error_code": "str|null"}
}
```

도구는 ID로 좌표를 서버에서 조회  LLM 임의 URL과 인증키 및 좌표 호출 금지  
상태 TripState  request_id  request  stage  evidence  missing_fields  candidates  validation  errors  counters  deadline_at  
Handoff마다 필수 필드와 source_id 대조  누락 시 다음 단계 통과 금지  
흐름 Planner 준비 → Executor 실제 도구 → Planner 후보 → Validator → 필요 시 Planner 1회 수정 → Validator → 표시  
규칙 검증 실패는 Recovery 대상 API 장애와 구분  재계획은 실제 확보된 배편과 장소 안에서만 수행  

공통 한도는 검증 전 정책 제안  실측 목표로 오해 금지  
후보 최대2개  재계획 최대1회  LLM 응답 요청 최대6회  에이전트 Tool 호출 최대8회  외부 HTTP 시도 최대10회  
tool_calls는 Executor의 get_sailings와 find_places 및 get_walk_route 실행 수  handoff와 submit_candidates 및 내부 validate는 제외  
서버 내부 HTTP 재시도는 tool_calls에 더하지 않고 http_attempts에 모두 포함  LLM 재시도는 model_calls에 포함  
재시도도 HTTP 한도에 포함  Executor 모델 루프 최대3회  Planner 준비와 후보 및 수정 최대3회  
HTTP 요청 timeout 3초  LLM 요청 8초  전체 요청 40초  외부 병렬 최대2개  
어느 한도든 먼저 소진되면 LIMIT_EXCEEDED 또는 TIMEOUT으로 종료  늦은 응답 폐기  
한도 내라고 모든 경로를 반드시 시도하지 않음  도보 구간 2개와 실제 후보 1개 우선  

## 10 Auto Healing과 독립 검증

부가 기능  외부 서비스 재시작과 임의 명령 실행 없음  테스트 환경 오류 주입만 허용  

| 장애 | 감지 | 정책 제안과 폴백 | 복구 검증과 사용자 및 로그 |
|---|---|---|---|
| Timeout | HTTP 3초 초과 | 동일 요청 최대1회 재시도  대기 0.5초  이후 허용된 유효 캐시 | 새 응답 스키마와 적용기간 검증  cached 표시  attempt 기록 |
| HTTP 429 | 상태 코드 | Retry-After가 남은 기한 이내면 최대1회  아니면 중단 | 새 2xx와 내용 검증  한도 안내  제공사와 코드 기록 |
| HTTP 5xx | 상태 코드 | 최대1회  이후 유효 캐시 | 새 응답 또는 캐시 유효성 별도 검사  DEGRADED 로그 |
| 필드 누락과 잘못된 JSON | Pydantic과 파서 | 재시도 최대1회  지속 오류는 불완전 결과 | 필수 필드 검사 통과 전 복구 아님  missing_fields 로그 |
| 갱신 지연과 만료 | expires_at 및 적용일 검사 | 더 최신의 사용 허용 원천만 조회  없으면 UNKNOWN | 취득 시각과 유효성 표시  stale을 최신으로 표시 금지 |
| LLM 호출 및 형식 실패 | timeout과 schema 불일치 | 모델 요청 총한도 내 최대1회 재시도  지속 실패는 일정 생성 불가 | Validator 통과 전 성공 아님  실제 원천 자료 조회 화면 유지 |
| DB 연결 실패 | readiness와 연결 예외 | 연결 재시도1회  권리와 유효성 확인된 메모리 스냅샷만 대안 | SELECT 1과 필요한 레코드 읽기 성공 확인  없으면 503 |

동일 의존성 연속 실패 3회면 30초 회로 열림 제안  새 자동 호출 중단  냉각 후 한 프로브만 허용  
제공사 정상화 확인과 사용자에게 캐시 제공 성공은 별도 기록  
복구 수행 결과의 성공 플래그만 신뢰 금지  별도 schema와 유효기간 검사 및 새 기능 요청으로 검증  
가이드용 장애 실험 최소3종 Timeout과 JSON 누락 및 DB 연결 실패  지속 실패 종료도 별도 확인  
JSON 로그 필드 request_id  incident_id  stage  dependency  error_code  attempt  source_ids  injected_at  detected_at  recovery_started_at  verified_at  final_status  latency_ms  cache_used  verification_kind  
LLM 호출수와 도구 호출수 및 토큰 사용량 기록  API 키와 DB 비밀번호 및 사용자 입력 원문 제외  
로그 보존 7일 제안  로컬 회전 로그와 운영 플랫폼 retention 별도 설정  
복구율과 MTTD 및 MTTR은 12절 정의 적용  현재 측정 없음  

## 11 배포와 CI

개발 Docker Compose 구성 frontend와 api 및 postgres  postgres named volume 사용 제안  
운영 배포 대상 미결정  팀 제공 단일 Docker 호스트 권장  Compose는 다중 서버 가용성 보장 수단 아님  
운영 DB는 호스트 지속 볼륨과 백업 또는 외부 관리 DB 중 접근 가능한 방안 Day 1 선택  임시 컨테이너 파일시스템에만 저장 금지  
환경변수 SEOUL_API_KEY  KAKAO_REST_KEY  LLM_API_KEY  DATABASE_URL  TOKEN_SIGNING_SECRET  
키값은 저장소에 넣지 않음  GitHub Actions에서 필요한 live job 키만 GitHub Secrets 사용  
PR CI는 계약과 계산 단위 및 API 통합 테스트  외부 키 없이 고정 테스트 fixture  
빌드 단계 프론트 타입 및 빌드와 Python 검사  테스트 실패 시 Docker 이미지 배포 차단  
실제 데이터 연결은 키가 있는 별도 수동 workflow 또는 Day 3 검증  정상 서비스 응답 증거와 source_id 기록  
배포 후 live와 ready 및 실제 한 요청 확인  실패 시 직전 이미지로 복귀  DB 스키마 파괴 변경 없음  
이 절은 구현 요구  이번 문서 작업에서 코드와 CI 또는 배포 파일 생성하지 않음  

## 12 테스트와 성공 지표

fixture와 장애 주입은 자동화 테스트 전용  화면 서비스 응답의 가짜 배편과 메뉴 생성 금지  
외부 API를 실제로 호출하지 않은 fixture 테스트는 L3와 L4 증거가 아님  

| 테스트 | 입력 조건 | 기대 결과와 실제 검증법 | 담당 기능과 자동화 |
|---|---|---|---|
| T01 정상 계산 | 허용 원천과 유효 배편  시간과 가격 및 영업 확인 | VALIDATED_PLANNED  독립 재계산과 원천 대조 | F02~F06  단위와 실제 연결 |
| T02 배편 없음 | 해당 날짜의 정상 빈 조회 | NO_FEASIBLE  NO_SAILING  허구 배편 없음 | F02  자동 |
| T03 적용기간과 휴일 오류 | 기간 밖 또는 휴일 구분 누락 | 배편 제외 또는 UNKNOWN  잘못된 정상 판정 없음 | F02 F05  자동 |
| T04 지난 출발과 준비 부족 | start_at 과거 또는 ready_at이 출항 이후 | 422 또는 배편 제외  출항과 같은 시각 경계 별도 | F01 F05  자동 |
| T05 시장 접근 초과 | 실제 경로로 end_at이 deadline보다 큼 | TIME_EXCEEDED  초과 초 일치  딱 일치 시 PASS | F05  자동 |
| T06 영업시간과 휴무 | hours null  휴무  브레이크타임 | UNKNOWN과 FAIL 구분  통과 금지 | F03 F05  자동 |
| T07 가격 미확인과 상한 | null  상한 동일  상한+1 | UNKNOWN  PASS  FAIL 순서  소계와 누락 구분 | F05  자동 |
| T08 경로 실패 | 필수 구간 API 실패 | 시간 UNKNOWN  임의 속도와 직선 추정 없음 | F03 F05 F07  주입 |
| T09 API 장애와 캐시 만료 | timeout  5xx  429  malformed  expired | 한도 준수  새 검증 또는 종료  최신 표시 금지 | F07  주입 |
| T10 존재하지 않는 LLM 장소 | 미등록 ID와 임의 가격 | 후보 거부  1회 수정 이후 종료 | F04 F05  자동 |
| T11 반복 한도 | 재계획2회 또는 HTTP11번째 시도 요청 | LIMIT_EXCEEDED  추가 호출0건 | F04 F07  자동 |
| T12 DB 실패와 회로 | 연결실패3회와 지속 장애 | 회로 열림  유효 스냅샷 또는503  재시도 종료 | F07  주입 |
| T13 실제 데이터와 권리 | 대상 날짜 원천과 허용서 및 키 | HTTP와 사용 필드 및 기간 대조 증거 확보 | A와 B  수동 및 통합 |
| T14 화면 통합 | S01~S07 실제 또는 테스트 fixture 경로 | 입력과 timeline 및 지도와 오류 UI 표시  키 노출 없음 | F01 F06  통합 |

Day 3 표본 제안  대표 정상 요청5개와 불가 및 미확인 요청 각1개 이상  장애3종 각3회  
통계적 신뢰 표본이 아닌 시연 표본  실제 가능한 정상 요청이 없으면 정상 완료로 포장 금지  

| 지표 | 산식과 분모 | 수용 기준 |
|---|---|---|
| 실제 조회 성공률 | 스키마와 필수필드 및 유효기간 검사 통과 실제 HTTP 조회 / 전체 실제 조회 시도 | 결과 수와 실패 원인 보고  목표 미정 |
| 유효 일정 생성률 | VALIDATED_PLANNED 반환 / 유효 입력 요청 | 불가와 확인필요 및 의존성 실패도 분모 포함  목표 미정 |
| 검증 통과율 | 필수 항목 전부 PASS 후보 / 검증 후보 | UNKNOWN을 PASS로 계산 금지 |
| API 장애 복구율 | 새 정상 응답과 독립 검증 성공 장애 / 전체 주입 장애 | 캐시 제공은 별도 폴백 성공률로 보고 |
| 폴백 성공률 | 사용 허용 및 유효 캐시로 제한 결과 반환 / 전체 주입 장애 | 제공사 정상화와 구분 |
| 평균 응답시간 | 요청 수신부터 성공 또는 실패까지 경과 합 / 전체 유효 요청 | 실패와 timeout 포함  평균과 최대 및 표본수 표시 |
| MTTD | detected_at - injected_at 평균 | 미탐지 건수 별도  시각은 동일 호스트 기준 |
| MTTR | verified_at - injected_at 성공건 평균 | 미복구 수와 종료시간 별도  캐시 전환시간과 구분 |

미측정 수치는 결과로 작성 금지  목표값은 Day 1 계정과 실제 지연 측정 후 팀 합의  
기본 수용 조건은 실제 정상 일정1건과 T02~T12 안전 처리  100% 성공률 주장 아님  

## 13 팀별 3일 작업과 통합

Day 1 시작 시 아래 범위와 설정값 사용자 확인  확인 전에는 설계 제안  

| 시점과 책임 | 산출물 | 선행과 전달 | 완료 및 GitHub 통합 |
|---|---|---|---|
| D1 A | 사용권과 적용일 증거  API 실제 응답 | 운영사 및 키 접근  B와 D에 필드와 source 전달 | 대상 날짜 배편과 방문지 및 경로 확인  계약 동결 전 merge |
| D1 B | Pydantic 계약  최소 DB와 계산 인터페이스 | A의 실제 필드  C와 D에 OpenAPI | 입력과 결과 상태 계약 확정  오후 통합 |
| D1 C | 메인과 결과 틀 및 상태 UI | 계약  개발 fixture는 테스트 전용 | 로딩과 미확인 및 오류 표시  오후 통합 |
| D1 D | Planner와 Executor 및 Tool 스키마 | 모델 계정과 B 계약 | 실제 도구 호출1건과 한도 상태 전달  종료 전 통합 |
| D1 전원 | Go 판정 기록 | 최소 데이터와 권리  부족 항목 확인 | 미확보 시 서비스 구현 착수 보류  조사와 계약 정리는 계속 가능 |
| D2 A | 조회 어댑터와 출처 및 갱신 | D1 사용 허용 원천 | 실제 결과를 B에 전달  점심 통합 |
| D2 B | 시간과 예산 및 영업 검증 | A 근거와 경로 | T03~T07 단위 검증  오후 통합 |
| D2 C | 지도와 장소 및 타임라인 | F05 응답 | S01~S05 상태 표시  오후 통합 |
| D2 D | 실제 도구와 두 에이전트 연결 | A 어댑터와 B Validator | 재계획1회와 한도 종료  저녁 전체 정상 경로 통합 |
| D3 A | 실제 연결과 장애 근거 | 데이터 최신성 재확인 | 정상과 미검증 상태 기록  오전 merge |
| D3 B | pytest와 API 통합 및 복구 | F07 정책 | 정상과 오류 및 한도 확인  오전 merge |
| D3 C | Docker UI와 통합 배포 | CI와 호스트 및 도메인 | live와 ready 및 결과 화면 점검  오후 배포 |
| D3 D | 장애3종 실험과 시연 로그 | 통합 환경 | MTTD와 MTTR 및 실제 일정 시연  종료 전 release |

계약 확정 후 B와 C와 D 병렬 개발 가능  A의 데이터 권리와 실제 필드는 일정 완성의 선행조건  
기능 브랜치별 짧은 PR  계약 변경은 B 책임으로 C와 D 동시 반영  코드 작업은 이번 요청 밖  

## 14 축소 기준과 최종 판단

한강버스 제외한 일반 관광 추천으로 변경 금지  사용자 확정 유지  
시간표와 경로 확보 실패는 No-Go  임의 시각과 이동시간을 만들어 데모 금지  
가격만 부족하면 배편과 시장 일정은 확인 필요로 제한 제공  식비 검증 완료로 표시 금지  
식당 영업 근거가 없으면 식당 단계 미확인  시장만 방문 가능한지 별도 판정  
지도 렌더링 장애는 외부 지도 링크로 축소 가능  도보시간 근거까지 생략하는 대안은 불가  
문화행사와 혼잡도 및 왕복과 다인과 로그인 및 이력은 3일 핵심 범위 밖  

| 판단 | 조건 | 조치 |
|---|---|---|
| No-Go 현재 | 시간표의 계산 활용과 표시 권한 미확보 또는 실제 구간 계산 근거 미확보 | 핵심 일정 서비스 착수 보류  저장하는 원천은 저장 허용도 필요 |
| Conditional Go | 합법적 예정 시간표와 실제 적용 구간 및 방문지와 경로 확보  실시간 출항 미확인 | 예정 운항과 누락 항목 명시  제한 MVP 개발 및 시연 |
| Go | 합의한 P0 데이터와 정상 시나리오 검증 완료 및 운영 확인 조건 충족 | 완료한 검증 범위만 제공  예정 정보만 확보된 상태는 Conditional Go 유지 |

Go의 운영 확인 증거는 대상 날짜와 운항편에 대한 운영사 또는 공식 채널의 당일 운항 확인 및 확인 시각 기록  
자동 실시간 API를 새 P0로 추가하는 조건 아님  수동 공식 확인만 가능하면 그 시점의 확인으로 표시  이후 결항과 잔여좌석 보장 금지  
이 증거 없이 공식 예정표만 검증한 MVP는 Conditional Go로 시연 가능  정상 일정1건도 예정 기준이라고 표시  

필수 미결정  시간표 권리와 최신 적용일  영업과 메뉴 데이터의 실제 범위  LLM 계정과 모델  지도 저장조건  운영 호스트  휴일 분류 원천  
설계 확인 필요  출발 경계와 귀가 제외  준비와 체류 기본값  호출 한도와 TTL 및 목표 지표  
최대 위험3개  운항자료 권한과 개정  방문지 근거 품질  3일 연동 작업량  
현재 성공확률 수치 산정 근거 없음  Day 1 선행조건 충족 시 한 구간 MVP 가능성 있음
