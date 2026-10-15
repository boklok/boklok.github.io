---
layout: post
title: "Azure 실습기 - 범위별 역할 할당, Elevate access, 그리고 지금까지를 한 장으로"
date: 2026-10-15 12:00:00 +0900
categories: [Azure, EntraID, M365]
tags: [azure, rbac, management-group, elevate-access, entra-id, powershell, az-module, readiness]
---

# Azure 실습기 - 범위별 역할 할당 · Elevate access · 운영 체인 지도

운영 Readiness 사전 프로그램 넷째 날. 지금까지 [실습환경 점검]({% post_url 2026-10-13-m365-azure-sandbox-readiness %}), [Dynamic 그룹·그룹 라이선스·Teams 저장 구조]({% post_url 2026-10-14-dynamic-group-licensing-teams-storage %})를 봤고, 오늘은 Azure로 넘어왔다.

1. **범위별 역할 할당**: 리소스 그룹에 준 권한은 어디까지 적용되고, 어디에 "보이는가"
2. **Elevate access**: 전역 관리자가 테넌트 전체 구독을 보려면 무엇을 거쳐야 하고, 어떻게 거둬야 하나
3. **지금까지 한 것을 한 장으로**: 운영 체인 지도

원칙은 그대로다. **"바꾼 뒤에는 다시 조회하고, 확인된 것만 기록한다."** 오늘도 내 예측이 한 번 틀렸고, 명령이 한 번 거부됐다.

> 테넌트 도메인, 구독 ID 같은 식별값은 `xxxx`로 가렸다. 사용자 이름은 개발자 Sandbox에 기본 제공되는 샘플 사용자다.

---

## 0. Azure 계층 한 줄 정리

```
Tenant Root Group (관리 그룹, 이름 = 테넌트 ID)
└─ 관리 그룹
   └─ 구독
      └─ 리소스 그룹
         └─ 리소스
```

- 역할은 **위에서 아래로만** 상속된다. 구독 Owner는 그 아래 모든 리소스 그룹에서 Owner다.
- 아래에서 준 권한은 위로 올라가지 않는다. 리소스 그룹 Reader는 구독 전체를 볼 수 없다.

여기까지는 시험(AZ-104) 공부 때 외운 내용이다. 오늘은 이걸 **조회 결과로** 확인했다.

### 준비: Graph와 Az는 창을 나눈다

같은 PowerShell 창에 Microsoft.Graph를 이미 불러온 상태에서 `Connect-AzAccount`를 하니 `Azure.Core` 어셈블리 충돌이 났다. 두 모듈이 서로 다른 버전의 같은 DLL을 쓰기 때문이다. **Az 작업은 새 창에서** 하는 것으로 정리했다.

---

## 1. 기준선: 구독 Owner가 2명인데, 하나는 사람이 아니다

```powershell
Get-AzRoleAssignment -Scope "/subscriptions/xxxx" |
  Select-Object DisplayName, ObjectType, RoleDefinitionName, Scope
```

```
DisplayName                                     ObjectType        RoleDefinitionName
-----------                                     ----------        ------------------
boklok                                          User              Owner
Microsoft-Developer Program-Sample Data Packs   ServicePrincipal  Owner
```

구독 Owner 중 하나가 **앱(서비스 주체)** 이었다. 이름으로 보면 샘플 데이터를 배포하는 앱으로 보이지만, 확인은 못 했다.

> 운영 관점: "구독 Owner가 누구인가" 점검을 사람 계정만 대상으로 하면 이런 앱 Owner가 빠진다. Owner 권한을 가진 앱은 누가 관리하는지, 비밀키는 언제 만료되는지까지 따라가야 한다.

---

## 2. 리소스 그룹 Reader는 구독 목록에 "보인다", 하지만 적용되지는 않는다

리소스 그룹을 하나 만들고(태그로 소유자와 용도를 붙였다), 테스트 사용자에게 **리소스 그룹 범위 Reader**를 줬다.

```powershell
New-AzResourceGroup -Name rg-d4-readiness -Location koreacentral `
  -Tag @{ Owner = "boklok"; Purpose = "D4-RBAC-lab" }
New-AzRoleAssignment -SignInName "AdeleV@xxxx" -RoleDefinitionName Reader `
  -Scope (Get-AzResourceGroup rg-d4-readiness).ResourceId
```

내 예측은 "구독 범위로 조회하면 AdeleV는 안 나온다"였다. **틀렸다.**

```powershell
Get-AzRoleAssignment -Scope "/subscriptions/xxxx"   # 기본 조회
```

```
DisplayName  RoleDefinitionName  Scope
-----------  ------------------  -----
boklok       Owner               /subscriptions/xxxx
AdeleV       Reader              /subscriptions/xxxx/resourceGroups/rg-d4-readiness   ← 나온다
```

