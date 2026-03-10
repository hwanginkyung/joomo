
# B. 차량등록 API 명세

  

> 로컬 참고용 (gitignore). Swagger 어노테이션이 실제 스펙.

> 공통 사항(인증 헤더, 에러 코드 등)은 [00_공통_API_SPEC.md](00_공통_API_SPEC.md) 참고.

  

---

  

## 플로우 개요

  

```

[프론트] 차량번호 입력

↓

[백엔드] 1.1 차량 조회: 내부 DB 중복 확인 + car365 API 조회

↓

[프론트] 결과 표시 (이미 등록 시 빨간 글씨 "이미 등록 또는 거래된 매물입니다")

↓

[프론트] 차량 원부 파일 업로드 (+ licensePlate 함께 전송)

↓

[백엔드] 1.2 OCR 파싱 + car365 비교: Upstage OCR → car365 재조회 → 필드별 비교

↓

[프론트] OCR 결과 + 비교 결과 표시

- 불일치 필드: 사용자에게 확인 요청

- OCR 빈값 필드: 수동 입력 유도

- pending=true이면 보완 필요 안내

↓

[프론트] 사용자가 확인·수정 후 저장 요청

↓

[백엔드] 1.3 차량 등록 저장: 중복 검사 → Vehicle + VehicleRegistration 생성 → 상태 계산

↓

[프론트] 저장 결과 표시 (vehicleStatus: DRAFT/LISTABLE)

```

  

- 저장 시 필수값(차대번호, 차명, 차종, 연식, 색상, 최초등록일)이 모두 있으면 `LISTABLE`, 하나라도 없으면 `DRAFT` 상태.

- 주민등록번호는 서버에서 자동 마스킹 처리 (`900101-1******`).

- car365 API 미연동 기간에는 VIN 비교만 수행하며, OCR VIN을 기준값으로 사용하여 MATCH 처리.

  

---

  

## 1.1 차량번호 조회

  

`POST /vehicle/registration/lookup`

  

****Headers****: `Authorization: Bearer {accessToken}`

  

차량 번호를 입력하면 자동차365 API로 차량 기본 정보를 조회한다. 이미 등록된 차량이면 `alreadyRegistered: true` + 메시지를 반환한다.

  

> ****참고****: 현재 car365 API 미연동 상태. 플레이스홀더가 빈 값을 반환한다.

  


### Request (application/json)  
  
| 필드 | 타입 | 필수 | 설명 |  
|------|------|------|------|  
| licensePlate | String | O | 차량 번호 (예: "123가4567") |

  

****Response**** `200 OK`

  

```json

{

"licensePlate": "123가4567",

"vin": "KMHD341DBNU000001",

"model": "아반떼 CN7",

"brand": "현대",

"vehicleType": "승용",

"modelYear": "2024",

"fuelType": "가솔린",

"engineDisplacement": "1598",

"exteriorColor": "흰색",

"firstRegistrationDate": "2024-03-15",

"alreadyRegistered": false,

"message": "차량 정보 조회 성공"

}

```

  


| 필드 | 타입 | 설명 |  
|------|------|------|  
| licensePlate | String | 차량 번호 |  
| vin | String | 차대번호 |  
| model | String | 차명(모델명) |  
| brand | String | 브랜드(제조사) |  
| vehicleType | String | 차종 (승용, 화물 등) |  
| modelYear | String | 연식 |  
| fuelType | String | 연료 타입 |  
| engineDisplacement | String | 배기량(cc) |  
| exteriorColor | String | 색상 |  
| firstRegistrationDate | String | 최초등록일 (YYYY-MM-DD) |  
| alreadyRegistered | Boolean | 이미 등록된 차량 여부 |  
| message | String | 응답 메시지 |

  

****비즈니스 로직****

1. 내부 DB에서 해당 차량번호로 이미 등록된 차량이 있는지 확인 (`existsByLicensePlate`)

2. car365 API로 차량 기본 정보 조회 (현재 플레이스홀더 → 빈 값 반환)

3. 이미 등록된 차량이면 `alreadyRegistered=true` + 메시지를 응답에 포함 (에러가 아닌 정상 응답)

4. 미등록 차량이면 car365 조회 결과 그대로 반환

  

---

  

## 1.2 원부 OCR 파싱 + car365 비교

  

