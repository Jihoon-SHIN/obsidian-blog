## 개요

이 문서는 `ensureMongooseConnection.ts`의 MongoDB 연결 설정에 대한 설명입니다. 각 `connectionMode`별로 어떤 노드에 연결되는지, 설정배경에 대해 설명합니다.

---

## 클러스터 노드 구성 (Prod)

|ID|nodeType|region|workloadType|역할|
|---|---|---|---|---|
|0|ELECTABLE|AP_NORTHEAST_2|OPERATIONAL|한국 Secondary|
|1|ELECTABLE|AP_NORTHEAST_2|OPERATIONAL|한국 Primary|
|2|ELECTABLE|AP_NORTHEAST_2|OPERATIONAL|한국 Secondary|
|3|ANALYTICS|AP_NORTHEAST_2|(없음)|한국 분석 노드|
|4|READ_ONLY|US_EAST_1|OPERATIONAL|미국 읽기 전용|

- fyi. ELECTABLE 노드들의 경우 화요일 정기점검 영향으로, primary 노드가 변경됩니다
- fyi. workloadType 의 OPERATIONAL 은 “운영용” 입니다. 분석용은 운영용이 아니기에 workloadType 이 없습니다.

### 노드 타입(nodeType) 설명

- **ELECTABLE**: Primary로 선출될 수 있는 노드. `votes: 1`, `priority > 0`. Primary가 죽으면 다른 ELECTABLE이 Primary가 됨.
- **READ_ONLY**: 읽기 전용 복제본. `votes: 0`, `priority: 0`. 선출 불가, 읽기만 가능.
- **ANALYTICS**: 분석 전용 노드. `votes: 0`, `priority: 0`. 무거운 분석 쿼리 격리용.

### Primary 선출 (Election)

MongoDB replica set은 **자동 장애 복구**를 위해 선출 메커니즘을 사용합니다.

**선출 과정**:

1. Primary가 응답하지 않으면 (heartbeat 실패)
2. ELECTABLE 노드들이 투표를 진행
3. 과반수 득표한 노드가 새 Primary가 됨

**선출 관련 속성**:

- `votes`: 투표권. 1이면 투표 가능, 0이면 투표 불가
- `priority`: 선출 우선순위. 높을수록 Primary 될 확률 높음. 0이면 Primary 불가

|노드 타입|votes|priority|Primary 가능|
|---|---|---|---|
|ELECTABLE|1|1 이상|O|
|READ_ONLY|0|0|X|
|ANALYTICS|0|0|X|

READ_ONLY와 ANALYTICS는 선출에 참여하지 않으므로 Primary가 될 수 없습니다.

---

## Read Preference란?

MongoDB replica set에서 **읽기 작업을 어떤 노드로 보낼지 결정하는 설정**입니다.

- 기본적으로 모든 읽기/쓰기는 Primary로 전송됨
- Read Preference를 설정하면 읽기 작업을 Secondary나 다른 노드로 분산 가능
- 쓰기는 항상 **Primary**로만 전송됨 (Read Preference 영향 없음)

### 사용 목적

- **Primary 부하 분산**: 읽기 트래픽을 Secondary로 분산
- **지역 최적화**: 가까운 노드에서 읽어서 latency 감소
- **워크로드 격리**: 분석 쿼리를 별도 노드에서 실행

---

## Read Preference 종류

|readPreference|동작|Secondary 없을 때|
|---|---|---|
|`primary`|Primary만|-|
|`secondary`|Secondary만 (ELECTABLE)|에러|
|`primaryPreferred`|Primary 우선|Secondary로 fallback|
|`secondaryPreferred`|Secondary 우선|Primary로 fallback|
|`nearest`|가장 가까운 노드 (모든 타입 포함)|아무거나|

### Read Preference가 선택할 수 있는 노드

각 readPreference 설정이 어떤 노드 타입을 선택할 수 있는지 정리합니다.

**`primary`**

- Primary만 선택
- Secondary, READ_ONLY, ANALYTICS 모두 선택 불가

> "All read operations use only the current replica set primary. This is the default read mode. If the primary is unavailable, read operations produce an error or throw an exception."

**`secondary`**

- ELECTABLE Secondary만 선택 (선출에 참여하는 노드 중 Primary가 아닌 것)
- READ_ONLY는 선출에 참여하지 않으므로 선택 불가
- ANALYTICS는 태그를 명시하면 선택 가능

> "Operations read only from the secondary members of the set. If no secondaries are available, then this read operation produces an error or exception."

**`secondaryPreferred`**

- ELECTABLE Secondary 우선 선택
- Secondary가 없으면 Primary로 fallback
- READ_ONLY는 선택 불가

> "Operations typically read data from secondary members of the replica set. If the replica set has only one single primary member and no other members, operations read data from the primary member."

**`nearest`**

- 네트워크 latency가 가장 낮은 노드 선택
- Primary, Secondary, READ_ONLY, ANALYTICS 모두 선택 가능
- READ_ONLY 노드를 선택할 수 있는 유일한 방법

> "The driver reads from a member whose network latency falls within the acceptable latency window. Reads in the nearest mode do not consider whether a member is a primary or secondary when routing read operations: primaries and secondaries are treated equivalently."

