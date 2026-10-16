# EKS에서 Graceful Shutdown 하는 방법(eks-graceful-shutdown.md)
Kubernetes에서 Pod는 일시적인(ephemeral) 엔티티로 간주되고 다양한 이유로 종료할 수 있으며 **Pod의 종료는 예외 상황이 아니라 Kubernetes의 정상적인 동작**입니다.

**Pod가 종료될 때 즉시 프로세스를 종료하면 처리 중인 요청(in-flight requests)이 강제로 중단되면서 Timeout이나 5xx Error가 발생**할 수 있습니다.
애플리케이션이 종료를 원활하게 처리하는 것은 안정적인 시스템을 구축하고 훌륭한 사용자 경험을 제공하는 데 핵심적인 요소입니다.
그렇기 때문에 **Pod의 종료 과정에서 애플리케이션이 새로운 요청을 받지 않도록 하고, 이미 처리 중인 요청을 정상적으로 완료한 뒤 데이터베이스 연결 등의 리소스를 정리하고 종료해야합니다.**

이러한 종료 방식을 **Graceful Shutdown**이라고 합니다.

Pod 삭제 요청이 Kubernetes API Server로 전달되면 API Server는 Pod를 삭제 대상으로 표시합니다.
이때 **kubelet은 해당 Pod의 컨테이너 종료 절차를 시작**하고, **EndpointSlice Controller는 종료 중인 Pod가 트래픽 대상에서 제외될 수 있도록 EndpointSlice의 상태를 변경**합니다.

# Graceful Shutdown을 구성하는 방법
## 애플리케이션에서 SIGTERM 처리하기
**Kubelet은 Container Runtime을 통해 각 컨테이너의 메인 프로세스(PID 1 프로세스)에 일반적으로 `SIGTERM`을 전달**합니다.
Kubernetes는 애플리케이션이 처리 중인 요청과 같은 내부 상태를 알 수 없기 때문에, **애플리케이션은 SIGTERM을 받으면 처리 중인 요청과 사용 중인 자원을 정리한 뒤 스스로 종료해야합니다.**
만약 SIGTERM을 무시하거나 Grace Period가 만료될 때까지 컨테이너가 종료하지 못하면 Container Runtime이 SIGKILL을 전달하여 프로세스를 강제로 종료합니다.

## preStop Hook 사용하기
**`preStop`은 컨테이너에 SIGTERM을 전달하기 전에 실행되는 Kubernetes Lifecycle Hook**입니다. SIGTERM으로 종료 처리를 시작할 수 없는 경우, preStop에서 애플리케이션의 종료 API나 명령을 호출할 수 있습니다.
또한, **preStop은 EndpointSlice 변경 사항이 kube-proxy나 Load Balancer까지 전파되기 전에 컨테이너가 먼저 종료되는 것을 방지하기 위해 일정 시간 대기하는 용도로 사용**할 수 있습니다. 이를 통해 종료 중인 Pod가 트래픽 대상에서 제외될 시간을 확보하고, 종료 과정에서 발생 할 수 있는 Timeout이나 5xx Error를 줄일 수 있습니다.

## terminationGracePeriodSeconds 설정하기
**terminationGracePeriodSeconds(Grace Period)는 Kubernetes가 Pod의 정상 종료를 기다리는 최대 시간**으로 기본값은 30초입니다.이 시간이 만료되었는데도 컨테이너가 실행 중이라면 SIGKILL을 전달하여 컨테이너를 강제로 종료합니다.

Grace Period는 SIGTERM이 전달되는 시점이 아니라 preStop이 실행되기 전부터 계산됩니다. 이때 preStop 실행 시간과 애플리케이션 종료 시간의 합이 terminationGracePeriodSeconds보다 길면 애플리케이션이 정리 작업을 완료하기 전에 SIGKILL에 의해 강제로 종료될 수 있습니다.

따라서 **Grace Period는 preStop 실행 시간과 최대 in-flight request 처리 시간, Connection 및 자원 정리 시간을 모두 고려하여 산정**되어야 합니다.

## EKS ELB Deregistration Delay 설정하기
Pod가 삭제로 EndpointSlice가 변경되면 AWS Load Balancer Controller는 이를 감지하고 **`target-type: ip` 라면 해당 Pod IP를 Target Group에서 Deregister**합니다. 만약 **Target이 즉시 Deregister되면 처리 중인 in-flight requests가 완료되기 전에 연결이 중단되어 Timeout이나 5xx Error가 발생**할 수 있습니다.
따라서 **Deregistration Delay를 설정하여 Target을 일정 시간 `draining` 상태로 유지하고, 새로운 요청은 전달하지 않으면서 기존 요청을 정상적으로 완료할 시간을 확보**해야합니다.

반면 `target-type: instance`에서는 ELB의 Target이 Worker Node이기 때문에, Pod가 종료되어도 ELB Target의 Deregistration은 발생하지 않습니다. 따라서 Pod 종료 시 해당 Pod를 트래픽 대상에서 제외하는 처리는 ELB가 아니라 EndpointSlice와 kube-proxy를 통해 이루어집니다.

Deregistration Delay와 terminationGracePeriodSeconds는 서로 다른 계층에서 동작합니다.
**Deregistration Delay는 ELB가 기존 요청과 Connection을 Drain하기 위해 기다리는 시간**이고,
**terminationGracePeriodSeconds는 Kubernetes가 Pod 내부의 애플리케이션이 정상적으로 종료될 때까지 기다리는 시간**입니다.