`POST /vehicle/registration/document-parse` (multipart/form-data)

  

****Headers****: `Authorization: Bearer {accessToken}`

  

차량 등록 원부 파일을 업로드하면 Upstage Document Digitization API로 OCR 파싱한다. `licensePlate`를 함께 전송하면 car365 조회 결과와 필드별 비교를 수행한다.

  

****Request**** (multipart/form-data)

  


| 필드 | 타입 | 필수 | 설명 |  
|------|------|------|------|  
| document | File | O | 차량 원부 파일 (PDF, JPG, JPEG, PNG) |  
| licensePlate | String | - | 차량 번호. 전송 시 car365 비교 수행 |
  

****Response**** `200 OK`

  

```json

{

"success": true,

"message": "파싱 완료",

"vin": "KMHD341DBNU000001",

"model": "아반떼 CN7",

"vehicleType": "승용",

"modelYear": "2024",

"exteriorColor": "흰색",

"firstRegistrationDate": "2024-03-15",

"serialNumber": "2024-01-00001",

"specificationNumber": "A1234567",

"cancellationDate": "",

"engineType": "G4NB",

"purpose": "자가용",

"sourceClassification": "국산",

"subType": "",

"manufactureDate": "2024-02-01",

"lastOwner": "홍길동",

"residentRegistrationNumber": "900101-*******",

"garageLocation": "서울특별시 강남구...",

"inspectionValidityPeriod": "2026-03-14",

"registrationConfirmationDate": "2026-01-30",

"closureDate": "",

"certificateIssueDate": "2026-01-30",

"approval": "대구광역시 ㅇㅇㅇ",

"ownershipHistories": [

{

"mainRegistration": "1",

"subRegistration": "",

"remarks": "소유권이전등록",

"residentRegistrationNumber": "900101-*******",

"registrationDate": "2024-03-15",

"receiptNumber": "2024-0001234"

}

],

"comparisons": [

{

"fieldName": "차대번호",

"fieldKey": "vin",

"apiValue": "KMHD341DBNU000001",

"ocrValue": "KMHD341DBNU000001",

"status": "MATCH",

"isMismatch": false,

"isOcrEmpty": false

}

],

"mismatchFields": [],

"ocrEmptyFields": [],

"mismatchCount": 0,

"ocrEmptyCount": 0,

"pending": false,

"pendingReasons": []

}

```

  

### OCR 파싱 필드

  


| 필드 | 타입 | 설명 |  
|------|------|------|  
| success | Boolean | 파싱 성공 여부 |  
| message | String | 결과 메시지 |  
| serialNumber | String | 일련번호 |  
| specificationNumber | String | 제원관리번호 |  
| cancellationDate | String | 말소등록일 |  
| model | String | 차명 |  
| vehicleType | String | 차종 |  
| vin | String | 차대번호 |  
| engineType | String | 원동기형식 |  
| purpose | String | 용도 |  
| modelYear | String | 연식(모델연도) |  
| exteriorColor | String | 색상 |  
| sourceClassification | String | 출처구분 |  
| firstRegistrationDate | String | 최초등록일 |  
| subType | String | 세부유형 |  
| manufactureDate | String | 제작연월일 |  
| lastOwner | String | 최종소유자 |  
| residentRegistrationNumber | String | 주민(법인)등록번호 (마스킹) |  
| garageLocation | String | 사용본거지(차고지) |  
| inspectionValidityPeriod | String | 검사유효기간 |  
| registrationConfirmationDate | String | 등록사항 확인일 |  
| closureDate | String | 폐쇄일 |  
| certificateIssueDate | String | 등본 발급일자 |  
| approval | String | 승인 |  
| ownershipHistories | Array | 순위번호 목록 |

  

### 순위번호 항목 (OwnershipHistoryItem)

  


| 필드 | 타입 | 설명 |  
|------|------|------|  
| mainRegistration | String | 주등록 |  
| subRegistration | String | 부기등록 |  
| remarks | String | 사항란 |  
| residentRegistrationNumber | String | 주민(법인)등록번호 (마스킹) |  
| registrationDate | String | 등록일자 |  
| receiptNumber | String | 접수번호 |

  

### 비교 결과

  


