---
layout: post
title: "M365·Azure 실습기 - 점검 명령 41개를 직접 돌려 보며 배운 것 (빈 출력은 0이 아니다)"
date: 2026-10-26 18:00:00 +0900
categories: [M365, Azure]
tags: [powershell, microsoft-graph, sharepoint-online, teams, exchange-online, purview, azure-rbac, azure-policy, troubleshooting]
---

# M365·Azure 실습기 - 점검 명령 41개를 직접 돌려 보며 배운 것

개인 M365 개발자 Sandbox에서 **운영 점검용 조회 명령 41개**를 하나씩 직접 실행해 봤다. 라이선스, Entra 외부 협업, Teams 정책, SharePoint 공유, Purview, Azure RBAC·Policy까지 범위가 넓었다.

목표는 값을 재는 것보다 **"이 명령이 정말 맞는 값을 돌려주는가"**를 확인하는 것이었다. 결과적으로 절반 가까이에서 명령이나 판단 기준을 고쳐야 할 이유가 나왔다.

> 테넌트 이름·ID·구독 ID는 `xxxx` 또는 `<테넌트>`로 가렸다. 모든 실습은 **조회(Get)만** 했고, 상태를 바꾸는 명령은 실행하지 않았다.

---

## 0. 검증 방법: 명령마다 세 블록

항목마다 같은 순서로 확인했다.

1. **명령 그대로** 실행한다.
2. 결과를 **숫자로 다시 센다.** 전체 개수, 걸러진 개수, 상태 코드를 찍는다.
3. **정답을 아는 대상**과 대조한다. 지난 실습에서 내가 직접 만든 사용자·역할 할당이 정답지 역할을 했다.

이 방식 덕분에 "출력이 없다"와 "0개다"를 구별할 수 있었다. 이번 실습의 가장 큰 교훈이 여기서 나왔다.

---

## 1. 빈 출력은 0이 아니다

가장 자주 만난 함정이다. 겉으로는 똑같이 "0" 또는 "아무것도 없음"인데, 이유가 전부 달랐다.

| 상황 | 실제 원인 |
|---|---|
| 90일 미로그인 사용자 0명 | 서버 필터가 **로그인 기록이 없는(null) 계정을 돌려주지 않음** |
| 감사 로그 검색 0건 | 감사가 꺼져 있어 **검색이 실패**했는데, 오류가 아니라 **노란 경고**만 나옴 |
| 정품 인증 한도 도달 사용자 0명 | 보고서 다운로드가 실패해서 **빈 목록을 센 것** |
| Entra 진단 설정 출력 없음 | `Invoke-AzRestMethod`는 **403이어도 빨간 오류를 띄우지 않음** |
| Azure Policy 비준수 0 | 규칙은 있는데 **평가할 리소스가 0개** |

그래서 조회 스크립트를 만들 때 규칙을 하나 정했다.

> **"없음"을 출력할 때는 반드시 이유를 같이 출력한다.** 상태 코드, 전체 개수, 실패 여부를 먼저 찍고, 측정이 안 됐으면 0이 아니라 **"측정 불가"**라고 쓴다.

### 예 1. 서버 필터와 null

```powershell
# 이 필터는 lastSignInDateTime이 null인(한 번도 로그인 안 한) 사용자를 빼고 돌려준다
Get-MgUser -All -Filter "signInActivity/lastSignInDateTime le $d" ...
```

수정한 방식은 전부 받아온 뒤 PC에서 판단하는 것이다.

```powershell
$cut = (Get-Date).ToUniversalTime().AddDays(-90)
$last = @($u.SignInActivity.LastSignInDateTime, $u.SignInActivity.LastNonInteractiveSignInDateTime) |
        Where-Object { $_ } | Sort-Object -Descending | Select-Object -First 1
# 로그인 기록이 없으면 계정 생성일로 판단
$inactive = if ($last) { $last -lt $cut } else { $u.CreatedDateTime -lt $cut }
```

