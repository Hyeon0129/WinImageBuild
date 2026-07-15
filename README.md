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
├── profiles\                    # "템플릿으로 저장" 눌러서 만든 빌드 설정 프리셋들
├── runs\                        # 빌드 1회당 폴더 하나: 그때 쓴 config.ini, build.ps1, 전체 로그
└── AppData\                     # 앱 자체의 빌드 기록/실행 로그 (history.json, server.log)
```


## 사용 순서

1. `WinImageBuild.exe` 실행 (관리자 권한 승인)
2. **대시보드**에서 ISO/드라이버가 제대로 인식됐는지, 환경 점검(Hyper-V/가상 스위치)이 정상인지 확인
3. **새 빌드** 화면에서 ISO 선택 → "마운트 & 에디션 확인" → 원하는 에디션 선택 → 출력 파일명 지정 → 드라이버 경로 확인(기본값 그대로면 대부분 OK) → 빌드 시작
4. 하단 로그 패널에서 실시간으로 진행 상황 확인 (제목을 드래그하면 별도 창으로 뗄 수 있음). 잘못 시작했으면 "작업 중지"로 취소 가능
5. 빌드 하나에 보통 15~25분 정도 걸립니다. 끝나면 `BuildOutput\`에 완성된 qcow2가 생깁니다
6. 자주 쓰는 조합(같은 에디션 + 같은 드라이버)은 "템플릿으로 저장"해두면 다음부터 매번 값 다시 넣을 필요 없습니다

## 참고

- `runs\` 안에 쌓이는 로그/config.ini는 그때그때의 실행 기록입니다. 오래된 것들은 가끔 정리해주세요
- 이 exe는 실행되는 위치를 기준으로 모든 경로를 잡습니다. 즉 이 폴더를 통째로 다른 드라이브나 다른 PC로 옮겨도 그대로 동작합니다 (경로에 특정 사용자 계정 이름 같은 게 박혀있지 않습니다)
- 빌드 로직 자체(sysprep, 드라이버 주입 등)는 손대지 않았습니다 — 원본 windows-imaging-tools 동작 그대로이고, 이 프로그램은 그 위에 자동화/모니터링 UI만 얹은 것입니다
