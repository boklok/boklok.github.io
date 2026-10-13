---
layout: post
title: "M365·Azure 실습환경 점검기 - 개발자 Sandbox부터 Azure 구독 조회까지"
date: 2026-10-13 14:00:00 +0900
categories: [M365, Azure, EntraID]
tags: [m365, azure, entra-id, powershell, microsoft-graph, sandbox, purview]
---

# M365·Azure 실습환경 점검기

M365·Azure 운영 실습에 들어가기 전에 **개인 실습 환경이 제대로 준비됐는지** 점검하는 사전 과제를 했다. 할 일 자체는 단순했는데, 중간에 로그인 오류와 PowerShell 모듈 충돌을 연달아 만나서 기록으로 남겨둔다.

> 테넌트 도메인, 테넌트 ID, 구독 ID 같은 식별값은 `xxxx`로 가렸다.

## 과제 개요

| 단계 | 내용 |
|---|---|
| 1 | Microsoft 365 개발자 프로그램 가입 + Instant Sandbox 개설 |
| 2 | 관리자 계정 MFA 등록 |
| 3 | M365 서비스 플랜 조회 (Teams, Entra ID P2, Purview) |
| 4 | Sandbox 테넌트의 Azure 구독 조회 (테넌트 소속, QuotaId, SpendingLimit, vCPU, 역할) |
| 5 | 필수 PowerShell 모듈 설치 |
| 6 | Microsoft Learn 연습평가 응시 가능 여부 확인 |

핵심 조건은 하나였다. **"서비스 플랜과 Azure 조건은 추정하지 말고, 관리자 화면이나 조회 명령의 실제 결과로 작성할 것."**

---

## 먼저 잡고 갈 개념: 테넌트와 구독

이 두 단어만 정리되면 나머지가 다 쉬워진다.

- **테넌트(Tenant)**: Microsoft 클라우드 안의 "우리 조직 전용 공간". 사용자, 그룹, 라이선스, 보안 정책이 들어 있고 Entra ID가 관리한다. 온프레미스로 치면 **AD 포리스트 하나**에 가깝다.
- **구독(Subscription)**: Azure 리소스(VM, 스토리지 등)를 만들고 **요금을 청구받는 단위**. 구독은 반드시 하나의 테넌트에 속한다.

```
테넌트 xxxx.onmicrosoft.com (Entra ID: 사용자 · 라이선스 · MFA)
 ├─ M365 라이선스: DEVELOPERPACK_V2_E5 → Teams, Entra P2, Purview …
 └─ Azure 구독: devProgram_… → VM, 네트워크 등 리소스
```

점검 항목이 M365 쪽(3단계)과 Azure 쪽(4단계)으로 나뉜 이유가 이 구조 때문이다.

---

## 1. 개발자 프로그램과 Instant Sandbox

**Microsoft 365 개발자 프로그램**은 개발자나 파트너에게 실습용 테넌트를 주는 프로그램이다. **Instant Sandbox**를 고르면 E5 라이선스와 샘플 사용자 16명 정도가 미리 들어간 테넌트가 바로 만들어진다.

가입할 때 "주 개발 목적"과 "관심 분야"를 묻는 설문이 나오는데, 이건 MS가 가입자를 분류하는 프로필 용도라서 **Sandbox 구성에는 영향이 없다.** 나는 Personal projects를 고르고, 관심 분야는 이번 과정과 관련 있는 Graph, Identity platform, Teams 쪽을 체크했다.

## 2. MFA와 보안 기본값

- **MFA**: 비밀번호 + 휴대폰 승인처럼 두 가지 이상으로 본인을 확인하는 방식.
- **보안 기본값(Security Defaults)**: 새 테넌트에 자동으로 켜지는 기본 보안 세트. 모든 사용자에게 MFA를 요구하고, 위험한 로그인 방식을 막는다. 조건부 액세스(P1/P2)의 무료 간이판이라고 보면 된다.

새 Sandbox 관리자로 처음 로그인하면 Authenticator 등록을 바로 요구한다. 등록하고 나면 이후 로그인마다 MFA 승인이 뜬다.

---

## 3. M365 서비스 플랜 조회 (Graph PowerShell)

### SKU와 서비스 플랜

라이선스는 **상자와 내용물** 구조다.

