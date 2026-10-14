---
layout: post
title: "M365 Sandbox 실습기 - Dynamic 그룹, 그룹 기반 라이선스, Teams 채널은 어디에 저장될까"
date: 2026-10-14 18:00:00 +0900
categories: [M365, EntraID, Teams]
tags: [m365, entra-id, dynamic-group, group-based-licensing, teams, sharepoint, microsoft-graph, powershell, audit-log]
---

# M365 Sandbox 실습기 - Dynamic 그룹 · 그룹 기반 라이선스 · Teams 저장 구조

운영 Readiness 사전 프로그램 3주 과정의 사흘째. 어제 [실습환경 점검]({% post_url 2026-10-13-m365-azure-sandbox-readiness %})에 이어 오늘은 개인 Sandbox에서 세 가지를 직접 바꾸고 재 봤다.

1. **Dynamic 그룹**: 부서 속성을 바꾸면 그룹에 자동으로 들어가고 빠지는가, 얼마나 걸리는가
2. **그룹 기반 라이선스**: 그룹에 라이선스를 붙이면 구성원이 상속받는가
3. **Teams 채널 3종**: 파일이 실제로 어디에 저장되는가

과정의 원칙은 어제와 같다. **"추정하지 말고 조회 결과로 쓴다. 바꾼 건 변경 전후 값까지 기록한다."** 오늘은 이 원칙 덕분에 내 착각을 두 번 잡았다.

> 테넌트 도메인, 객체 ID 같은 식별값은 `xxxx`로 가렸다.

---

## 0. 시작 전: 이 테넌트로 무엇까지 실습할 수 있나

어제 SKU를 봤으니, 오늘은 **실습에 필요한 서비스 플랜이 있는지**를 판정표로 뽑았다.

```powershell
$need = [ordered]@{
  'AAD_PREMIUM'          = 'Entra ID P1 : Dynamic 그룹, 그룹 기반 라이선스, CA'
  'AAD_PREMIUM_P2'       = 'Entra ID P2 : PIM, Access Review, ID Protection'
  'LOCKBOX_ENTERPRISE'   = 'Customer Lockbox'
  'TEAMS1'               = 'Teams'
  'SHAREPOINTENTERPRISE' = 'SharePoint Online (Plan 2)'
}
```

```
ServicePlan          판정      포함SKU
-----------          ----      -------
AAD_PREMIUM          사용 가능 DEVELOPERPACK_V2_E5
AAD_PREMIUM_P2       사용 가능 DEVELOPERPACK_V2_E5
LOCKBOX_ENTERPRISE   사용 가능 DEVELOPERPACK_V2_E5
TEAMS1               사용 가능 DEVELOPERPACK_V2_E5
SHAREPOINTENTERPRISE 사용 가능 DEVELOPERPACK_V2_E5

사용자 수 : 17  (UsageLocation 없는 사용자 0명)
그룹 수   : 8   (Dynamic 0개)
```

E5 개발자 SKU라 P1·P2가 다 있다. 3주차 PIM·Access Review까지 이 테넌트로 가능하다는 근거가 생겼다.

### 좌석(seat)이라는 말

라이선스는 "몇 명분"으로 산다. 그 1명분이 **좌석**이다. `Enabled 25 / ConsumedUnits 17`이면 25석을 사서 17석을 쓰고 8석이 남았다는 뜻이고, M365 관리 센터의 "17/25개 할당됨"과 같은 숫자다. 좌석이 바닥나면 그룹에 새로 들어온 사람이 라이선스를 못 받는다.

---

## 1. Dynamic 그룹: 부서를 바꾸면 자동으로 들어가고 빠지는가

### 규칙

```powershell
$Rule = '(user.department -eq "Finance") and (user.accountEnabled -eq true) and (user.userType -eq "Member")'

New-MgGroup -DisplayName "DG-DEPT-Finance" -MailNickname "DG-DEPT-Finance" `
  -MailEnabled:$false -SecurityEnabled:$true `
  -GroupTypes @("DynamicMembership") -MembershipRule $Rule -MembershipRuleProcessingState "On"
```

부서 조건만 걸지 않고 **비활성 계정과 Guest를 뺐다.** 나중에 이 그룹에 라이선스를 붙이면, 퇴사자 계정이나 외부인에게 좌석이 나가는 걸 규칙 단계에서 막을 수 있다.