| 필드 | 타입 | 설명 |  
|------|------|------|  
| comparisons | Array | 필드별 비교 결과 전체 목록 |  
| mismatchFields | Array | 불일치 필드 목록 (isMismatch=true 전체, OCR 빈값 포함 가능) |  
| ocrEmptyFields | Array | OCR 빈 값 필드 목록 |  
| mismatchCount | Integer | 불일치 필드 수 |  
| ocrEmptyCount | Integer | OCR 인식 실패 필드 수 |  
| pending | Boolean | 보완 필요 여부 |  
| pendingReasons | Array | 보완 사유 목록 |

  

### FieldComparisonItem

  


| 필드 | 타입 | 설명 |  
|------|------|------|  
| fieldName | String | 필드명 (한글) |  
| fieldKey | String | 필드 키 (영문) |  
| apiValue | String | car365 API 조회 값 |  
| ocrValue | String | OCR 파싱 값 |  
| status | String | MATCH / DIFFERENT / OCR_EMPTY / API_EMPTY / BOTH_EMPTY |  
| isMismatch | Boolean | 불일치 여부 |  
| isOcrEmpty | Boolean | OCR 값 비어있음 여부 |

  

****비즈니스 로직****

1. 파일 유효성 검증: 빈 파일(F001), MIME 타입(F005: PDF/JPG/JPEG/PNG만 허용), 파일 크기(F002)

2. Upstage Document Digitization API로 OCR 파싱 → `VehicleDocParseResponseDto` 생성

3. `licensePlate`가 있으면 car365 API 재조회 → 공통 필드(차대번호, 차명, 차종, 연식, 색상, 최초등록일) 비교

4. 비교 결과를 `FieldComparisonItem` 목록으로 구성 (MATCH/DIFFERENT/OCR_EMPTY/API_EMPTY)

5. OCR 빈값 필드 → `pendingReasons`에 "필드명 OCR 인식값 없음" 추가

6. 불일치 필드 → `pendingReasons`에 "필드명 car365 불일치" 추가

7. `pending = pendingReasons가 있으면 true`

  

> ****car365 미연동 기간****: API가 빈 값을 반환하므로 모든 비교 필드가 `API_EMPTY` 상태. 단, VIN은 OCR 값을 기준값으로 대입하여 `MATCH` 처리.

  

****에러****

  

- F001: 빈 파일

- F005: 허용되지 않은 파일 형식 (PDF, JPG, JPEG, PNG만 허용)

- F002: 파일 크기 초과

- VR003: 원부 문서 파싱 실패

- VR005: 외부 API 호출 실패

  

---

  

## 1.3 차량 등록 저장

  

`POST /vehicle/registration`

  

****Headers****: `Authorization: Bearer {accessToken}`

  

차량번호 조회 + 원부 OCR 결과를 확인/수정한 뒤 최종 저장한다.

  

****Request**** (application/json)

  


| 필드 | 타입 | 필수 | 설명 |  
|------|------|------|------|  
| licensePlate | String | O | 차량 번호 |  
| vin | String | O | 차대번호 |  
| serialNumber | String | - | 일련번호 |  
| specificationNumber | String | - | 제원관리번호 |  
| cancellationDate | String | - | 말소등록일 |  
| model | String | - | 차명 |  
| vehicleType | String | - | 차종 |  
| engineType | String | - | 원동기형식 |  
| purpose | String | - | 용도 |  
| modelYear | String | - | 연식 |  
| exteriorColor | String | - | 색상 |  
| sourceClassification | String | - | 출처구분 |  
| firstRegistrationDate | String | - | 최초등록일 |  
| subType | String | - | 세부유형 |  
| manufactureDate | String | - | 제작연월일 |  
| lastOwner | String | - | 최종소유자 |  
| residentRegistrationNumber | String | - | 주민(법인)등록번호 |  
| garageLocation | String | - | 사용본거지 |  
| inspectionValidityPeriod | String | - | 검사유효기간 |  
| registrationConfirmationDate | String | - | 등록사항 확인일 |  
| closureDate | String | - | 폐쇄일 |  
| certificateIssueDate | String | - | 등본 발급일자 |  
| approval | String | - | 승인 |  
| ownershipHistories | Array | - | 순위번호 목록 |

  

****Response**** `200 OK`

  