- **SKU**: 라이선스 상품 하나(상자). 예: `DEVELOPERPACK_V2_E5`
- **서비스 플랜**: 상자 안의 개별 기능. 예: `TEAMS1`, `AAD_PREMIUM_P2`, `MIP_S_CLP2`
- **ProvisioningStatus = Success**: 그 기능이 테넌트에서 실제로 활성화까지 끝났다는 뜻

### 조회 명령

빠르게 하려고 Graph 전체가 아니라 필요한 하위 모듈만 먼저 설치했다.

```powershell
Install-Module Microsoft.Graph.Authentication, Microsoft.Graph.Identity.DirectoryManagement -Scope CurrentUser -Repository PSGallery

Connect-MgGraph -TenantId xxxx.onmicrosoft.com -Scopes Organization.Read.All -NoWelcome
```

`-TenantId`에는 **계정 주소가 아니라 테넌트 도메인**을 넣어야 한다. 처음에 `admin@xxxx.onmicrosoft.com`을 넣었다가 `Invalid tenant id provided`를 받았다.

```powershell
# 테넌트 초기 도메인
(Get-MgOrganization).VerifiedDomains | ? IsInitial | select Name

# 보유 SKU
Get-MgSubscribedSku | select SkuPartNumber, ConsumedUnits, @{n='Enabled';e={$_.PrepaidUnits.Enabled}} | ft

# 서비스 플랜 펼치기
$plans = Get-MgSubscribedSku | % { $s = $_.SkuPartNumber; $_.ServicePlans | select @{n='Sku';e={$s}}, ServicePlanName, ProvisioningStatus }

# Teams / Entra ID P2
$plans | ? ServicePlanName -in 'TEAMS1','AAD_PREMIUM_P2' | ft

# Purview 계열
$plans | ? ServicePlanName -match 'MIP_S_CLP|ADVANCED_AUDITING|EQUIVIO|INSIDER_RISK|COMMUNICATIONS_|RECORDS_MANAGEMENT|INFO_GOVERNANCE|DATA_INVESTIGATIONS|LOCKBOX' | ft
```

### 결과

```
SkuPartNumber       ConsumedUnits Enabled
-------------       ------------- -------
DEVELOPERPACK_V2_E5            17      25

Sku                 ServicePlanName ProvisioningStatus
---                 --------------- ------------------
DEVELOPERPACK_V2_E5 TEAMS1          Success
DEVELOPERPACK_V2_E5 AAD_PREMIUM_P2  Success
```

Purview 쪽은 `MIP_S_CLP1/2`(정보 보호), `M365_ADVANCED_AUDITING`(고급 감사), `EQUIVIO_ANALYTICS`(eDiscovery Premium), `INSIDER_RISK`(내부자 위험 관리), `COMMUNICATIONS_COMPLIANCE`, `RECORDS_MANAGEMENT` 등 12개 플랜이 전부 Success였다. Purview는 단일 플랜이 아니라 여러 기능의 묶음이라서 과제에서도 "Purview **계열**"이라고 표현한 것이다.

라이선스 25개 중 17개가 사용 중인 건 Instant Sandbox에 미리 들어 있는 샘플 사용자 몫이다.

---

## 4. Azure 구독 조회 (Az PowerShell)

```powershell
Install-Module Az.Accounts, Az.Resources, Az.Compute -Scope CurrentUser -Repository PSGallery
Update-AzConfig -EnableLoginByWam $false
Connect-AzAccount -Tenant xxxx.onmicrosoft.com
```

로그인 후 점검 항목 5개를 한 번에 뽑는 블록:

```powershell
Get-AzTenant | select Id, Name, @{n='Domains';e={$_.Domains -join ', '}} | fl

$sub   = Get-AzSubscription -TenantId xxxx.onmicrosoft.com | select -First 1
Set-AzContext -Subscription $sub.Id | Out-Null
$pol   = ((Invoke-AzRestMethod -Path "/subscriptions/$($sub.Id)?api-version=2022-12-01").Content | ConvertFrom-Json).subscriptionPolicies
$cores = Get-AzVMUsage -Location koreacentral | ? { $_.Name.Value -eq 'cores' }
$me    = (Get-AzADUser -SignedIn).Id
$roles = Get-AzRoleAssignment -Scope "/subscriptions/$($sub.Id)" | ? ObjectId -eq $me

[pscustomobject]@{
  구독이름      = $sub.Name
  QuotaId       = $pol.quotaId
  SpendingLimit = $pol.spendingLimit
  vCPU          = "$($cores.Limit) (사용 $($cores.CurrentValue)) / koreacentral"
  역할          = ($roles | % { "$($_.RoleDefinitionName) @ $($_.Scope)" }) -join '; '
} | fl
```

