---
layout: post
title: "M365 실습기 - 보안 기본값, 비상 계정, 그리고 그룹 라이선스가 안 붙던 이유"
date: 2026-10-16 18:00:00 +0900
categories: [M365, EntraID]
tags: [m365, entra-id, security-defaults, conditional-access, break-glass, emergency-access, group-based-licensing, dynamic-group, audit-log, powershell, readiness]
---

# M365 실습기 - 보안 기본값 · 비상 계정 · 그룹 라이선스 원인 추적

운영 Readiness 사전 프로그램 1주차 마지막 날. 지금까지 [실습환경 점검]({% post_url 2026-10-13-m365-azure-sandbox-readiness %}), [Dynamic 그룹·그룹 라이선스·Teams 저장 구조]({% post_url 2026-10-14-dynamic-group-licensing-teams-storage %}), [Azure 계층과 RBAC]({% post_url 2026-10-15-azure-rbac-elevate-and-chain-map %})를 봤다.

오늘 한 일은 두 가지다.

1. **D5 실습**: 보안 기본값·조건부 액세스(CA) 상태를 확인하고, 비상 접근 계정(break-glass)을 만들었다.
2. **이틀째 풀리지 않던 문제**: "그룹에 라이선스를 줬는데 안 붙는다"의 원인을 실험으로 좁혔다.

결론부터 말하면, 그룹 라이선스는 **같은 명령인데 고정 그룹에선 됐고 Dynamic 그룹에선 안 됐다.** 원인을 끝까지 밝히지는 못했지만, 어디가 문제인지는 꽤 정확히 짚었다.

> 테넌트 도메인, 테넌트 ID 같은 식별값은 `xxxx`로 가렸다. 사용자 이름은 개발자 Sandbox 기본 샘플 사용자와 실습용으로 만든 계정이다.

---

## 1. 보안 기준선: 전역 관리자가 1명뿐이었다

먼저 지금 상태를 읽었다. 바꾸는 건 없다.

```powershell
Connect-MgGraph -Scopes "Policy.Read.All","RoleManagement.ReadWrite.Directory","AuditLog.Read.All" -NoWelcome
$GA = "62e90394-69f5-4237-9190-012177145e10"   # 전역 관리자 roleTemplateId (모든 테넌트 공통)

Get-MgPolicyIdentitySecurityDefaultEnforcementPolicy | Select DisplayName, IsEnabled
Get-MgIdentityConditionalAccessPolicy -All
Get-MgRoleManagementDirectoryRoleAssignment -Filter "roleDefinitionId eq '$GA'" -ExpandProperty Principal
Get-MgRoleManagementDirectoryRoleEligibilityScheduleInstance -Filter "roleDefinitionId eq '$GA'"
Get-MgReportAuthenticationMethodUserRegistrationDetail -All -Filter "isAdmin eq true"
```

```
=== 보안 기본값 ===
Security Defaults      True
=== 조건부 액세스 정책 ===
CA 정책 0개
=== 전역 관리자 - 활성 할당 ===
boklok@xxxx    #microsoft.graph.user /
=== 전역 관리자 - 적격(PIM) ===
적격 할당 0건
=== 관리자 MFA 등록 현황 ===
boklok@xxxx    True  passKeySynced,microsoftAuthenticatorPush,softwareOneTimePasscode
```

| 항목 | 결과 | 의미 |
|---|---|---|
| 보안 기본값 | 사용 중 | 모든 사용자 MFA 등록 필수, 관리자는 로그인마다 MFA |
| 조건부 액세스 | 0개 | 보안 기본값과 CA는 **둘 중 하나만** 쓸 수 있어서 정상 조합 |
| 전역 관리자 | **1명**, PIM 적격 0 | 이 계정이 잠기거나 MFA 기기를 잃으면 **테넌트를 관리할 사람이 없다** |

비상 계정이 왜 필요한지가 조회 결과에 그대로 나와 있었다.

### 보안 기본값에는 "예외"가 없다

보안 기본값은 켜거나 끄는 것만 있다. 특정 사용자를 빼는 설정이 없다. 그래서 "비상 계정은 MFA 정책에서 제외한다"는 흔한 권고는 **CA로 전환한 뒤에만** 할 수 있다. 실무에서 보안 기본값을 쓰는 조직이면 이 전환이 비상 계정 설계의 전제 조건이 된다.

---

## 2. 비상 접근 계정 만들기

```powershell
# 비밀번호는 화면에 출력하지 않고 클립보드로만 넘김
New-MgUser -DisplayName "Break Glass 01" -UserPrincipalName breakglass01@xxxx.onmicrosoft.com `
  -AccountEnabled -UsageLocation KR -MailNickname breakglass01 `
  -PasswordProfile @{ Password = $pw; ForceChangePasswordNextSignIn = $false }

# PIM 적격이 아니라 '활성·영구'로 할당
New-MgRoleManagementDirectoryRoleAssignment -PrincipalId $bg.Id -RoleDefinitionId $GA -DirectoryScopeId "/"
```