`Get-AzRoleAssignment -Scope`는 기본값으로 **그 범위 아래(하위 범위)에 준 할당까지 같이 나열**한다. 권한이 구독에 적용된다는 뜻이 아니다. `-AtScope`를 붙이거나 할당 범위로 필터하면 사라진다.

```powershell
Get-AzRoleAssignment -Scope "/subscriptions/xxxx" -AtScope   # AdeleV 안 나옴
```

그래서 결과를 볼 때 판정을 바꿨다. **"목록에 있느냐"가 아니라 "할당된 범위가 어디냐"로 본다.**

```powershell
function Show-Assign([string]$scope) {
  Get-AzRoleAssignment -Scope $scope |
    Select-Object @{n='적용';e={
        if ($_.Scope -eq $scope) {'직접'}
        elseif ($_.Scope -eq '/' -or $scope.StartsWith($_.Scope.TrimEnd('/') + '/')) {'상속'}
        else {'아님(하위)'} }},
      DisplayName, RoleDefinitionName, @{n='할당된범위';e={$_.Scope}}
}
```

```
=== 리소스 그룹 범위에서 본 할당 ===
적용  DisplayName  RoleDefinitionName  할당된범위
----  -----------  ------------------  ----------
상속  boklok       Owner               /subscriptions/xxxx
직접  AdeleV       Reader              /subscriptions/xxxx/resourceGroups/rg-d4-readiness

=== 구독 범위에서 본 할당 ===
직접      boklok   Owner    /subscriptions/xxxx
아님(하위) AdeleV   Reader   /subscriptions/xxxx/resourceGroups/rg-d4-readiness
```

> 운영 관점: "구독 권한 목록"을 기본 명령으로 뽑아 감사 자료로 내면, 리소스 그룹에만 권한이 있는 사람이 구독 권한자처럼 섞여 들어간다. 반대로 범위를 잘못 걸러서 상위에서 상속받은 권한을 빠뜨릴 수도 있다. 권한 목록에는 **할당 범위 열을 꼭 같이 남긴다.**

---

## 3. Elevate access: 전역 관리자도 처음엔 관리 그룹을 못 본다

전역 관리자이자 구독 Owner인 내 계정으로 관리 그룹을 조회했다.

```powershell
Get-AzManagementGroup
# 거부: Microsoft.Management/managementGroups/read 권한 없음
```

Entra ID의 전역 관리자는 **Azure 리소스 권한과 별개**다. 테넌트 전체 구독을 보려면 먼저 Elevate access로 루트(`/`) 범위의 User Access Administrator를 받아야 한다.

```powershell
Invoke-AzRestMethod -Method POST `
  -Path "/providers/Microsoft.Authorization/elevateAccess?api-version=2016-07-01"
# StatusCode 200 → 다시 로그인(토큰 갱신)
Get-AzRoleAssignment -Scope "/" -RoleDefinitionName "User Access Administrator"
# boklok  /   ← 루트 범위에 생김
```

이 권한으로 할 수 있는 일은 **"권한을 주는 것"** 이다. 그래서 실제로 많이 쓰는 절차는 이렇다.

1. Elevate access로 루트 권한을 잠시 받는다
2. 전체 구독 조회가 필요한 계정에 **Tenant Root Group 범위 Reader**를 준다
3. Elevate access를 **즉시 해제**한다

```powershell
$RootMg = "/providers/Microsoft.Management/managementGroups/<테넌트 ID>"
New-AzRoleAssignment -SignInName "AdeleV@xxxx" -RoleDefinitionName Reader -Scope $RootMg
Get-AzRoleAssignment -SignInName "AdeleV@xxxx" -Scope "/subscriptions/xxxx"
# Reader  /providers/Microsoft.Management/managementGroups/xxxx  ← 구독에 상속되어 보임
```

Root 관리 그룹 Reader 하나로 **테넌트 안의 모든 구독**을 읽을 수 있게 된다. 전체 구독 인벤토리를 만들 때 필요한 권한이다.

### 해제가 PowerShell로는 거부됐다

```powershell
Remove-AzRoleAssignment -SignInName "boklok@xxxx" `
  -RoleDefinitionName "User Access Administrator" -Scope "/"
# Forbidden
```

부여는 REST 호출로 됐는데, 같은 권한을 가진 상태에서 해제는 거부됐다. 원인은 아직 모른다. 대신 포털에서 해제했다.

> Entra ID › 속성 › **Azure 리소스에 대한 액세스 관리** = 아니요 → 저장

다시 로그인한 뒤 확인했다.

