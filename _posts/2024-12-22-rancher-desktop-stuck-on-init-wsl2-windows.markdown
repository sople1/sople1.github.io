---
layout: post
title:  "WSL2 환경에서 Rancher Desktop 실행 문제"
date:   2024-12-22 00:08:47 +0900
categories: [wsl2, wsl, 'windows subsystem for linux', docker, moby, mobyd, windows]
---

## 개요

Windows 10 WSL2 환경에서 Rancher Desktop 실행 시 Initializing Rancher Desktop 그래프는 움직이는데, 이 그래프가 몇분이고 지속되는 문제가 발생함

## 상황

Rancher Desktop 실행 시 WIndows (10~)에서는 Windows Subsystem for Linux (WSL)을 통해 Docker(mobyd)를 구동한다. 즉, 가상머신(VM)을 이용하기 때문에 이를 구동하고 WIndows에 Docker 엔진 링크를 연결해주어야만 Docker를 이용할 수 있다.

그런데, 특정 환경에서 Initializing Rancher Desktop Step에서 계속 멈춰있는 현상이 발생하였다. 이에 이를 해결하기 위해 여러 방법을 진행하였다.

1. rdctl factory-reset
    - Rancher Desktop 이미지를 처음부터 새로 만들도록 하는 것이다.
    - 동일하게 그래프가 계속 멈춰 있었다.
2. %userprofile%\.docker, %userprofile%\.kube, %userprofile%\.kubelr 폴더 삭제
    - 초기상태로 돌릴 목적이었다.
    - 근데 증상은 고쳐지지 않았다.
3. WSL2 이미지 수동 설치 후 진행
    - factory-reset만으로 초기 상태 그대로 돌릴 수 없다는 내용을 찾았고, 이를 해결하기 위해 직접 distro를 설치할 수 있는 방법을 찾아 실행하였다.
    - 물론 증상은 고쳐지지 않았다.

## 전개

다양한 방법을 시도하다가 Rancher Desktop 설치 시 번들로 포함된 다양한 유틸리티를 통해 해결 실마리를 잡을 수 있었다.

우선, C:\Program Files\Rancher Desktop\resources\resources\win32 (설치 위치) 내에 있는 파일은  앞서 설치를 별도로 시도한 distro, docker utility, wsl utility가 있었다.

그 중, internal 폴더 안에서 실행되는 파일 중 wsl-helper가 WSL2 Dockerd 와 연결해주는 역할임을 알 수 있었다.

(wsl-helper.exe docker-proxy serve)

그리고, 앞서 그래프가 멈춰있는 상황에서 이 유틸리티를 실행하자 (거짓말처럼) 그래프가 사라졌다.

즉, 실행 과정이 완료된 것이다.

물론, 이 유틸리티는 닫으면 WIndows 환경에서 Docker를 사용할 수 없다.

그렇다는 것은, 이 유틸리티를 어떻게든 실행시키기만 하면 정상 동작을 한다는 이야기인 것이다.

## 난관

그러나, 이 유틸리티를 원래 어떻게 실행했는지 스크립트를 뒤져보고 코드를 뒤져봐도 뭐가 나오지 않았다.

그렇게 뒤지던 와중에 %localappdata%\rancher-desktop\logs\background.log 를 확인했고, .kube\config 파일을 필요로 함을 알게 되었다.

물론 이 파일이 그저 필요해서 원래 위치인 %userprofile%\.kube\config 에 다음 내용대로 만들어주고 끝낼 수 있었다면 이 글은 여기서 끝났을 것이다.

(실제로 최종 단계는 이 파일을 생성하며 끝이 났다. Rancher Desktop은 프록시로 차단된 환경에서도 설치를 지원하고, kube를 설치 안해도 돌아는 가지만, 기본적으로 이 파일을 참조한다는 것을 알았다.)

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: rancher-desktop
    cluster:
      server: https://127.0.0.1:6443
      certificate-authority-data: <Random Strings>
      insecure-skip-tls-verify: false
users:
  - name: rancher-desktop
    user:
      client-certificate-data: <Random Strings>
      client-key-data: <Random Strings>
contexts:
  - name: rancher-desktop
    context:
      cluster: rancher-desktop
      name: rancher-desktop
      user: rancher-desktop
preferences: {}
current-context: rancher-desktop
```

하지만, 방금 확인한 로그 파일에서는 %userprofile%/.kube/config를 인식하지도 않았고, 엉뚱한 경로를 지정하며 파일을 읽을 수 없다는 오류만 반복 출력되고 있었다.

## 해결

결론적으로 앞서 이야기한 %userprofile%/.kube/config를  Rancher Desktop이 인식할 수 있기만 하면 된다. 하지만, 우선 왜 앞선 로그 파일에서 인식하지 못했는지를 알아보니 Rancher Desktop이 인식하는 .kube/config 파일의 위치가 예측과 달랐음을 알 수 있었다.

여러 문서를 살펴보면 %userprofile% 이 아닌 %home%을 참조하여 경로를 생성하다는 것을 알 수 있었다. 그런데 %home%이 %userprofile%과 다를 수도 있는 특이 환경이 존재할 수 있었다.

따라서 이를 인식할 수 있는 환경으로 만들어주면 되는데, kube는 %kubeconfig% 변수를 통해 실제 kube config 파일 위치를 지정할 수 있었다. 따라서 이를 %userprofile%/.kube/config 로 지정하여 추가하였다.

그리고 Rancher Desktop 은 factory-reset 하더라도 .kube/config를 새로 만들어주지는 않기 때문에 이를 만들어야 한다는 점이 중요했다.

이 파일이 없으면… background.log에서 wsl-helper 를 실행하는 단계까지 가지 못함을 알 수 있었다.

(왜 이런 구성인지는 아직도 이해 불가)

## 다음 과정

뭐긴 뭐야, Rancher Desktop 팀에 Report 후 어떤 반응이 있을지를 확인해야지.

하지만 이 해결법은 아직 공유되지 않았을 것이다.

왜냐면 이건 나도 못 찾다가 직접 알아낸 거라서…

## 그리고 진짜 다음 과정

node.js node_modules 읽는 시간 때문에 코드 실행을 못해서 http 서버가 늦게 뜨는 문제가 있음을 알게 되었다. macOS + colima 에서만 쓰다 보니 크게 못 느꼈는데, 가뜩이나 파일 읽기가 느린 Windows 10 + WSL2 환경에서 코드를 돌려보니 이 현상이 두드러지게 나타났다.

아니, 왜 js 덩어리 실행하는 데 파일 읽는 속도까지 영향을 주고 있는거야…