```
[변경이력] 2026-10-16 11:18:55 | 사용자 breakglass01 생성 | 전: 없음 → 후: 클라우드 전용, 사용 위치 KR, 라이선스 없음
[변경이력] 2026-10-16 13:03:04 | breakglass01 역할 @ 디렉터리(/) | 전: 없음 → 후: 전역 관리자 활성·영구
=== 전역 관리자 활성 할당 (재조회) ===
boklok@xxxx
breakglass01@xxxx
```

이번에도 바꾼 뒤에는 다시 조회했고, 재조회로 확인된 것만 변경이력에 남겼다.

### MS 권고와 대조해 보니

[Microsoft Entra 비상 접근 계정 관리](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access) 문서와 비교했다.

| MS 권고 | 이번 실습 |
|---|---|
| 비상 계정 **2개 이상** | 1개 |
| 클라우드 전용 `.onmicrosoft.com` 계정 | 충족 |
| 전역 관리자를 **영구 활성**으로 (PIM 적격 X) | 충족 |
| **피싱 방지 MFA**(FIDO2 패스키 등), 평소 관리자와 **다른 수단** | 미등록 |
| 차단·제한하는 CA 정책에서 제외 | 보안 기본값이라 불가 |
| 모든 로그인에 알림, **90일마다** 점검 | 미구성 |

PIM 적격으로만 두면 안 되는 이유가 문서에 있다. 승인해 줄 사람이 없는 상황이면 적격 역할은 활성화조차 못 해서 테넌트 관리가 잠긴다. 비상 계정은 그 상황을 위한 계정이다.

### 실수 하나: 비밀번호를 잃었다

비밀번호를 화면에 출력하지 않으려고 클립보드에만 복사했다. 그런데 바로 다음에 화면을 캡처(Win+Shift+S)했고, **캡처가 클립보드를 덮어써서** 비밀번호가 사라졌다.

Sandbox라서 필요할 때 다른 관리자가 비밀번호를 재설정하면 그만이다. 하지만 실무라면 이건 **비상 계정의 가장 기본이 무너진 것**이다. 비상 계정은 만드는 순간 자격 증명을 어디에 누가 보관하는지가 같이 정해져 있어야 한다. 생성 절차와 보관 절차를 한 단계로 묶어야 한다는 걸 몸으로 배웠다.

---

## 3. 그룹 라이선스가 안 붙던 이유 추적

### 지금까지의 상황

[10/14 글]({% post_url 2026-10-14-dynamic-group-licensing-teams-storage %})에서 Dynamic 그룹 `DG-DEPT-Finance`에 PowerShell로 E5 라이선스를 할당했는데, 성공 응답과 달리 **그룹에 할당이 저장되지 않았다.** 21시간이 지나도 그대로였다.

어제 대조 실험으로 "M365 관리 센터 UI로 할당하면 그룹에 저장된다"까지는 확인했다. 그래서 의심은 이렇게 두 갈래였다.

- **H2**: Dynamic 그룹이라서 안 된다
- **H3**: PowerShell·Graph로 할당해서 안 된다

### 감사 로그로 어제를 다시 보다

먼저 어제 대조 실험에서 실제로 무슨 일이 있었는지 감사 로그로 되짚었다.

```powershell
Get-MgGroup -Filter "startswith(displayName,'SG-LIC-TEST')" -Property Id,DisplayName,CreatedDateTime,AssignedLicenses,LicenseProcessingState
Get-MgAuditLogDirectoryAudit -Filter "activityDateTime ge 2026-10-15T06:00:00Z" -All
```

```
DisplayName  생성(KST)   그룹할당     처리상태
SG-LIC-TEST  15:05:07    E5          QueuedForProcessing
SG-LIC-TEST2 15:10:23    (없음)
SG-LIC-TEST2 15:17:33    E5          ProcessingComplete      ← 같은 이름 그룹이 2개!

KST       Activity                                       대상          수행
15:07:41  Set group license                              SG-LIC-TEST   CloudLicensingSystem
15:07:48  Update group                                   SG-LIC-TEST   CloudLicensingSystem
15:10:23  Add group                                      SG-LIC-TEST2  boklok
15:17:33  Add group                                      SG-LIC-TEST2  boklok
15:26:11  Set group license                              SG-LIC-TEST2  CloudLicensingSystem
15:26:15  Start applying group based license to users    SG-LIC-TEST2  Group-Based Licensing
15:26:15  Finish applying group based license to users   SG-LIC-TEST2  Group-Based Licensing
```

여기서 세 가지를 알았다.

1. **같은 이름의 그룹이 2개** 만들어져 있었다. 실험을 다시 돌리다 생긴 중복이다. Entra는 그룹 표시 이름 중복을 막지 않는다. 이름으로 조회하던 스크립트가 배열을 받아서 처리상태가 `System.Object[]`로 찍힌 이유도 이거였다.
2. UI로 할당하면 `CloudLicensingSystem`이 **Set group license**를 남기고, 이어서 Group-Based Licensing 엔진이 **Start → Finish applying**을 남긴다. 두 번째 그룹은 이 과정이 **4초** 만에 끝났다.
3. 그런데 첫 번째 그룹(SG-LIC-TEST)은 같은 UI 할당인데 **22시간째 QueuedForProcessing**이고, 적용 이벤트가 없다. 새 미해결 건이다.

