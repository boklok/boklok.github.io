---
layout: post
title: "M365 실습기 - Exchange만 껐는데 사서함이 2GB로 남았다 (서비스 플랜 의존성과 겹치는 플랜)"
date: 2026-10-19 12:00:00 +0900
categories: [M365, EntraID, Azure]
tags: [m365, license, service-plan, exchange-online, kiosk, myanalytics, mfa, sign-in-log, azure-budget, powershell, microsoft-graph, readiness]
---

# M365 실습기 - 서비스 플랜 의존성과 겹치는 플랜

운영 Readiness 사전 프로그램이 실제 수행계획에 맞춰 개편되면서, 이번 주부터 1주차를 공식 일정으로 다시 진행한다. 지난주 글은 [보안 기본값·비상 계정]({% post_url 2026-10-16-security-defaults-breakglass-group-license-root-cause %})까지였다.

오늘(D1)은 세 가지를 했다.

1. **실습 환경 재확인**: MFA가 "등록됐다"가 아니라 "실제로 걸렸다"를 로그로 확인하고, 비어 있던 비용 예산을 만들었다.
2. **사용자 5명 생성과 라이선스 할당**: 할당 직후에는 상태가 비어 있었다.
3. **서비스 플랜 하나 끄기**: Exchange만 끄려다 두 번 거부당했고, 끄고 나서도 사서함이 사라지지 않았다.

세 번째가 오늘의 핵심이다. 라이선스를 "제품 하나"로 생각하면 운영에서 바로 틀린다는 걸 확인했다.

> 테넌트 도메인 같은 식별값은 `xxxx`로 가렸다. 사용자는 실습용으로 만든 계정이다.

---

## 1. MFA는 "등록"이 아니라 "실제로 요구됐는지"로 본다

MFA 등록 여부는 쉽게 나온다.

```powershell
Get-MgReportAuthenticationMethodUserRegistrationDetail -Filter "userPrincipalName eq '$me'"
# IsMfaRegistered : True
# 등록방법        : passKeySynced, microsoftAuthenticatorPush, softwareOneTimePasscode
```

하지만 "등록돼 있다"와 "로그인할 때 실제로 MFA를 요구받았다"는 다르다. 그래서 로그인 기록의 **인증 요구(authenticationRequirement)**를 봤다. 여기서 두 번 막혔다.

- Graph **v1.0** 로그인 기록에는 이 값이 **비어 있다**. **beta**로 조회해야 나온다.
- beta 로그인 기록에 `$select`를 붙이면 `400 Unsupported Query`가 난다. `$filter`와 `$top`만 쓴다.

```powershell
(Invoke-MgGraphRequest -Method GET -Uri "beta/auditLogs/signIns?`$filter=userPrincipalName eq '$me'&`$top=5").value |
  ForEach-Object { "{0} | {1} | {2} | {3}" -f ([datetime]$_.createdDateTime).ToLocalTime(), $_.appDisplayName,
    $_.authenticationRequirement, (($_.authenticationDetails | ForEach-Object { $_.authenticationMethod }) -join ' + ') }
```

```
2026-10-19 08:50:26 | Azure Portal               | multiFactorAuthentication | Previously satisfied
2026-10-19 08:45:51 | Microsoft Azure PowerShell | multiFactorAuthentication | Previously satisfied
```

두 로그인 모두 **MFA가 요구됐다.** `Previously satisfied`는 같은 세션에서 앞서 받은 MFA로 이번 요구를 통과했다는 뜻이다. 로그인 기록은 몇 분 늦게 쌓이므로 방금 한 로그인은 바로 안 보일 수 있다.

---

## 2. 비용 예산: API는 422, 포털은 정상

이 Sandbox 구독은 지출 한도(SpendingLimit)가 **Off**다. 크레딧이 떨어져도 자동으로 멈추지 않는다는 뜻이라, 예산 경보가 유일한 안전장치다.

```powershell
Get-AzConsumptionBudget
# Operation returned an invalid status code '422'
```