`QuotaId`와 `SpendingLimit`은 `Get-AzSubscription` 출력에서 바로 보이지 않아서, ARM REST API(`subscriptionPolicies`)를 직접 호출해서 가져왔다.

### 결과와 의미

| 항목 | 결과 | 의미 |
|---|---|---|
| 테넌트 소속 | Sandbox 테넌트와 일치 | 구독이 내 Sandbox 테넌트에 붙어 있음 |
| QuotaId | `PayAsYouGo_2014-09-01` | **종량제** 구독 (무료 체험판이면 `FreeTrial_…`) |
| SpendingLimit | `Off` | **지출 한도 없음** → 리소스를 켜두면 계속 과금 |
| vCPU 할당량 | `10 (사용 0)` | Korea Central에서 동시에 켤 수 있는 최대 코어 수 |
| 역할 | `Owner` | 구독 안의 리소스와 권한을 모두 관리하는 최고 권한 |

몇 가지 짚어둘 것:

- **QuotaId + SpendingLimit 조합**: `PayAsYouGo` + `Off`면 크레딧이 소진되면 멈추는 체험판이 아니다. 실습 후에는 VM 같은 리소스를 반드시 정리해야 한다.
- **vCPU 할당량**: 2코어 VM이라면 5대까지 띄울 수 있다는 뜻이다. 비용 폭주를 막는 안전장치이기도 하고, 늘리려면 할당량 증가 요청을 해야 한다.
- **Entra 역할 ≠ Azure 역할**: 전역 관리자(Entra 역할)라고 해서 자동으로 구독의 Owner(Azure RBAC 역할)가 되는 게 아니다. 두 권한 체계는 별개라서 이 항목을 따로 확인한다.

### 리소스 공급자 등록

처음 조회했을 때 vCPU만 빈 값으로 나왔다. 원인은 **`Microsoft.Compute` 리소스 공급자가 등록되지 않은 새 구독**이었기 때문이다. Azure 서비스는 구독마다 "이 서비스를 쓰겠다"고 활성화해야 한다.

```powershell
Get-AzResourceProvider -ProviderNamespace Microsoft.Compute | select ProviderNamespace, RegistrationState -First 1
# NotRegistered

Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
# Registering → 잠시 후 다시 조회하면 Total Regional vCPUs 10
```

---

## 5. 필수 PowerShell 모듈

```powershell
Install-Module Microsoft.Graph, Az, MicrosoftTeams, Microsoft.Online.SharePoint.PowerShell -Scope CurrentUser -Repository PSGallery
Get-InstalledModule Microsoft.Graph, Az, MicrosoftTeams, Microsoft.Online.SharePoint.PowerShell | select Name, Version | ft
```

| 모듈 | 다루는 대상 | 설치 버전 |
|---|---|---|
| Microsoft.Graph | M365·Entra 전반 (사용자, 라이선스, 그룹) | 2.41.1 |
| Az | Azure 구독과 리소스 | 16.4.0 |
| MicrosoftTeams | Teams 정책과 팀 관리 | 8.0.0 |
| Microsoft.Online.SharePoint.PowerShell | SharePoint 사이트와 테넌트 설정 (**Windows 전용**) | 16.0.27709.12000 |

## 6. Microsoft Learn 연습평가

개인 MS 계정으로 Learn 프로필에 로그인하고, 시험 페이지(예: `learn.microsoft.com/ko-kr/credentials/certifications/exams/ms-102/`)에서 **연습 평가 시작**을 누르면 50문항 모의고사가 열린다. 문제마다 정답과 해설을 바로 볼 수 있어서 공부용으로도 좋다.

---

## 트러블슈팅 모음

오늘 시간을 가장 많이 쓴 부분.

### ① WAM 오류: `A window handle must be configured`

```
Connect-MgGraph : InteractiveBrowserCredential authentication failed:
A window handle must be configured.
```

