---
layout: post
title: "Teams 실습기 - 채팅 파일과 회의 녹화는 누구의 저장소에 들어갈까"
date: 2026-10-21 12:00:00 +0900
categories: [M365, Teams]
tags: [teams, sharepoint, onedrive, exchange, meeting-recording, m365-group, microsoft-graph, powershell]
---

# Teams 실습기 - 채팅 파일과 회의 녹화는 누구의 저장소에 들어갈까

개인 M365 개발자 Sandbox에서 Teams 저장 구조를 이어서 실습했다. [지난번]({% post_url 2026-10-14-dynamic-group-licensing-teams-storage %})에는 채널 3종(표준·비공개·공유)의 파일 위치를 확인했다. 이번에는 남은 질문 네 가지를 직접 만들어 보고 Graph로 추적했다.

1. Team 하나를 만들면 뒤에서 무엇이 생기나
2. 1:1 채팅으로 보낸 파일은 어디에 저장되나
3. 회의 녹화는 어디에 저장되나 (채널 회의 / 일반 회의)
4. Teams에서 잡은 회의는 어디에 기록되나

> 테넌트 이름·ID·회의 링크는 `xxxx`로 가렸다. 사용자는 실습용 계정이다.

---

## 1. Team 하나 = 그룹 + 사서함 + 사이트

지난번에 만든 팀을 Graph로 다시 봤다.

```powershell
$g = Get-MgGroup -GroupId $TeamId -Property DisplayName,Mail,ProxyAddresses,GroupTypes,ResourceProvisioningOptions,Visibility
Invoke-MgGraphRequest -Method GET -Uri "v1.0/groups/$TeamId/calendar"
Invoke-MgGraphRequest -Method GET -Uri "v1.0/groups/$TeamId/sites/root?`$select=webUrl"
```

```
[① M365 그룹]   유형=Unified | Teams 연결=Team | 공개범위=Private
[② 그룹 사서함] 주소 <팀>@xxxx.onmicrosoft.com | Calendar 있음 (소유자 = 그룹 주소)
[③ SharePoint]  https://xxxx.sharepoint.com/sites/<팀>
```

Teams에서는 팀 하나로 보이지만, 실제로는 **M365 그룹(구성원 명단) + 그룹 사서함(일정·대화) + SharePoint 사이트(파일)** 세 개가 함께 만들어진다.

그룹의 `proxyAddresses`에 이런 값도 있었다.

```
SPO:SPO_<사이트 ID>@SPO_<테넌트 ID>
```

그룹 개체 안에 "**내 파일은 이 SharePoint 사이트에 있다**"는 연결 정보가 들어 있는 셈이다.

---

## 2. 클라이언트에서 직접 만들기

Teams 웹을 InPrivate 창에서 열고 실습 계정으로 로그인했다. 평소 쓰는 Teams와 섞이지 않게 하려는 것이다. 그리고 네 가지를 만들었다.

| | 작업 |
|---|---|
| A | 실습 사용자와 1:1 채팅을 열고 `chat-file.txt` 파일 전송 |
| B | 표준 채널에서 **지금 모임** → 녹화 15초 → 중지 |
| C | 일정에서 **지금 모임**(채널 밖) → 녹화 15초 → 중지 |
| D | 일정에서 **새 모임** 예약 (다음 날 18:00) |

녹화 파일은 회의가 끝나고 몇 분 뒤에 생긴다.

---

## 3. Graph로 추적한 결과

```powershell
# A: 보낸 사람 OneDrive에서 파일명으로 검색
(Invoke-MgGraphRequest -Method GET -Uri "v1.0/me/drive/root/search(q='chat-file')").value

# B: 팀 드라이브에서 mp4 검색 → items/{id}로 parentReference 확인
(Invoke-MgGraphRequest -Method GET -Uri "v1.0/groups/$TeamId/drive/root/search(q='mp4')").value

# C: 내 OneDrive /Recordings
(Invoke-MgGraphRequest -Method GET -Uri "v1.0/me/drive/root:/Recordings:/children").value