API로는 조회가 안 됐다. 그런데 포털(구독 › 비용 관리 › 예산)에서는 "예산이 없습니다"가 정상으로 표시됐다. **예산이 아예 없었던 것**이다. 422가 나는 이유는 아직 모른다.

포털에서 바로 만들었다. 경고 조건을 넣다가 한 번 막혔다.

> 작업 그룹 또는 하나 이상의 이메일 주소가 모든 예산 임계값에 필요합니다.

경고를 받을 곳이 없으면 저장되지 않는다. 작업 그룹을 만들 필요는 없고 **이메일만 넣으면** 된다. 월 3만 원, 실제 비용 50%·100%, 예측 비용 50%로 설정했다.

---

## 3. 사용자 5명에게 라이선스: 직후 재조회는 비어 있었다

```powershell
Set-MgUserLicense -UserId $upn -AddLicenses @(@{SkuId = $sku.SkuId}) -RemoveLicenses @()
```

할당 직후 각 사용자의 라이선스 상태를 다시 조회하니 **5명 모두 공란**이었다. 그런데 좌석 수는 이미 18 → 22로 늘어 있었다. 약 2분 뒤 다시 조회하니 이렇게 나왔다.

```
d1-user01    State=Active   AssignedByGroup=(없음=직접)
...
d1-user05    State=Active   AssignedByGroup=(없음=직접)
[좌석] 23 / 25
```

좌석이 먼저 바뀌고, 사용자별 상태는 나중에 채워진다. 지난주부터 반복하는 원칙 그대로다. **명령 성공은 적용 완료가 아니다. 기다렸다가 다시 조회하고, 확인된 것만 기록한다.**

사서함은 할당하고 약 7분 안에 만들어져 있었다. 관리 센터 메일 탭에 `0% (8KB / 100GB)`로 표시됐다.

---

## 4. Exchange만 끄기: 두 번 거부당하다

가이드의 D1 과제는 "서비스 플랜 하나를 끄고 사용자 화면 변화 확인"이다. 가장 눈에 띄는 Exchange를 골랐다.

### 첫 번째 함정: 비슷한 이름의 플랜이 3개

E5 SKU 안에 Exchange 계열 플랜이 하나가 아니었다.

```
ServicePlanName       ServicePlanId
EXCHANGE_ANALYTICS    34c0d7a0-…    ← MyAnalytics
EXCHANGE_S_DESKLESS   4a82b400-…    ← Exchange Online Kiosk
EXCHANGE_S_ENTERPRISE efb87545-…    ← Exchange Online (Plan 2), 사서함
```

처음 스크립트는 이름이 `EXCHANGE_S_`로 시작하는 첫 번째 플랜을 고르게 짜여 있어서 Kiosk가 잡혔다. 플랜은 **정확한 이름으로** 지정해야 한다.

### 두 번째 함정: 의존 관계

`EXCHANGE_S_ENTERPRISE`만 끄는 요청은 거부됐다.

```
Set-MgUserLicense : Cannot process request because a service plan depends on another service plan.
Status: 400 (BadRequest)
```

"뭔가가 의존한다"고만 알려 줬다. 같은 작업을 관리 센터에서 해 보니 **무엇이 무엇에 의존하는지**까지 알려 줬다.

> To assign a license that contains **Microsoft MyAnalytics(전체)**, you must also assign one of the following service plans: **Exchange Online(계획 2)**.

재미있는 건 관리 센터 앱 목록에서 MyAnalytics(전체)가 **"사용하지 않음, 조직 수준에서 할당"**으로 표시돼 있었다는 점이다. 화면에서는 이미 안 쓰는 앱처럼 보이는데, 그 앱이 Exchange 해제를 막고 있었다.

두 플랜을 **함께** 끄니 바로 적용됐다.

```powershell
Set-MgUserLicense -UserId $Target -AddLicenses @(@{
  SkuId = $sku.SkuId; DisabledPlans = @($ex.ServicePlanId, $exa.ServicePlanId) }) -RemoveLicenses @()
# [재조회] Disabled: EXCHANGE_ANALYTICS, EXCHANGE_S_ENTERPRISE
```