**출처**: [https://www.mongodb.com/docs/manual/core/read-preference/](https://www.mongodb.com/docs/manual/core/read-preference/)

**핵심**: READ_ONLY 노드는 선출에 참여하지 않기 때문에 `secondary`/`secondaryPreferred`로는 선택되지 않습니다. READ_ONLY 노드로 쿼리를 보내려면 반드시 `nearest`를 사용해야 합니다.

---

## Read Preference Tags

### Atlas 사전 정의 태그 (자동 설정, 변경 불가)

|태그|설명|예시|
|---|---|---|
|`nodeType`|노드 유형|`ELECTABLE`, `READ_ONLY`, `ANALYTICS`|
|`workloadType`|워크로드 유형|`OPERATIONAL` (ANALYTICS 제외)|
|`region`|클라우드 리전|`AP_NORTHEAST_2`, `US_EAST_1`|
|`provider`|클라우드 제공자|`AWS`, `GCP`, `AZURE`|
|`availabilityZone`|가용영역|`apne2-az1`, `use1-az6`|

### 태그 배열 동작 방식

MongoDB는 `readPreferenceTags` 배열을 **순서대로** 시도합니다:

```tsx
readPreferenceTags: [
  { region: "AP_NORTHEAST_2", workloadType: "OPERATIONAL" }, // 1순위
  { workloadType: "OPERATIONAL" }, // 2순위 (fallback)
]

```

1. 첫 번째 조건 매칭 시도
2. 실패하면 두 번째 조건으로 fallback
3. 모두 실패하면 에러

**출처**: [https://www.mongodb.com/docs/manual/core/read-preference-tags/](https://www.mongodb.com/docs/manual/core/read-preference-tags/)

---

## Connection Mode 설정

### 1. default

```tsx
.with("default", () => ({
  readPreference: ReadPreferenceMode.primary,
}))

```

- **용도**: 쓰기 작업, 실시간 데이터가 필요한 읽기
- **대상 노드**: Primary만
- **태그**: 없음 (primary는 태그 사용 불가)

### 2. readonly

```tsx
.with("readonly", () => {
  const readPreferenceTags: TagSet[] = [
    {
      region: REGION === "us-east-1" ? "US_EAST_1" : "AP_NORTHEAST_2",
      workloadType: "OPERATIONAL",
    },
    { workloadType: "OPERATIONAL" },
  ];
  return {
    readPreference:
      REGION === "us-east-1"
        ? ReadPreferenceMode.nearest
        : ReadPreferenceMode.secondaryPreferred,
    readPreferenceTags,
  };
})

```

- **용도**: 일반 읽기 작업
- **리전별 동작**:

|Lambda 리전|readPreference|대상 노드|
|---|---|---|
|ap-northeast-2 (한국)|secondaryPreferred|한국 ELECTABLE Secondary|
|us-east-1 (미국)|nearest|미국 READ_ONLY|

- **한국이 `secondaryPreferred`인 이유**:
    - READ_ONLY 노드가 한국에 없음
    - `nearest`를 쓰면 Primary도 선택될 수 있음
    - Secondary 우선, 없으면 Primary fallback으로 안정성 확보
- **미국이 `nearest`인 이유**:
    - 미국에는 READ_ONLY 노드만 있음
    - `secondary`/`secondaryPreferred`는 READ_ONLY 선택 불가
    - `nearest`만 READ_ONLY 노드 선택 가능

### 3. analytics

```tsx
.with("analytics", () => ({
  readPreference: ReadPreferenceMode.secondary,
  readPreferenceTags: [{ nodeType: "ANALYTICS" }],
}))

```

- **용도**: 무거운 분석 쿼리, ETL, 보고서
- **대상 노드**: ANALYTICS 노드만
- **워크로드 격리**: 분석 쿼리가 운영 DB에 영향 주지 않도록 분리

---

## 제약 사항

### Primary 제외 불가

MongoDB 태그만으로는 "Secondary + READ_ONLY만, Primary 제외"가 불가능합니다.

- Primary와 Secondary 모두 `nodeType: ELECTABLE`
- 태그로 구분 불가
- `secondaryPreferred`가 Primary를 최대한 피하는 최선의 방법

### READ_ONLY 노드 선택 조건

READ_ONLY 노드로 쿼리를 보내려면 반드시 태그를 사용해야 합니다.

> "Read-only nodes don't provide high availability because they don't participate in elections. They can't become the primary for their cluster. To direct queries to read-only nodes, use pre-defined replica set tags."
> 
> - - [https://www.mongodb.com/docs/atlas/cluster-config/multi-cloud-distribution/](https://www.mongodb.com/docs/atlas/cluster-config/multi-cloud-distribution/)

---

## 환경 변수

|변수|설명|값|
|---|---|---|
|`REGION`|Lambda 배포 리전|`ap-northeast-2` (기본), `us-east-1`|
|`STAGE`|환경|`prod`, `stg`, `dev`|

`REGION`은 `packages/server-common/src/constants/ENVIRONMENT.ts`에서 정의됩니다.

---

## 참고 문서

- [MongoDB Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/)
- [MongoDB Read Preference Tags](https://www.mongodb.com/docs/manual/core/read-preference-tags/)
- [Atlas Multi-Cloud Distribution](https://www.mongodb.com/docs/atlas/cluster-config/multi-cloud-distribution/)
- [Atlas Replica Set Tags](https://www.mongodb.com/ko-kr/docs/atlas/reference/replica-set-tags/)