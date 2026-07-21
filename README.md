# Claude-agent-IaC

# 🤖 Claude Agent IaC - KT Cloud VM Skill

> **Claude Code 기반 KT Cloud VM 스킬 구조 및 토큰 절약 분석**
> KT Cloud VM 생성 및 조회 작업을 효율적으로 수행하기 위한 Claude Agent 스킬 구성과, 스킬 적용을 통한 토큰 절약 효과를 정리했습니다.

---

## 📂 스킬 디렉토리 구조

kt-cloud-vm/
├── SKILL.md                 # 워크플로우 및 제약 규칙 (VM 생성/조회 전용)
└── references/
    ├── flavors.md            # 플레이버 ID 목록 (vCPU / RAM / UUID)
    ├── networks.md           # 티어명 + osnetworkid 매핑 (19개)
    ├── images.md             # OS 이미지 ID (Windows / Linux)
    └── keypairs.md           # 키페어 목록

### 📄 각 파일의 역할

| 파일명 | 역할 | 주요 내용 |
| :--- | :--- | :--- |
| **SKILL.md** | 스킬 진입점 | 인증 방식, API 엔드포인트, VM 생성 워크플로우, 안전 규칙 |
| **references/flavors.md** | 플레이버 레퍼런스 | 전체 플레이버 목록(vCPU/RAM/UUID) 및 자주 쓰는 사양 빠른 참조 |
| **references/networks.md** | 네트워크 레퍼런스 | 19개 전체 티어 정보(티어명/CIDR/osnetworkid) |
| **references/images.md** | 이미지 레퍼런스 | Windows/Linux OS 이미지 전체 (이름/UUID) |
| **references/keypairs.md** | 키페어 레퍼런스 | 키페어 목록 및 SSH 접속 방법 안내 |

---

## ⚙️ SKILL.md 핵심 규칙

### ⚠️ 작업 제한사항
* **허용 작업:** VM 생성(`POST /servers`), VM 조회(`GET`)
* **절대 금지:** VM 삭제, 수정, 볼륨/네트워크/보안그룹/키페어 삭제 및 수정 등 일체의 변경 작업
* **예외 처리:** 변경·삭제 작업은 **사용자의 명시적 재확인**이 있을 때만 예외적으로 실행

---

### 🚦 오류 처리 및 원칙
애매한 상황이 발생하거나 오류가 나면 절대 임의 판단하지 않고 아래 순서로 원인을 파악합니다.