### 반영 시간 측정

변경 직후부터 10초 간격으로 구성원을 다시 조회했다.

```powershell
Update-MgUser -UserId $TestUpn -Department "R&D"
$changedAt = Get-Date
do {
  $m = @(Get-MgGroupMember -GroupId $grp.Id -All | % { $_.AdditionalProperties.userPrincipalName })
  "{0}  실제인원 {1}  (변경 후 {2:N0}초)" -f (Get-Date -Format HH:mm:ss), $m.Count, ((Get-Date)-$changedAt).TotalSeconds
  if ($TestUpn -notin $m) { break }
  Start-Sleep 10
} while (((Get-Date)-$changedAt).TotalMinutes -lt 30)
```

| 측정 | 방법 | 결과 | 말할 수 있는 범위 |
|---|---|---|---|
| 그룹 생성 후 첫 반영 | 조회 2회 (생성 직후, 6분 뒤) | 0명 → 1명 | **6분 24초 이내** (정확한 시점은 모름) |
| 부서 변경 → 편입 | 60초 간격 조회 | 1분 뒤 반영 | **약 1분 이내** |
| 부서 원복 → 제외 | 10초 간격 조회 | 72초 미반영 → 82초 반영 | **72~82초 사이** |

### 착각 ①: "반영은 1초 이내"

처음엔 그룹을 만들고 몇 분 뒤 확인 스크립트를 돌렸는데, 첫 조회에서 바로 반영돼 있었다. 그래서 "1초 이내"라고 적으려다 멈췄다. **이미 반영된 뒤에 처음 쟀을 뿐이다.** 측정 방식이 말해 줄 수 있는 건 "생성 시각부터 첫 확인 시각 사이 어딘가"까지다. 그래서 변경 시각을 변수에 저장하고 간격을 좁혀 다시 쟀다.

### MS 문서 기준