### 실험 A: 고정 그룹 + PowerShell

어제 만든 중복 그룹 중 첫 번째는 고정 그룹이고, 구성원이 있고, 할당이 없었다. 조건이 딱 맞아서 그대로 썼다.

```powershell
Set-MgGroupLicense -GroupId $gA -AddLicenses @(@{SkuId = $sku.SkuId}) -RemoveLicenses @()
Start-Sleep 90
Get-MgGroup -GroupId $gA -Property AssignedLicenses,LicenseProcessingState
```

```
요청: 13:10:47
응답: 오류 없음
[13:12:24] 그룹할당=0d1bdf5e-… / 처리상태=ProcessingComplete
```

**1분 37초 이내에 적용 완료.** PowerShell 경로 자체는 정상이다. H3는 기각했다.

### 실험 B: Dynamic 그룹 + 같은 명령

그럼 10/14에만 생긴 일시적 문제였을까? 같은 명령을 `DG-DEPT-Finance`에 다시 돌렸다. 구성원은 이미 직접 할당을 받고 있어서 좌석은 늘지 않는다.

```
요청: 13:14:24
응답: 오류 없음
[13:16:00] 그룹할당= / 처리상태=
```

**재현됐다.** 이틀 연속, 같은 명령, 같은 시간대에 고정 그룹은 되고 Dynamic 그룹은 안 된다.

| | 고정 그룹 | Dynamic 그룹 |
|---|---|---|
| 명령 | `Set-MgGroupLicense` | `Set-MgGroupLicense` (동일) |
| 응답 | 오류 없음 | 오류 없음 |
| 그룹 할당 저장 | **됨** | **안 됨** (10/14, 10/16 두 번) |
| 처리상태 | ProcessingComplete | 공란 |

### 여기까지의 결론

원인 범위는 **이 Dynamic 그룹**으로 좁혀졌다. Dynamic 그룹은 그룹 기반 라이선스와 함께 쓸 수 있는 조합으로 알려져 있다. 그래서 "Dynamic 그룹은 원래 안 된다"고 단정할 수는 없다. 이 그룹만의 상태인지, 이 Sandbox 환경의 Dynamic 그룹 전반 문제인지는 아직 모른다.

다음 실험은 정해 뒀다.

- 새 Dynamic 그룹을 하나 더 만들어 같은 명령으로 재현해 보기 → 그룹 고유 문제인지 판정
- `DG-DEPT-Finance`에 UI로 할당해 보기 → 경로와 그룹 유형의 조합 확인

---

## 4. 오늘 배운 것

1. **변수는 하나씩만 바꾼다.** "PowerShell이 원인"이라는 가설은 그럴듯했지만, 그룹 유형만 바꿔 같은 명령을 돌리자 바로 틀린 걸 알 수 있었다. 한 번에 두 가지를 바꿨다면 아무것도 말할 수 없었을 것이다.
2. **감사 로그는 "누가"까지 말해 준다.** 수행자가 사람 계정인지, `CloudLicensingSystem` 같은 시스템 앱인지, Group-Based Licensing 엔진인지가 찍힌다. 경로를 구분하는 가장 확실한 증거였다.
3. **이름은 식별자가 아니다.** 같은 이름의 그룹이 2개 생겨도 Entra는 막지 않는다. 스크립트는 이름이 아니라 ID로 다뤄야 한다.
4. **비상 계정은 계정 + 보관 절차 + 점검 주기가 한 세트다.** 계정만 만들면 반쪽이다.
5. **보안 기본값은 단순한 대신 예외가 없다.** 예외가 필요해지는 순간 CA로 넘어가야 한다.

---

## 1주차 정리

닷새 동안 직접 해 본 것을 운영 역량 체인(전략·소유권 / 정책·통제 / 플랫폼 운영 / 기술 지원 / MS 창구 / 가치·활용)에 붙여 한 장으로 정리해서 제출했다. 다 붙이고 나니 패턴이 하나 보였다.

**"설정은 정상처럼 보이는데 실제는 다른" 곳**이 영역마다 있었다.

- M365: 명령은 성공인데 그룹 라이선스는 0건
- Teams: 팀은 1개인데 SharePoint 사이트는 3개
- Azure: 구독 권한 목록에 하위 범위 할당이 섞여 나옴, 구독 Owner 중 하나는 사람이 아니라 앱
- Identity: 전역 관리자는 있지만 1명뿐

운영을 넘겨받을 때 맨 먼저 확인할 곳이 바로 이런 지점일 것이다.

다음 주는 Graph 앱 등록과 인증서 기반 접속으로, 오늘까지 손으로 돌린 조회를 자동 수집으로 바꿔 본다.
