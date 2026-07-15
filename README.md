## WinImageBuild

Windows 이미지(qcow2)를 Ironic/Bifrost 배포용으로 만들 때 매번 반복하던 수동 작업 ISO 마운트, config.ini 수정, PowerShell 등등을 자동화한 도구입니다. 
[cloudbase/windows-imaging-tools](https://github.com/cloudbase/windows-imaging-tools) 


### 사전 준비

- **Window ADK 설치** : ADK 폴더 내에 있음
- **Hyper-V 활성화** : 제어판 → 프로그램 → Windows 기능 켜기/끄기 → Hyper-V 체크 후 재부팅 / Hyper-V, Hyper-V 관리 도구, Hyper-V 플랫폼 체크
```powershell
# 이더넷 2는 각 시스템에서 사용가능한 인터페이스 이름
New-VMSwitch -Name "ExternalSwitch" -NetAdapterName "이더넷 2" -AllowManagementOS $true

# ExternalSwitch 생성 확인
PS C:\WINDOWS\system32> Get-VMSwitch

Name           SwitchType NetAdapterInterfaceDescription
----           ---------- ------------------------------
ExternalSwitch External   Realtek USB GbE Family Controller
Default Switch Internal
```

- **외부 가상 스위치 설정** 호스트와 VM 통신용


## 폴더 구조

```
WinImageBuild\ (=dist, 이 폴더 자체가 배포 단위입니다)
├── WinImageBuild.exe          # 실행 파일. 관리자 권한으로 실행됩니다.
├── SourceISO\                 # 빌드에 쓸 Windows 설치 ISO + VirtIO 드라이버 ISO를 여기 넣으세요
│   ├── virtio-win.iso          #   (VM용 드라이버, 기본 제공)
│   └── Windows 10 22H2 ...iso  #   (설치 미디어, 직접 준비)
├── BuildOutput\                # 빌드 끝나면 완성된 .qcow2가 여기 쌓입니다 (처음엔 비어있음)
├── drivers\
│   └── lan\                    # 이미지에 슬립스트림할 추가 드라이버(랜카드 등). 서버 기종 바뀌면 여기만 교체
│       ├── PRO1000\
│       ├── PRO2500\
│       ├── PRO40GB\
│       ├── PROCGB\
│       └── PROXGB\
├── CloudbaseInit\              # 손대지 않아도 되는 고정 파일들
│   ├── CloudbaseInitSetup_1_1_8_x64.msi
│   ├── cloudbase-init.conf
│   └── cloudbase-init-unattend.conf
├── ADK\                         # Windows ADK 설치 프로그램 (환경 점검의 "설치" 버튼이 이걸 실행함)
│   └── adksetup.exe
├── windows-imaging-tools\      # 실제 빌드 엔진 (cloudbase 원본 그대로, 위 GitHub 링크 참고)
│   ├── bin\
│   ├── docs\
│   ├── Examples\
│   ├── Tests\
│   ├── UnattendResources\      # sysprep 중 게스트 안에서 도는 스크립트, Unattend 리소스
│   ├── WinImageBuilder.psm1
│   ├── Config.psm1
│   ├── UnattendTemplate.xml    # OOBE 자동화 답변 파일 (관리자 비번은 CHANGE_ME로 되어있음, 필요시 교체)
│   └── config.ini              # 참고용 원본 템플릿 (앱이 실제로 쓰는 건 runs\ 안에 매번 새로 생성됨)
├── profiles\                    # (예약됨, 현재 UI에는 없는 기능) 빌드 설정 프리셋 저장 위치
├── runs\                        # 빌드 1회당 폴더 하나: 그때 쓴 config.ini, build.ps1, 전체 로그
└── AppData\                     # 앱 자체의 빌드 기록/실행 로그 (history.json, server.log)
```

## 폴더별 역할

- **SourceISO\**: 빌드에 쓸 Windows 설치 ISO와 VirtIO 드라이버 ISO를 두는 곳. 새 빌드 창의 ISO 목록은 이 폴더를 스캔해서 채워집니다.
- **BuildOutput\**: 빌드가 끝나면 완성된 `.qcow2` 파일이 저장되는 곳.
- **drivers\lan\**: 이미지에 슬립스트림할 추가 드라이버(랜카드 등). 서버 기종이 바뀌면 이 폴더만 교체하면 됩니다.
- **CloudbaseInit\**: Cloudbase-Init 설치 파일/설정. 특별한 사정이 없으면 그대로 둡니다.
- **ADK\**: Windows ADK 설치 프로그램. 환경 점검에서 ADK 미설치로 나오면 "설치" 버튼을 눌러 이 파일을 실행합니다(자동 실행 안 됨).
- **windows-imaging-tools\**: 실제 이미지 빌드를 수행하는 엔진(cloudbase 원본). `config.ini`는 참고용 원본 템플릿이고, 실제 빌드 설정은 매번 `runs\<빌드ID>\config.ini`로 새로 생성됩니다.
- **profiles\**: 빌드 설정 프리셋을 저장하기 위해 예약해 둔 폴더 — 현재 UI에는 해당 기능이 없습니다.
- **runs\**: 빌드 1회당 폴더가 하나씩 생기고, 그때 쓰인 `config.ini`/`build.ps1`/전체 로그가 남습니다. 오래된 것들은 가끔 정리해주세요.
- **AppData\**: 앱 자체의 실행 기록(`history.json`)과 서버 로그(`server.log`) — 이 PC에서만 의미 있는 데이터입니다.

## 사용 방법

1. `WinImageBuild.exe` 실행 (관리자 권한 승인)
2. 첫 화면의 **환경 점검** 카드에서 Hyper-V / Windows ADK / 외부 가상 스위치 상태를 확인합니다. 준비가 안 된 항목만 "설치"/"활성화" 버튼이 나타나며, 버튼을 눌러야만 설치·활성화가 실행됩니다(자동 실행 없음)
3. 우측 상단 **새 빌드** 버튼 → ISO 선택 → "마운트 & 에디션 확인" → 원하는 에디션 선택 → 출력 경로/CPU·RAM·디스크 크기 지정 → 드라이버 경로 확인(기본값 그대로면 대부분 OK) → 빌드 시작
4. 빌드는 여러 건을 동시에 진행할 수 있습니다. **빌드 현황** 표에서 각 항목의 진행률(%)·경과 시간·크기가 실시간으로 갱신되고, "상세" 버튼을 누르면 그 빌드만의 실시간 로그를 볼 수 있습니다
5. 잘못 시작했으면 해당 항목의 "중지" 버튼으로 즉시 취소할 수 있습니다 — 취소하면 생성 중이던 Hyper-V VM과 미완성 이미지 파일이 자동으로 정리됩니다
6. 빌드 하나에 보통 15~25분 정도 걸립니다. 완료·실패·취소된 항목은 5초 뒤 표에서 자동으로 사라지고, 완성된 이미지는 `BuildOutput\`에 남습니다

이 exe는 실행되는 위치를 기준으로 모든 경로를 잡습니다 — 폴더째로 다른 드라이브나 다른 PC로 옮겨도 그대로 동작합니다(경로에 특정 사용자 계정 이름이 박혀있지 않습니다). 빌드 로직 자체(sysprep, 드라이버 주입 등)는 손대지 않았고, 원본 windows-imaging-tools 동작 그대로이며 이 프로그램은 그 위에 자동화/모니터링 UI만 얹은 것입니다.