덤으로 하나 더 찾았다. `(Get-Date).ToString('yyyy-MM-ddTHH:mm:ssZ')`는 **한국 시간에 `Z` 글자만 붙인다.** UTC로 바꾸려면 `.ToUniversalTime()`이 필요하다.

### 예 2. 상태 코드부터 본다

```powershell
$r = Invoke-AzRestMethod -Method GET -Path "/providers/microsoft.aadiam/diagnosticSettings?api-version=2017-04-01-preview"
"HTTP 상태: $($r.StatusCode)"          # 200이어야 '설정 없음'을 믿을 수 있다
"설정 수: $(@(($r.Content | ConvertFrom-Json).value).Count)"
```

---

## 2. 숫자는 맞는데 세는 대상이 틀렸다

### 그룹으로만 받은 라이선스가 "직접 할당"으로 잡힘

```powershell
# 그룹 표시 없는 경로가 '하나라도' 있으면 직접 할당으로 센다
Where-Object { $_.LicenseAssignmentStates | Where-Object { -not $_.AssignedByGroup } }
```

지난 실습에서 본 것처럼, 그룹 기반 라이선스 중에는 `assignedByGroup`이 **비어 있는 경로**가 섞여 나올 때가 있다. 그래서 그룹으로만 받은 사용자가 직접 할당으로 잡혔다. 내 Sandbox에서는 **23명 → 실제 22명**이었다.

```powershell
# 그룹 경로가 '하나도 없는' 사용자만 센다
Where-Object { $_.LicenseAssignmentStates -and -not ($_.LicenseAssignmentStates | Where-Object AssignedByGroup) }
```

### 구독 역할 조회에 하위 리소스 그룹이 섞임

`Get-AzRoleAssignment -Scope /subscriptions/<구독>`은 구독에 직접 준 할당만이 아니라 **상위 관리 그룹에서 상속된 것과 하위 리소스 그룹의 것까지** 같이 돌려준다. 그런데 `Scope` 열을 안 찍으면 구분이 안 된다.

| 할당 | 실제 범위 |
|---|---|
| 사용자 A Owner | 구독 직접 |
| **앱(서비스 주체) Owner** | 구독 직접 — 개발자 프로그램이 넣은 것으로 보임 |
| 사용자 B **Contributor** | **리소스 그룹 하나에만** |
| 사용자 C Reader | **루트 관리 그룹에서 상속** |

`Scope` 없이 보면 사용자 B가 구독 전체의 Contributor처럼 보인다. "구독 Owner 3명 이하" 같은 기준은 **구독 직접 + 상위 상속만** 세고, **사람과 앱을 나눠서** 세야 한다.

### 정책별 건수 합계 ≠ 비준수 리소스 수

`Get-AzPolicyState ... | Group-Object PolicyDefinitionName`은 정책별 건수다. 리소스 하나가 규칙 3개를 어기면 3번 세어진다. 비율을 내려면 **고유 리소스 ID** 기준으로 다시 세야 한다.

---

## 3. 조용히 틀리는 명령

### `Select-Object`는 없는 속성에도 오류를 내지 않는다

Teams 정책 8종을 확인하면서 이 검사를 넣었다.

```powershell
$names = $obj.PSObject.Properties.Name
$miss  = $wanted | Where-Object { $_ -notin $names }
"없는 속성: " + $(if ($miss) { $miss -join ', ' } else { '없음' })
```

실제로 `Get-CsTeamsUpdateManagementPolicy`에서 **존재하지 않는 속성 이름**이 하나 걸렸다. 그냥 `Select-Object`로 봤다면 빈 칸으로 보이고 지나갔을 것이다.

### v1.0에는 없는 속성

```
Get-MgDeviceManagementManagedDevice ... -Property SkuFamily
→ 400 BadRequest: Could not find a property named 'SkuFamily' on type 'microsoft.graph.managedDevice'
```