1. **API 가이드 문서 확인:** [KT Cloud Open API 가이드](https://cloud.kt.com/docs/open-api-guide/d/guide/)에서 필수 파라미터, 타입, 제약사항을 우선 조회
2. **파라미터 검증:** 요청 Body와 가이드를 비교하여 누락, 타입 오류, 잘못된 값 유무 검증
3. **리소스 상태 조회(GET):** 이미지, 플레이버, 네트워크 등 참조 리소스의 유효성을 개별 확인
4. **원인 보고:** 원인 파악 후 사용자에게 보고하고 작업을 즉시 중지

#### 💡 트러블슈팅 사례 (과거 발생 오류)
| 오류 현상 | 발생 원인 | 해결 방법 |
| :--- | :--- | :--- |
| **VM 생성 500** | `availability_zone` 누락 | `"availability_zone": "DX-G-YS"` 항목 추가 |
| **VM 생성 500** | `boot_index` 타입 오류 | 정수 `0` -> 문자열 `"0"`으로 수정 |
| **이미지 404** | API 경로 중복 (`/v2/v2/`) | `/gd4/image/images` 경로 사용 (`v2` 제거) |

---

### 🛡️ 안전 및 후속 처리 절차

#### ✅ VM 생성 2단계 승인 프로세스
* **1차 확인:** VM 이름, 티어, OS, 사양, 스토리지, 키페어 정돈 후 `"진행할까요?"` 확인
* **2차 확인:** `"⚠️ 실제 클라우드 리소스를 생성합니다. 정말 생성하시겠습니까?"` 최종 승인 요청

#### 🔑 Windows VM 패스워드 복호화
VM 생성 응답의 `adminPass`는 실제 패스워드와 다를 수 있으므로, 반드시 저장된 키페어로 복호화하여 전달합니다.

1. 암호화된 패스워드 조회:
   `curl -s -H "X-Auth-Token: $TOKEN" "[https://api.ucloudbiz.olleh.com/gd4/server/servers/](https://api.ucloudbiz.olleh.com/gd4/server/servers/){VM_ID}/os-server-password" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('password',''))"`

2. 개인키로 복호화:
   `echo "$ENCRYPTED" | base64 -d | openssl rsautl -decrypt -inkey /home/cloud-user/dev-kice-key.pem -pkcs`

> 💡 복호화 실패 시 VM이 아직 부팅 중일 수 있으므로 **30초 후 재시도**합니다. (기본 Windows 계정: `Administrator`)

#### 📋 Confluence 이력 자동 기록
VM 생성이 성공하면 아래 Confluence 페이지 테이블의 마지막 행에 명세를 자동 기록합니다.
* **대상 페이지:** `[KICE] AI기반 학력진단시스템 - KT Cloud VM 생성 이력`
* **기록 컬럼:** `생성일` / `VM 이름` / `VM ID` / `티어` / `IP` / `OS` / `사양` / `키페어`
* **처리 순서:** `VM 생성 성공` -> `GET /servers/{id} (IP 조회)` -> `getConfluencePage` -> `테이블 행 추가` -> `updateConfluencePage`

---

## 📊 토큰 사용량 비교 및 분석

### 1. 방식별 토큰 소비량 예측

#### ❌ 스킬 미사용 (API 직접 조회 방식)
| 작업 | 예상 토큰 |
| :--- | :--- |
| 플레이버 목록 API 조회 + JSON 파싱 | ~3,000 |
| 네트워크 티어 API 조회 + 파싱 | ~800 |
| 이미지 목록 API 조회 + 파싱 | ~1,500 |
| 키페어 조회 | ~300 |
| API 오류 / 재시도 반복 처리 | ~3,000 ~ 5,000 |
| **소계** | **~8,000 ~ 10,500 tokens** |

#### ✅ 스킬 사용 (레퍼런스 파일 참조 방식)
| 작업 | 예상 토큰 |
| :--- | :--- |
| `SKILL.md` 로드 | ~1,200 |
| `references/flavors.md` | ~900 |
| `references/networks.md` | ~400 |
| `references/images.md` | ~700 |
| `references/keypairs.md` | ~100 |
| **소계** | **~3,300 tokens** |

---

### 2. 결과 요약

> 💡 **스킬 도입을 통해 세션당 평균 50~65%의 토큰을 절약했습니다.**

| 항목 | 수치 |
| :--- | :--- |
| **스킬 미사용** | ~8,000 ~ 10,500 tokens/세션 |
| **스킬 사용 시** | ~3,300 tokens/세션 |
| **절약 토큰** | **약 5,000 ~ 7,000 tokens** |
| **절약률** | **약 50 ~ 65% 절감 📉** |

#### 🎯 주요 절약 포인트
1. **API 조회 제거 (절약분의 ~70%):** API 요청 및 큰 JSON 응답 파싱 대신 로컬 레퍼런스 파일 즉시 참조
2. **오류 및 재시도 방지:** 엔드포인트/파라미터 탐색 과정에서의 404, 500 반복 호출 차단
3. **대화 왕복 최소화:** 2단계 승인 및 작업 절차가 정의되어 불필요한 대화 단계를 축소

---

## 💡 스킬 트리거 예시

아래와 같은 사용자 자연어 요청 시 `kt-cloud-vm` 스킬이 자동으로 활성화됩니다.

* *"KT Cloud에 VM 만들어줘"*
* *"dev-DMZ에 윈도우 서버 생성해줘"*
* *"플레이버 목록 알려줘"*
* *"KT 클라우드 이미지 목록 보여줘"*
* *"네트워크 티어 조회해줘"*
* *"VM 목록 보여줘"*

---
_최초 작성일: 2026-04-29 | 최종 수정일: 2026-04-29_
