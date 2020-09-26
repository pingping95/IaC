# Ansible 기초 개념 & 설치 & 초기 설정

- Ansible 기본 개념
1. Inventory : 대상 머신을 정의하는 파일
2. Module : 앤서블에서 어떤 작업을 수행할 것인지 정의하는 것으로, 앤서블에서 실행되는 명령어이다.
3. Ad-hoc : 앤서블 모듈을 단독으로 실행하는 명령어
4. Playbook : Ansible Module들을 수행하는 Script 파일
5. Role : Playbook를 부품화하여 사용하는 것

- Ansible 특징
1. 선언형
2. 멱등성
3. Agent-less
4. 재사용성 (Reusable)

- Module에서 Shell과 Command, template의 차이
    - command : 원격 노드에서 명령을 실행. **쉘을 통해 처리하지 않으므로 변수 등이 적용 X**
    - shell : 노드에서 명령을 실행. 원격 노드의 쉘(/bin/sh)을 통해 명령 실행
    - template : Jinja 2 파이썬 템플릿 엔진을 사용하며 동적인 값을 변수화 처리를 통해 각 사용 환경에 맞게 설정해줄 수 있는 모듈이다.

# Ansible 설치

## 순서

- epel-release 설치
- ansible 설치
- version 확인

```bash
yum -y install epel-release

yum -y install ansible

ansible --version
```

# Ansible 기초 설정

## 1. 인증

1. SSH-Key를 통한 인증

    Ansible은 ssh 프로토콜을 사용하여 통신을 하기 때문에 ssh 인증이 필요하다.

    SSH 인증은 크게 1) Password 인증과 2) Key 인증이 있는데 접속할 때 인증 정보를 요구하지 않게 하기 위해 Managed Host 부분에 public key를 전달하는 과정이 필요하다.