Microsoft 문서는 Dynamic 그룹 처리가 **보통 몇 시간 안에 끝나지만 24시간을 넘길 수도 있고, 판단 전에 최소 24시간을 기다리라**고 한다 ([Troubleshoot dynamic groups](https://learn.microsoft.com/en-us/troubleshoot/azure/active-directory/troubleshoot-dynamic-groups)). 내 측정값은 **사용자 17명짜리 테넌트**의 값이라 큰 조직에 그대로 쓰면 안 된다.

> 운영 관점: Dynamic 그룹은 "부서 값 하나"에 권한과 라이선스가 걸린다. 부서를 옮긴 사람은 기존 그룹에서 자동으로 빠지고, 그 그룹으로 받던 것도 같이 사라진다. 부서 값 오타(`Finance` vs `Finance `)도 같은 결과를 낸다.

---

## 2. 그룹 기반 라이선스: "성공"했는데 적용되지 않았다

Dynamic 그룹에 E5를 붙여서 구성원이 **상속(Inherited)**으로 받는지 보려고 했다. 구성원 한 명은 이미 E5를 **직접 할당(Direct)**으로 받고 있어서, 두 경로를 동시에 가지면 좌석이 1개인지 2개인지도 같이 보려 했다.

### 라이선스 경로 확인 방법

```powershell
(Get-MgUser -UserId $upn -Property LicenseAssignmentStates).LicenseAssignmentStates |
  Select-Object SkuId, AssignedByGroup, State, Error
```

- `AssignedByGroup`이 비어 있으면 **Direct**
- 그룹 ID가 들어 있으면 **Inherited**

### 실행

```powershell
Set-MgGroupLicense -GroupId $grp.Id -AddLicenses @(@{ SkuId = $sku.SkuId }) -RemoveLicenses @()
```

오류 없이 끝났다. 그런데 15분 동안 10초마다 봐도 구성원에게 Inherited가 붙지 않았다. 좌석은 17 → 17.

### 착각 ②: "좌석이 그대로니까 동시 보유해도 1석이다"

스크립트가 마지막에 "좌석 변화 없음 = 1석 확인"이라고 찍었다. 하지만 **상속 자체가 안 붙었으니 좌석이 변할 이유가 없었던 것**이다. 근거가 아니다. 스크립트를 고쳐서 상속이 Active일 때만 좌석 결론을 내도록 했다.

### 같은 상태를 여러 곳에서 대조

| 확인 위치 | 그룹 E5 할당 |
|---|---|
| `Set-MgGroupLicense` 실행 | 오류 없음 |
| Graph API 직접 호출 (`POST /groups/{id}/assignLicense`) | 오류 없음 |
| Graph 그룹 조회 `assignedLicenses` (목록·단건) | **0건** |
| Entra 관리 센터 › 그룹 › 라이선스 | **"라이선스 할당을 찾을 수 없습니다"** |
| M365 관리 센터 › 제품 › 라이선스 목록 | **그룹이 표시됨** |
| M365 관리 센터 › 사용자 › 라이선스 및 앱 | 직접 할당만, 그룹 상속 없음 |

화면마다 말이 다르다. 그래서 **감사 로그**를 봤다.

```powershell
Get-MgAuditLogDirectoryAudit -Filter "activityDateTime ge 2026-10-14T02:24:00Z and activityDateTime le 2026-10-14T02:26:00Z" -All |
  % { $e=$_; foreach($t in $e.TargetResources){ foreach($p in $t.ModifiedProperties){
      [pscustomobject]@{ KST=$e.ActivityDateTime.AddHours(9).ToString('HH:mm:ss'); 활동=$e.ActivityDisplayName
                         대상="$($t.DisplayName) $($t.UserPrincipalName)"; 속성=$p.DisplayName; 변경전=$p.OldValue; 변경후=$p.NewValue } } } } | fl
```

```
KST    : 11:24:58
활동   : Update group
대상   : DG-DEPT-Finance
속성   : Included Updated Properties
변경후 : ""                      ← 실제로 바뀐 속성이 없다

KST    : 11:25:06
활동   : Change user license
속성   : LicenseAssignmentDetail
변경후 : [{"ReferenceObjectId":"xxxx-…", ...}]
```

- `Update group`은 **success지만 바뀐 속성 목록이 비어 있다.** 요청은 받았는데 저장된 변경이 없다는 뜻이다.
- 8초 뒤의 `Change user license`에서 `ReferenceObjectId`를 조회해 보니 **그룹이 아니라 사용자 본인**이었다. 그룹 상속이 아니라 기존 직접 할당 기록이 다시 쓰인 것이다.

**결론: 그룹 라이선스 할당은 적용되지 않았다. 원인은 아직 모른다.** 질문 목록에 올리고 동료 Sandbox에서 재현해 보기로 했다.

> 오늘의 가장 큰 교훈. **명령이 오류 없이 끝난 것과 변경이 적용된 것은 다르다.** 변경한 뒤에는 다시 조회해서 확인하고, 그다음에 변경이력을 남긴다. 내 첫 스크립트는 재조회 없이 변경이력을 기록하게 짜여 있었다.

### 참고: 감사 로그 시간은 UTC

`activityDateTime 02:24Z`는 한국 시간 11:24다. 필터는 UTC로 걸고, 출력할 때 `+9시간` 하거나 `ToLocalTime()`을 쓴다.

---

## 3. Teams 채널 3종: 파일은 어디에 저장되나

팀 하나에 채널 세 개를 만들고 파일을 하나씩 올린 뒤, Graph로 실제 저장 위치를 추적했다.

| 채널 유형 | 누가 보나 |
|---|---|
| 표준 | 팀 구성원 전원 |
| 비공개 | 팀 안에서 초대된 일부 |
| 공유 | 초대된 사람. 다른 팀·다른 조직 사람도 가능 |

```powershell
foreach ($ch in Get-MgTeamChannel -TeamId $team.Id -All) {
  $ff = Get-MgTeamChannelFileFolder -TeamId $team.Id -ChannelId $ch.Id
  "{0}  →  {1}" -f $ch.DisplayName, $ff.WebUrl
}
```

### 결과

```
CH-Standard → https://xxxx.sharepoint.com/sites/TM-D3-Readiness/Shared Documents/CH-Standard
CH-Private  → https://xxxx.sharepoint.com/sites/TM-D3-Readiness-CH-Private/Shared Documents
CH-Shared   → https://xxxx.sharepoint.com/sites/TM-D3-Readiness-CH-Shared/Shared Documents
```

```
팀 TM-D3-Readiness
├─ 표준 채널   → 팀 사이트 안의 "폴더"
├─ 비공개 채널 → 별도 SharePoint 사이트 ①
└─ 공유 채널   → 별도 SharePoint 사이트 ②
```

**팀은 1개인데 SharePoint 사이트는 3개다.** Teams의 "파일" 탭은 SharePoint 문서 보관함을 보여 주는 창일 뿐이다.

> 운영 관점: "팀 사이트의 공유·권한 설정을 점검했다"고 해도 비공개·공유 채널 사이트는 빠졌을 수 있다. 점검 대상 목록을 만들 때 채널 사이트까지 포함해야 한다.

### 함정: 공유 채널이 `unknownFutureValue`로 나온다

처음 조회에서 공유 채널의 `MembershipType`이 `shared`가 아니라 `unknownFutureValue`로 나왔다. 오류가 아니라 Graph의 **확장형 열거형(evolvable enum)** 동작이다. 비교적 새로 생긴 값은 요청 헤더를 붙여야만 돌려준다 ([channel resource type](https://learn.microsoft.com/en-us/graph/api/resources/channel)).

```powershell
Invoke-MgGraphRequest -Method GET -Uri "v1.0/teams/$($team.Id)/channels?`$select=id,displayName,membershipType" `
  -Headers @{ Prefer = "include-unknown-enum-members" }
```

헤더를 붙이고 다시 돌리니 `shared`로 나왔다. 헤더 없이 집계하면 **공유 채널 수가 0으로 잡힌다.**

### 같이 확인한 것

- **Owner 1명.** 이 사람이 퇴사하면 관리할 사람이 없는 팀이 된다.
- **그룹 만료 정책 없음.** 안 쓰는 팀이 생겨도 자동으로 정리되지 않는다.
- 생성 시각을 손으로 적어 둔 메모(13:52, 13:53)와 Graph의 `CreatedDateTime`(13:52:35, 13:53:00)을 대조했더니 일치했다. 변경이력의 시각은 손 메모가 아니라 시스템 기록으로 남기기로 했다.

---

## 4. PowerShell 5.1에서 겪은 것

| 증상 | 원인 | 대응 |
|---|---|---|
| `$a ? $b : $c` 구문 오류 | 삼항 연산자는 PowerShell 7부터 | `if (...) { } else { }`로 작성 |
| 스크립트의 한글이 깨짐 | 5.1은 BOM 없는 UTF-8을 ANSI(CP949)로 읽음 | 스크립트를 **UTF-8(BOM)**으로 저장 |
| CSV를 엑셀에서 열면 한글 깨짐 | 5.1 `Export-Csv` 기본 인코딩 | `-Encoding UTF8` 명시 |
| `용어가 cmdlet ... 인식되지 않습니다` | 스크립트가 현재 폴더에 없음 | 경로 확인, 받은 파일은 `Unblock-File` |
| 시간이 `오전 4:51`로 찍힘 | Graph 시간은 UTC | `.ToLocalTime()` |

어제 글에서 "앞으로는 PowerShell 7로 하겠다"고 했는데, 실제 운영 환경은 5.1일 수도 있어서 스크립트는 **5.1 문법 기준**으로 맞추는 쪽으로 정리했다.

---

## 정리

세 실습 모두 결국 같은 이야기였다.

1. **명령 성공 ≠ 적용.** 바꾼 뒤에는 다시 조회한다.
2. **측정값은 측정 방식이 말해 주는 범위까지만 쓴다.** "1초 이내"가 아니라 "6분 24초 이내"다.
3. **한 화면만 보고 판정하지 않는다.** Graph, Entra 관리 센터, M365 관리 센터, 감사 로그가 서로 다른 말을 할 수 있다.
4. **모르는 건 모른다고 적고 넘어간다.** 그룹 라이선스 미적용은 시간을 정해 두고 끊었다.

"설정은 되어 있는데 실제로는 다르게 돌아가고, 아무도 모르는" 상태를 오늘 세 번 봤다. 운영에서 찾아야 하는 게 바로 이런 것들이다.

내일은 Azure 계층(관리 그룹 › 구독 › 리소스 그룹)과 RBAC를 본다.