> 같은 실패라도 경로마다 알려 주는 정보가 다르다. PowerShell은 "의존 관계가 있다"까지, 포털은 "누가 누구에게 의존하는지"까지 알려 줬다.

---

## 5. 껐는데 사서함이 남았다: 겹치는 플랜

끈 사용자(user03)와 그대로 둔 사용자(user02)를 비교했다.

```
d1-user03(끔)      Exchange 할당 계획 상태: Deleted
d1-user02(그대로)   Exchange 할당 계획 상태: Enabled
```

여기까지는 예상대로다. 그런데 관리 센터 메일 탭을 보니 사서함이 **사라지지 않았다.**

| 항목 | 끄기 전 | 끈 뒤 (약 3~4분) |
|---|---|---|
| 사서함 저장소 | 0% (8KB / **100GB**) | 0% (13KB / **2GB**) |
| 기타 작업 | 공유 사서함 변환 · Exchange 속성 · **소송 보존 관리** | 소송 보존 관리 **사라짐** |
| Graph Exchange 할당 상태 | Enabled | Deleted |

원인은 같은 SKU에 남아 있는 **Exchange Online Kiosk** 플랜이다. Plan 2를 꺼도 Kiosk가 켜져 있으니, 사서함이 Kiosk 수준(2GB)으로 내려간 채 유지됐다. 관리 센터 사용자 목록의 "라이선스" 열은 두 사용자 모두 똑같이 E5로 보인다. **라이선스 열만 봐서는 실제 기능 상태를 알 수 없다.**

소송 보존 메뉴가 사라진 게 Plan 2 전용 기능이기 때문인지는 공식 문서로 아직 확인하지 않았다.

---

## 운영에서 쓸 판독 순서

오늘 결과를 운영 문의 처리 순서로 정리했다.

**"메일이 안 돼요 / 라이선스가 안 붙었어요"**
1. 사용자 `licenseAssignmentStates`: 할당 경로(직접/그룹)와 상태. 할당 직후면 1~2분 뒤 다시 본다.
2. `Get-MgUserLicenseDetail`: 해당 서비스 플랜이 Success인지 Disabled인지 본다.
3. `assignedPlans`: 서비스별 Enabled / Deleted를 본다.
4. 관리 센터 메일 탭: 사서함이 있는지, 할당량이 얼마인지 본다.

**"이 기능만 꺼 주세요"**
1. 정확한 플랜 이름을 확인한다(비슷한 이름 주의).
2. **의존하는 플랜**이 있는지 확인한다. 모르면 포털에서 해제를 시도해 오류 문구로 확인한다.
3. 같은 기능을 주는 **겹치는 플랜**이 있는지 확인한다. 남아 있으면 기능이 낮은 수준으로 유지된다.
4. 변경 전 값 저장 → 변경 → 재조회 → 화면 비교. 원복 방법도 함께 기록한다.

---

## 정리

1. **라이선스는 서비스 플랜 묶음이다.** 같은 SKU 안에 비슷한 이름의 플랜이 여러 개 있다.
2. **플랜끼리 의존 관계가 있다.** 하나만 끄려고 하면 거부될 수 있다.
3. **겹치는 플랜이 있으면 꺼도 기능이 남는다.** 사라지는 게 아니라 수준만 낮아진다.
4. **MFA는 로그인 기록의 인증 요구로 확인한다.** v1.0이 아니라 beta에 값이 있다.
5. **명령 성공 ≠ 적용 완료.** 할당 직후 공란은 기다렸다가 다시 본다.

이번 주부터는 변경을 기록하는 항목도 늘었다. 언제·무엇·전후 값·누가·왜에 **근거·검증 결과·원복 방법**을 더한 8항목이다. 오늘 바꾼 12건도 모두 이 형식으로 남겼다.

내일은 하이브리드 인증 구조를 공부하고, 지난주 원인을 좁혀 둔 Dynamic 그룹 라이선스 문제를 새 그룹으로 다시 확인해 본다.
