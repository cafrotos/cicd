# 1. Yêu cầu
* Labtop đã cài sẵn docker, docker-compose
* Cài đặt [rancher k3d](https://github.com/rancher/k3d)
* Cài đặt [kubectl](https://kubernetes.io/vi/docs/tasks/tools/install-kubectl/)
* Cài đặt [helm v3](https://helm.sh/docs/intro/install/)

# 2. Mô tả project
Project demo về cách triển khai CICD cho 1 ứng dụng(Golang) lên Kubernetes(k8s), sử dụng gitlab ci và helm.

Vì là bản demo trên môi trường máy cá nhân nên để tiết kiệm tài nguyên máy tính, project sẽ sử dụng k3s/k3d là phiên bản nhẹ hơn của k8s trên môi trường docker để thay thế cho phiên bản đầy đủ của k8s. Mọi thông tin về k3s trong [đường dẫn](https://k3s.io/) này.

# 3. Các bước thực hiện
## Bước 1: Triển khai gitlab-ce, gitlab-runner sử dụng docker-compose

Vì việc khởi chạy gitlab ce, gitlab runner trên k3s/k3d sẽ gặp nhiều bất cập, do phải chạy docker-in-docker, đồng thời việc chạy gitlab-runner cũng sẽ phải chạy docker-in-docker nên gitlab-ce, gitlab-runner sẽ được chạy tách biệt với k3s/k3d. Để cho tiện dụng, chúng ta sẽ sử dụng docker-compose
```bash
cd local-gitlab
docker-compose up -d
```
## Bước 2: Tạo repository và tiến hành tích hợp gitlab-runner vào repository

* Mở trình duyệt web truy cập vào trang web [http://localhost](http://localhost)
* Cập nhật mật khẩu cho tài khoản root
* Từ màn hình chính, chọn `New Project` 
* Điền các thông tin cần thiết và chọn `Create Project`
* Trên màn hình dashboard repository vừa tạo, chọn mục `Setting => CI/CD`
* Trên màn hình `CI/CD` chọn `Expland` mục `Runners`
* Kéo xuống phần `Set up a specific Runner manually` và copy token để đăng ký runner
* Chạy câu lệnh ben dưới để đăng ký gitlab runner
```bash
docker-compose exec gitlab-runner gitlab-runner register -n --url http://{ip_host} --registration-token {Thay thế bằng token vừa copy} --clone-url http://{ip_host} --executor docker --docker-image "docker:latest" --docker-privileged
```

## Bước 3: Triển khai k3s/k3d
* Chạy lệnh bash sau để khởi tạo 1 cụm Kubernetes với 1 worker
```bash
k3d create --workers 1
```
* Sau đó update lại KUBECONFIG theo hướng dẫn của k3d trên màn hình
```bash
export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
```
* Tiến hành cập nhật lại địa chỉ server của k3s/k3d, việc này rất cần thiết để các runner có thể gọi tới api của k3s/k3d phục vụ việc update các pod/deployment/service/...
```bash
nano $KUBECONFIG
<Thay đổi dòng server, chuyển localhost thành địa chỉ ip của máy>
<VD: server: https://192.168.1.102:6443>
```

## Bước 4: Tạo helm config
* Chạy lệnh sau để tạo các tệp cấu hình
```bash
helm create <tên cấu hình>
```
* Chỉnh sửa các các tệp cấu hình các ứng dụng sẽ chạy trong k3s/k3d theo ý muốn

## Bước 5: Tạo file cấu hình gitlab-ci
Tham khảo tài liệu tại [đây](https://docs.gitlab.com/ee/ci/yaml/)

## Bước 6: Cấu hình biến môi trường cho gitlab-runner
Ta cần cập nhật các biến sau: KUBECONFIG, DOCKER_USER, DOCKER_USER_TOKEN(Token của user trên docker registry), APP_NAME, 

* `KUBECONFIG` Sẽ được lấy bằng cách base64 file config của k3s/k3d sau đó sẽ được ghi lại vào môi trường của runner, xem trong mục deploy của file `.gitlab-ci.yml` của project để biết thêm chi tiết. Sử dụng lệnh sau để base64 file config
```bash
cat $KUBECONFIG | base64
```
* `DOCKER_USER` là username của docker registry
* `DOCKER_USER_TOKEN` là token trong mục setting của docker registry
* `APP_NAME` là tên của app sẽ deploy lên k3s/k3d

## Bước 7: Viết file .gitlab-ci.yml
Tham khảo file có sẵn trong project
## Bước 8: Push code lên local gitlab tận hưởng thành quả