```json

{

"vehicleId": 1,

"licensePlate": "123가4567",

"vehicleStatus": "DRAFT",

"pending": true,

"serialNumber": "2024-01-00001",

"vin": "KMHD341DBNU000001",

"model": "아반떼 CN7",

"vehicleType": "승용",

"modelYear": "2024",

"exteriorColor": "흰색",

"firstRegistrationDate": "2024-03-15",

"ownershipHistories": [],

"message": "차량 등록 정보"

}

```

  


  
| 필드 | 타입 | 설명 |  
|------|------|------|  
| vehicleId | Long | 차량 ID |  
| licensePlate | String | 차량 번호 |  
| vehicleStatus | String | 차량 상태: `DRAFT` / `LISTABLE` |  
| pending | Boolean | 보완 필요 여부 (DRAFT이면 true) |  
| documentUrl | String | 원부 파일 URL |  
| message | String | 결과 메시지 |  
| (원부 필드) | - | 1.2 OCR 파싱 필드와 동일 |

  

****비즈니스 로직****

1. JWT에서 memberId 추출 → SellerProfile 조회 (S004)

2. VIN 정규화: null/빈값이면 C001 에러

3. 중복 검사: licensePlate 중복(V002), VIN 중복(V003)

4. Vehicle 엔티티 생성 (licensePlate, VIN, countryCode="KR")

5. VehicleRegistration 엔티티 생성 (원부 전체 필드)

6. 주민등록번호 자동 마스킹 (본인 + 순위번호 항목)

7. 필수값 기반 상태 계산: 모두 있으면 `LISTABLE`, 하나라도 없으면 `DRAFT`

8. DataIntegrityViolationException 발생 시 중복 재확인 (race condition 대응)

  

****에러****

  

- AUTH004: 유효하지 않은 토큰

- S004: 판매자 프로필 없음

- V002: 차량 번호 중복

- V003: 차대번호 중복

- C001: 잘못된 입력값 (차대번호 비어있음)

  

---

  

## 1.4 차량 원부 수정

  

`PATCH /vehicle/registration/{vehicleId}`

  

****Headers****: `Authorization: Bearer {accessToken}`

  

저장된 차량 원부 정보를 수정한다. ****null 필드는 무시****된다 (부분 수정).

  

****Path Parameter****

  


| 파라미터 | 타입 | 설명 |  
|----------|------|------|  
| vehicleId | Long | 차량 ID |

  

****Request**** (application/json)

  

1.3과 동일한 필드 중 `licensePlate`를 제외한 항목을 수정할 수 있다. `vin`은 선택적으로 수정 가능하며, null인 필드는 수정하지 않는다.

  

****Response**** `200 OK`

  

1.3 Response와 동일.

  

****비즈니스 로직****

1. JWT에서 memberId 추출 → Vehicle 조회 (V001) → 소유자 검증 (C005)

2. VehicleRegistration 조회 (VR001)

3. VIN 변경 시: 빈 문자열이면 C001, 다른 차량과 중복이면 V003

4. null이 아닌 필드만 업데이트 (부분 수정)

5. 순위번호가 전달되면 전체 교체

6. 주민등록번호 변경 시 자동 마스킹

7. 필수값 기반 상태 재계산 (DRAFT ↔ LISTABLE)

  

****에러****

  

- AUTH004: 유효하지 않은 토큰

- V001: 차량 없음

- C005: 접근 거부 (소유자 불일치)

- VR001: 차량 원부 정보 없음

- V003: 차대번호 중복 (변경 시)

- C001: 잘못된 입력값 (빈 문자열 차대번호)

  

---

  

## 1.5 차량 등록 정보 조회

  

`GET /vehicle/registration/{vehicleId}`

  

****Headers****: `Authorization: Bearer {accessToken}`

  

저장된 차량 등록 정보(원부 데이터 포함)를 조회한다.

  

****Path Parameter****

  


| 파라미터 | 타입 | 설명 |  
|----------|------|------|  
| vehicleId | Long | 차량 ID |

  

****Response**** `200 OK`

  

1.3 Response와 동일.

  

****비즈니스 로직****

1. JWT에서 memberId 추출 → Vehicle 조회 (V001) → 소유자 검증 (C005)

2. VehicleRegistration 조회 (VR001) → 응답 반환

  

****에러****

  

- AUTH004: 유효하지 않은 토큰

- V001: 차량 없음

- C005: 접근 거부 (소유자 불일치)

- VR001: 차량 원부 정보 없음