같은 속성이 **베타 엔드포인트**에서는 조회됐다. 문서나 예제에서 베타 속성 이름을 가져다 v1.0 cmdlet에 쓰면 이렇게 된다. 반대로 그룹 명명 정책은 Beta 모듈이 없어도 **v1.0 REST**(`v1.0/groupSettings`)로 읽을 수 있었다.

### 값 대신 형식 이름이 나오는 출력

교차 테넌트 액세스 기본 설정을 `Select-Object`로 찍으면 이렇게 나온다.

```
Microsoft.Graph.PowerShell.Models.MicrosoftGraphCrossTenantAccessPolicyB2BSetting
```

중첩 개체라서 `usersAndGroups.accessType`까지 점으로 들어가야 값(`allowed`/`blocked`)이 보인다. 그리고 이 정책에는 **`isServiceDefault`**라는 좋은 지표가 있었다. 한 번도 안 바꿨으면 `True`다.

### REST 호출은 첫 페이지만 온다

서비스 상태 이슈를 `?$top=100`으로 직접 불렀더니 미해결이 3건이었다. cmdlet의 `-All`로는 9건이었다. 내가 **`@odata.nextLink`를 따라가지 않아서** 나머지가 빠진 것이다. 검증하려다 내가 틀린 사례라 기록해 둔다.

---

## 4. 실행 환경 함정

### SharePoint Online 모듈 16.0.27709.12000: 접속 자체가 안 됨

```
경고: DLL 'msalruntime'을(를) 로드할 수 없습니다. (HRESULT: 0x8007007E)
Connect-SPOService : No valid OAuth 2.0 authentication session exists
```

처음에는 PowerShell 7 호환 문제라고 생각했다. 그런데 **Windows PowerShell 5.1에서도 똑같이** 실패했다. 찾아보니 이 버전의 **배포 결함**이었다. 로그인 브로커 DLL을 새로 넣으면서 거기에 필요한 `msalruntime.dll`을 패키지에서 빠뜨렸고, MS가 GitHub 이슈에서 확인했다.

해결은 이전 버전을 함께 설치하고 **새 창에서 첫 명령으로** 그 버전을 불러오는 것이다.

```powershell
Install-Module Microsoft.Online.SharePoint.PowerShell -RequiredVersion 16.0.27612.12000 -Scope CurrentUser -Force
# 새 Windows PowerShell 창에서, 다른 SPO 명령보다 먼저
Import-Module Microsoft.Online.SharePoint.PowerShell -RequiredVersion 16.0.27612.12000
Get-Module Microsoft.Online.SharePoint.PowerShell | Select-Object Name,Version   # 한 줄만 나와야 함
Connect-SPOService -Url "https://<테넌트>-admin.sharepoint.com"
```

같은 창에서 이미 새 버전이 자동으로 불러와졌다면, 이전 버전을 `-Force`로 불러와도 실패한다. .NET 어셈블리는 한 번 올라가면 창을 닫기 전까지 내려가지 않기 때문이다.

### Exchange Online과 Security & Compliance를 함께 접속할 때

`Connect-ExchangeOnline` 다음에 `Connect-IPPSSession`을 하면 **이름이 같은 cmdlet은 나중 세션이 덮어쓴다.** `Get-AdminAuditLogConfig`의 `UnifiedAuditLogIngestionEnabled`는 MS 문서에 "Security & Compliance PowerShell에서는 감사가 켜져 있어도 항상 False"라고 나와 있다. 그래서 **Exchange만 단독으로 다시 접속해서** 확인했다. 내 Sandbox는 그래도 False여서, 실제로 감사가 꺼져 있었다.

### 일반 PowerShell 창에서 저장이 안 됨

`Out-File`이 `Access denied`로 실패했다. 폴더 권한을 보니 `BUILTIN\Users`가 읽기만 가능했다. 예전에는 관리자 창에서 실행해서 몰랐던 것이다. 관리자 창에서 내 계정에 수정 권한을 한 번 주고 해결했다.