![Untitled](https://user-images.githubusercontent.com/67780144/94347283-b9964380-006d-11eb-8072-3b58621d810f.png)



- key 생성

    root → root 로의 public key 인증

```bash
ssh-keygen

ssh-copy-id root@<<Managed-Host IP>>
```

- test

    비밀번호를 묻지 않고 ssh 접속이 가능하면 성공.

```bash
ssh root@<<Managed-Host IP>>
```

- Root → Root가 아닌 다른 User와 ansible 통신

    ⇒ ssh-copy-id 를 user에게 전달한다. (만약, user_name이 devops라면 ssh-copy-id devops@~~)

- /etc/hosts

    임시적으로 [Localhost](http://Localhost) 자체 Name Resolving을 통해 접근하려는 호스트의 Domain Name으로 접속하기 위해 설정해줌

```bash
[[ 예시 ]]
..
..
192.168.56.10 master master.local
192.168.56.21	host1	 host1.local
192.168.56.22	host2	 host2.local
192.168.56.23	host3	 host3.local
```

<< 만약 자체 Name Server에서 Resolving이 가능하다면 위 작업은 하지 않아도 된다. >>

2. Managed_node의 원격 사용자가 sudo 권한을 주고 싶을 때

- scp 명령어를 통해 파일 전송 (root로 접근을 해야 함)
- ansible 명령어를 통해 파일 전송 or 파일 생성 모듈로 특정 텍스트 포함한 파일 생성

```bash
sudoers.yml
---
- name: Ansible user can sudo without password
  hosts: all
  become: true               # /etc/sudoers.d/ 때문에 Root 권한 필요
  tasks:
    - name: Copy sudoers file to /etc/sudoers.d/
      copy:                  # copy 모듈 사용
        content: "user  ALL=(ALL)  NOPASSWD:ALL"
        dest: /etc/sudoers.d/user          # 만들 위치 설정

ansible -m command -a "head -1 /etc/shadow" -u user --become all
```

- 직접 원격 접속한 후 파일 생성

```bash
echo "user ALL=(ALL)  NOPASSWD:ALL" > /etc/sudoers.d/user
```

3. SSH Authorized key를 특정 유저 계정에 추가

```bash
- name: Set authorized key took from file
  authorized_key:
    user: charlie          # autho_keys 파일이 수정 될 원격 호스트의 유저 이름
    state: present
    key: "{{ lookup('file', '/home/charlie/.ssh/id_rsa.pub') }}"
```

→ root 유저일 경우 /root/.ssh/id_rsa.pub

→ 인벤터리 파일에 ip만 추가해주면 몇 번이든 다시 돌려도 키 값이 없는 Node에만 rsa 키를 등록한다.

## 2. ansible.cfg

- 기본 설정 파일
    - 설정파일을 참조하는 순서
        1. 현재 디렉터리의 ansible.cfg
        2. $HOME/.ansible.cfg
        3. /etc/ansible/ansible.cfg
    - ansible 설정 파일은 여러 섹션으로 구성
        - 자주 사용하는 섹션은 [default], [privileged_escalation] 섹션이다.

- 예시

```bash
[defaults]
inventory = ./inventory      # 인벤토리 파일 지정
remote_user = admin          # 원격 사용자 : admin
ask_pass = false             # password 질의 X

[privilege_escalation]
become = true                 # 관리 대상 호스트에서 작업 권한 상승 활성화 여부
become_method = sudo          # 권한 상승 방법으로 sudo, su 가 있다.
become_user = root            # 권한 상승 계정
become_ask_pass = false       # 권한 상승 시 암호 질의 여부

host_key_checking = false     # 접속 시 key 확인 작업 유무 (yes 안치고 싶을 때)
```

## ~/.vimrc 설정

- vi 에디터에서 yaml 파일을 편리하게 편집하기 위한 환경 만들기

```bash
# vi ~/.vimrc
syntax on
autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab autoindent nu
```

# Tips

- 개행 처리

```bash
msg: |                # 각 라인을 개행(줄바꿈)처리한다.
  first line
  second line
  third line

hosts:                # YAML List 형식
  - server1
  - server2
  - server3
```

- 변수 정의
    - 플레이북에서 변수 정의

        ```bash
        - hosts: all
          vars:
            service: httpd
            service_port: 80
        ```

    - 외부파일에서 변수 정의

        ```bash
        - hosts: all
          vars_files:
          - vars/service.yaml

        # 외부 파일은 ( vars/~~ ) 딕셔너리 형태여야 한다.
        service: httpd
        service_port: 80
        ```

    - playbook에서 변수 사용

        ```bash
        - {{service}}
        - {{service_port}}
        ```

    - 예시

        ```bash
        # 변수 정의
        vars:
          service: httpd

        # 변수 사용
        tasks:
          - name: install {{service}} packages
            yum:
              name: "{{service}}"
              state: present

        # key에 대한 value가 변수로 시작하는 경우 value와 변수를 구분하기 위해
        # 변수가 포함된 이중괄호를 끈 따옴표에 붙여야 함

        # 햇갈리다면 애초에 변수가 포함된 문장은 "" 로 시작해도 무방하다.
        ```

    - Inventory 내에서의 변수 정의

        ```bash
        [web]
        server1.test.com service=httpd

        [db]
        server2.test.com

        [db:vars]
        service=MariaDB
        ```

        → inventory 파일에 변수 등록 시 복잡 + 관리 힘듬

        ⇒ 이러한 이유로 보편적으로 group_vars, host_vars를 사용함

    - host_vars, group_vars
        - 그룹변수의 디렉터리명 : group_vars, 파일명 : inventory 내의 Section 이름과 일치해야 함

        ~/test_ansible/group_vars/db

        service: dhcpd

        - 호스트변수의 디렉터리명 : host_vars, 파일명 : inventory 내의 host 이름과 일치해야 함

        ~/test_ansible/host_vars/server2.test.com

        service: named

    - CLI에서의 변수 정의

        ```bash
        ansible-playbook server1.test.com test.yaml -e "service=httpd"

        # ad-hoc 명령어 라인에 변수가 오면 최상위 우선순위가 된다.
        ```

    - 변수 사용

        users.user1.uid =⇒ 1000

        ~= users['user1']['uid']

        users.user2.homedir =⇒ /home/user2

        ~= users['user2']['homedir']

    - register 변수
        - task의 실행 결과를 변수에 저장 (주로 Debugging 목적)
    - debug 모듈
        - 디버깅을 위한 목적으로 사용, 메시지나 변수값을 출력해줌
    - Magic 변수
        - ansible에 의해 자동으로 설정되는 변수
        1. hostvars : 관리대상 호스트의 변수를 얻을 때 사용
        2. group_names : 현재 호스트가 속한 그룹 리스트
        3. groups : inventory 내의 전체 그룹 및 모든 호스트
        4. inventory_hostname : inventory에 실제 기록되어 있는 호스트 이름
    - Fact 변수
        - ansible이 managed node에서 자동으로 검색한 변수

            호스트 이름, 커널 버젼, 환경 변수, CPU 정보, 메모리 정보, 디스크 정보, 네트워크 정보, 운영체제 정보, IP 주소, ...

        - ad-hoc 명령어로 팩트 수집

            ansible server1 -m setup

    - ansible playbook
        - playboox syntax 체크

            ansible-playboox —syntax-check test.yaml

    - ansible dry-run (시뮬레이션)
        - 플레이가 실행되지만 managed node에는 아무런 변화 X

            ansible-playboox -C test.yam

- 제어 구문
    - when
    - loop
- Handler
- 오류 처리

    플레이북 실행 시 위에서 아래로 순차적으로 실행되는데 중간에 에러가 발생할 시 더이상 실행되지 않으므로 error를 무시하고 플레이북을 계속 실행시킬 수 있다.

- role : 컨텐츠를 그룹화하여 다른 사용자와 쉽게 공유할 수 있으며 협업이 가능
    - defaults : 역할변수의 기본값이 설정되어 있는 디렉터리
    - files : 역할작업에서 참조하는 정적 파일이 있는 디렉터리
    - handlers : 역할의 핸들러가 정의되어 있는 디렉터리
    - meta : 작성자, 라이센스, 플랫폼 등 역할 종속성을 포함한 역할에 대한 정보가 들어있는 디렉터리
    - templates : Jinja2 템플릿이 있는 디렉터리
    - tests : 역할을 테스트할 때 사용하는 인벤터리가 있는 디렉터리
    - vars : 역할 변수의 값을 정의하는 디렉터리

![Untitled 1](https://user-images.githubusercontent.com/67780144/94347285-bac77080-006d-11eb-80ff-e14e5d8bed2d.png)

- 모듈 간단 사용법
    1. uptime

        ansible all -m shell "uptime" -k

    2. disk 확인

        ansible all -m shell "df -h" -k

    3. 메모리 확인

        ansible all -m shell "free -h" -k

    4. 유저 생성

        ansible all -m user -a "name=test password=1234" -k

    5. 파일 전송

        ansible nginx -m copy -a "src=./test dest=/tmp/" -k

- Reference (좋은 글 감사합니다)

    [https://doctorlinux.tistory.com/44](https://doctorlinux.tistory.com/44)