| 시점 | `Get-AzManagementGroup` |
|---|---|
| Elevate 전 | 거부 |
| Elevate 중 | Tenant Root Group 조회됨 |
| 해제 후 | **다시 거부** ← 해제가 실제로 적용됨 |

> 운영 관점 두 가지.
> - **올린 권한은 쓰고 나면 바로 거둔다.** 그리고 "거두는 방법"도 미리 검증해 둔다. 오늘처럼 명령이 안 먹으면 권한이 남는다.
> - 실습으로 준 **Root 관리 그룹 Reader는 아직 남아 있다.** 이걸 회수하려면 다시 Elevate가 필요하다. 언제 회수할지는 조에서 정하기로 했다. 남겨 둔 권한은 변경이력에 "잔존"으로 적어 둔다.

---

## 4. 어제의 미해결 건: 21시간 뒤에도 그룹 라이선스 0건

[어제 글]({% post_url 2026-10-14-dynamic-group-licensing-teams-storage %})의 그룹 기반 라이선스 미적용을 아침에 다시 조회했다.

```
그룹 assignedLicenses       : 0건
licenseProcessingState      : (공란)
구성원 licenseAssignmentStates : Direct · Active 만 (Inherited 없음)
```

약 21시간이 지났는데 **그룹에 라이선스 자체가 저장되지 않았다.** "구성원에게 늦게 퍼지는 것"이 아니라 **그룹 할당이 처음부터 적용되지 않은 것**으로 판정을 좁혔다. 원인은 여전히 모르고, 동료의 Sandbox에서 재현해 보기로 했다.

---

## 5. 지금까지를 한 장으로: 운영 체인 지도

나흘간 한 일을 한 장에 정리했다. 축은 두 개다.

- **세로: 영역.** Identity, M365, Teams, Azure
- **가로: 운영 역량.** ① 주인·전략 ② 거버넌스·정책 ③ 플랫폼 운영 ④ 기술 지원 ⑤ 벤더 창구 ⑥ 가치·활용

규칙은 하나다. **해 본 것과 확인한 것만 쓴다. "할 예정"은 넣지 않는다.**

| | ② 거버넌스·정책 | ③ 플랫폼 운영 | ④ 기술 지원 | ⑥ 가치·활용 |
|---|---|---|---|---|
| **Identity** | 감사 로그로 역할·속성 변경 이력 확인 | 사용자 생성·부서 변경, Dynamic 그룹 규칙 관리 | | |
| **M365** | 라이선스로 가능한 통제 기능 판정 | 직접 할당 확인, **그룹 라이선스 미적용(원인 미확인)** | "라이선스 미적용" 문의 1차 판독 | 좌석 현황 집계 |
| **Teams** | 그룹 만료 정책 점검, 채널 사이트 공유 점검 범위 | 채널별 저장 위치 추적, Team Owner 수 확인 | | |
| **Azure** | 구독 Owner 점검(앱 Owner 포함) | 범위별 역할 할당, Elevate → Root MG Reader → 해제 | | |

만들고 나서 보이는 것이 두 가지 있었다.

1. **한 일이 ②와 ③에 몰려 있다.** ① 주인·전략과 ⑤ 벤더 창구는 비어 있다. 빠뜨린 게 아니라 샌드박스에서는 실습할 수 없는 영역이다. 실제 조직을 만나야 채울 수 있다.
2. **네 영역 모두에서 "설정은 정상처럼 보이는데 실제는 다른" 지점이 나왔다.**
   - M365: 명령은 성공했는데 그룹 라이선스는 0건
   - Teams: 팀은 1개인데 SharePoint 사이트는 3개
   - Azure: 구독 권한 목록에 하위 범위 할당이 섞여 나옴
   - Azure: 구독 Owner 중 하나가 사람이 아니라 앱

운영 인수인계 때 점검표를 만든다면, 이 네 곳을 맨 위에 둘 것 같다.

---

## 정리

1. **목록에 보이는 것 ≠ 적용되는 것.** 역할 할당은 할당 범위로 판정한다.
2. **전역 관리자 ≠ Azure 권한.** 전체 구독을 보려면 Elevate가 필요하고, 쓰고 나면 바로 거둔다.
3. **거두는 절차도 검증한다.** 명령이 거부되면 다른 경로로 해제하고, 해제가 적용됐는지 다시 조회한다.
4. **남겨 둔 권한은 남겼다고 적는다.**
5. **한 장으로 모아 봐야 패턴이 보인다.** 하루씩 볼 때는 별개였던 문제들이, 모아 놓고 보니 같은 종류였다.

미해결 질문은 세 개다. 그룹 라이선스 미적용 원인, Elevate 해제가 PowerShell에서 거부된 원인, Root MG Reader 회수 시점. 답이 나오면 이어서 쓰겠다.