```powershell
icacls <작업폴더> /grant "${env:USERDOMAIN}\${env:USERNAME}:(OI)(CI)M"
```

### 콘솔에 붙여 넣는 `if / else`

```powershell
if ($x) { ... }
else { ... }      # 콘솔은 위 줄 }에서 if를 끝내고 실행해 버린다 → 'else'를 명령으로 인식 못 함
if ($x) { ... } else { ... }   # 같은 줄에 쓴다
```

---

## 5. 포털에서 확인한 것: 화면이 바뀌어 있었다

- **Microsoft 365 Apps 관리 센터**: 한국어 화면에서 "인벤토리(Inventory)"가 **"재고"**로 번역돼 있어서 한참 찾았다.
- **에이전트 관리**: 메뉴가 **Agent 365**로 바뀌어 있었다. 사용자 액세스 기본값은 **"모든 사용자"**였다.
- **릴리스 기본 설정**: 창 위에 **"대상 릴리스가 2027년 1월에 사용 중지됨"** 안내가 있었다. "파일럿 사용자는 대상 릴리스로" 같은 기준은 앞으로 표준·지연 릴리스 체계로 바꿔야 한다.
- **사용량 보고서**: 개발자 Sandbox에서는 포털도 "데이터 없음", Graph 보고서 API도 404(`UnknownTenantId`)였다. 원인은 공식 설명을 찾지 못했다.

---

## 6. Sandbox 기본값에서 눈에 띈 것

손대지 않은 테넌트의 기본값이 생각보다 넓게 열려 있었다.

| 영역 | 기본값 |
|---|---|
| SharePoint 기본 공유 링크 | **Anyone**(로그인 없이), **만료 없음**, 파일·폴더 **편집** |
| SharePoint 외부 공유 수준 | 가장 넓은 단계(`ExternalUserAndGuestSharing`) |
| Entra 게스트 초대 | 게스트 포함 **누구나**, 모든 도메인 허용 |
| Teams 외부 공유 채널 | 허용 — 그런데 Entra B2B 직접 연결이 **양방향 차단**이라 실제로는 동작 안 함 |
| 통합 감사 로그 | **꺼짐** |
| 서비스 상태 이메일 알림 | **꺼짐** |

Teams와 Entra의 조합처럼, **한 설정만 보면 판단이 틀린다.** 실제 동작은 여러 곳의 설정이 겹쳐서 정해진다.

---

## 정리

1. **빈 출력은 0이 아니다.** 상태 코드, 전체 개수, 실패 여부를 먼저 찍고, 측정이 안 되면 "측정 불가"라고 쓴다.
2. **정답을 아는 대상으로 검증한다.** 직접 만든 사용자·할당이 있으면 명령의 오차가 바로 보인다.
3. **범위와 주체를 나눠서 센다.** 구독 직접 / 하위 / 상속, 사람 / 앱.
4. **조용히 틀리는 곳을 의심한다.** `Select-Object`, 서버 필터의 null, REST 페이지, v1.0 대 베타.
5. **모듈 버전과 세션도 측정 조건이다.** SPO 모듈 결함, EXO·IPPS 덮어쓰기처럼 환경 때문에 값이 틀릴 수 있다.
6. **상태를 바꾸는 명령은 조회 스크립트에 섞지 않는다.** 보고서 생성(`Start-…`)이나 감사 켜기도 상태 변경이다.

### 참고 문서

- [Microsoft Entra cmdlets for configuring group settings](https://learn.microsoft.com/en-us/entra/identity/users/groups-settings-cmdlets)
- [Turn auditing on or off (Microsoft Purview)](https://learn.microsoft.com/en-us/purview/audit-log-enable-disable)
- [Connect-SPOService fails with "Unable to load DLL 'msalruntime'" in 16.0.27709.12000 (GitHub)](https://github.com/SharePoint/sp-dev-docs/issues/11061)