Graph와 Az 모듈은 Windows에서 기본으로 **WAM(Web Account Manager)** 로그인을 쓰는데, 일부 PowerShell 창에서는 로그인 팝업을 띄우지 못한다.

```powershell
Set-MgGraphOption -DisableLoginByWAM $true   # Graph
Update-AzConfig -EnableLoginByWam $false     # Az
```

Graph 쪽 설정은 **새 PowerShell 창부터** 적용됐다. 같은 창에서 바로 재시도하면 똑같은 오류가 난다.

### ② 디바이스 코드 로그인 차단: `530035`

WAM을 피하려고 `-UseDeviceCode`로 로그인했더니 브라우저에 "이에 대한 액세스 권한이 없습니다"가 떴다. 메시지만 보면 권한 문제 같지만, Error Code **530035**는 **보안 기본값이 이 로그인을 차단했다**는 뜻이다. 디바이스 코드 흐름은 피싱에 악용되기 쉬워서 위험한 로그인 방식으로 분류된다.

보안 기본값을 끄는 건 MFA 요구사항과 충돌하므로, 새 창에서 브라우저 로그인으로 우회했다.

### ③ Graph와 Az를 한 창에서 쓰면 생기는 충돌

```
'Azure.Core, Version=1.51.1.0 ...' 어셈블리에서
'Azure.Identity.SharedTokenCacheCredentialOptions' 형식을 로드할 수 없습니다.
```

```
Get-MgSubscribedSku : 메서드를 찾을 수 없습니다.
'Void Microsoft.Graph.Authentication.AzureIdentityAccessTokenProvider..ctor(Azure.Core.TokenCredential, ...)'
```

Graph와 Az는 내부적으로 같은 라이브러리(**Azure.Core**)를 서로 다른 버전으로 쓴다. 한 PowerShell 세션에서는 먼저 로드된 버전만 쓸 수 있어서, **나중에 로드한 쪽이 깨진다.**

- Az를 먼저 쓴 창에서 Graph를 실행 → Graph 실패
- Graph를 먼저 쓴 창에서 Az를 실행 → Az 실패

오류가 엉뚱한 곳에서 난다. `$sub`가 비면서 `Set-AzContext`가 null 인수 오류를 내고, 그 뒤로 `Get-AzRoleAssignment : Scope '/subscriptions' should have even number of parts.`까지 연쇄적으로 터진다. 근본 원인은 첫 줄의 어셈블리 경고 하나다.

**해결: Graph 창과 Az 창을 분리한다.** Windows PowerShell 5.1에서 특히 잘 생기고, PowerShell 7에서는 훨씬 덜하다. 앞으로 실습은 PowerShell 7로 할 생각이다.

모듈 버전이 섞였는지 의심될 때는 이걸로 확인할 수 있다.

```powershell
Get-Module Microsoft.Graph.Authentication, Microsoft.Graph.Identity.DirectoryManagement -ListAvailable | select Name, Version
Get-Module Az.Accounts   # 현재 세션에 Az가 로드돼 있는지
```

### ④ 출력이 빈 줄로 나오는 현상

두 명령을 한꺼번에 붙여넣었더니 두 번째 명령 결과가 빈 줄로만 나왔다. PowerShell이 여러 결과를 **첫 번째 객체의 표 형식에 맞춰** 출력해서, 속성이 다른 두 번째 결과가 비어 보인 것이다. 끝에 `| ft`를 붙이거나 한 줄씩 실행하면 된다.

### ⑤ 붙여넣기 실수

명령을 두 번 붙여넣어서 `| ftGet-AzVMUsage`처럼 이어진 적이 있다. 주소창에 URL을 두 번 붙여넣어 404가 난 적도 있다. 결과가 이상하면 입력부터 다시 보자.

---

## 정리

오늘 과제는 결국 **"실습할 테넌트가 있고, 관리자로 안전하게 들어갈 수 있고, 필요한 라이선스와 Azure 자원이 준비됐고, 다룰 도구가 설치됐는지"**를 실제 조회로 확인하는 일이었다.

가장 크게 남은 교훈 두 가지:

1. **메시지보다 코드를 본다.** "권한이 없습니다"는 권한 문제가 아니라 보안 기본값의 로그인 방식 차단(530035)이었다.
2. **Graph와 Az는 창을 나눈다.** 첫 줄의 `Azure.Core` 경고 하나가 뒤따르는 모든 오류의 원인이었다.