# D: Exchange 일정
(Invoke-MgGraphRequest -Method GET -Uri "v1.0/me/events?`$filter=subject eq '<회의 제목>'").value
```

| | 결과 |
|---|---|
| A 채팅 파일 | `xxxx-my.sharepoint.com/personal/<보낸 사람>/Documents/Microsoft Teams 채팅 파일/chat-file.txt` |
| B 채널 회의 녹화 | `xxxx.sharepoint.com/sites/<팀>/Shared Documents/<채널>/Recordings/…mp4` |
| C 일반 회의 녹화 | `xxxx-my.sharepoint.com/personal/<녹화한 사람>/Documents/Recordings/…mp4` |
| D 예약 회의 | `/me/events`에 존재, `isOnlineMeeting=True`, `onlineMeetingProvider=teamsForBusiness` |

주소가 `-my.sharepoint.com/personal/…`로 시작하면 **개인 OneDrive**, `sharepoint.com/sites/…`로 시작하면 **팀 SharePoint**다.

### 지난번 결과와 합친 Teams 저장 위치

| 어디서 만든 파일 | 실제 저장 위치 | 주인 |
|---|---|---|
| 표준 채널 | 팀 사이트 / Shared Documents / 채널 폴더 | 팀 |
| 비공개 채널 | 별도 사이트 (팀명-채널명) | 채널 |
| 공유 채널 | 별도 사이트 | 채널 |
| 1:1 채팅 파일 | **보낸 사람 OneDrive** / Microsoft Teams 채팅 파일 | **개인** |
| 채널 회의 녹화 | 팀 사이트 / 채널 폴더 / Recordings | 팀 |
| 일반 회의 녹화 | **녹화한 사람 OneDrive** / Recordings | **개인** |
| 예약 회의 | Exchange 사서함 일정 | 사서함 |

---

## 4. 실습하다 걸린 함정 두 개

### 폴더 이름이 한국어였다

처음에는 영어 폴더 이름으로 조회했다.

```powershell
Invoke-MgGraphRequest -Method GET -Uri "v1.0/me/drive/root:/Microsoft Teams Chat Files:/children"
# NotFound (404)
```

실제 폴더 이름은 **`Microsoft Teams 채팅 파일`**이었다. OneDrive 언어에 따라 폴더 이름이 달라지는 것으로 보인다. 그래서 폴더 경로를 적어 넣는 대신 **파일명 검색 → `items/{id}`의 `parentReference.path` 확인** 방식으로 바꿨다.

### 시간이 9시간 다르게 나왔다

18:00으로 잡은 회의가 Graph에서는 `09:00`으로 나왔다. Graph는 기본으로 **UTC**를 돌려준다. 09:00 UTC = 18:00 KST이니 맞는 값이다. 스크립트 결과를 보고서에 옮길 때는 시간대를 꼭 표시해야 한다.

---

## 5. 운영에서는 어떻게 쓰일까

**"그 파일은 누구 것인가?"**를 먼저 물으면 볼 곳이 정해진다.

- **퇴사자 처리**: 채팅으로 보낸 파일과 직접 녹화한 회의는 **그 사람 OneDrive**에 있다. 계정을 지우기 전에 OneDrive의 `Microsoft Teams 채팅 파일`과 `Recordings`를 먼저 확인해야, 상대방이 받은 파일과 회의 녹화를 잃지 않는다.
- **"녹화가 어디 있어요?"**: 채널에서 열린 회의면 팀 사이트, 아니면 **녹화 버튼을 누른 사람**의 OneDrive다. 주최자가 아니다.
- **"비공개 채널 권한이 이상해요"**: 비공개·공유 채널은 별도 사이트라 팀 사이트 권한과 따로 움직인다. 팀 하나가 사이트 여러 개다.
- **"Teams에 일정이 안 떠요"**: Teams 일정은 Exchange 일정을 보여 주는 창이다. 먼저 Exchange 서비스 플랜과 사서함을 본다.
- **보존·보안 정책**: Teams 파일을 지키려면 SharePoint(채널), **OneDrive(채팅·일반 회의 녹화)**, Exchange(일정)를 모두 범위에 넣어야 한다.

---

## 정리

1. **Team = M365 그룹 + 그룹 사서함 + SharePoint 사이트.** 그룹과 사이트의 연결값은 `proxyAddresses`에 있다.
2. **채팅 파일과 일반 회의 녹화는 개인 OneDrive**, 채널 파일과 채널 회의 녹화는 팀 SharePoint에 저장된다.
3. **Teams 회의 = Exchange 일정 이벤트**다.
4. 폴더 이름은 언어에 따라 다를 수 있으니 **파일명으로 검색**한다.
5. Graph 시간은 **UTC**다.
