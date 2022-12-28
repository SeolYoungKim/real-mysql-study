# 사용자 및 권한 
- MySQL의 사용자 계정은 아이디 뿐만 아니라 IP까지 확인한다. 
- MySQL 8.0부터는 권한을 묶어서 관리하는 역할 개념이 도입됐다 


## 사용자 식별 
- 사용자의 접속 지점(호스트명 or 도메인 or IP 주소)도 계정의 일부다 


- MySQL에서 계정을 언급할 때 다음과 같이 아이디와 호스트를 함께 명시해야 함
  - `'svc_id'@'127.0.0.1''` : id와 ip 주소는 역따옴표(`)혹은 홑따옴표(')로 감싼다 
  - `'svc_id'@'%'` : 다음과 같이 작성하면 모든 IP or 모든 호스트명에서 접속 가능


- 만약, 동일한 아이디가 있다면 어떤 계정을 선택할까?
  - `'svc_id'@'192.168.0.10'` : 비밀번호 123 
  - `'svc_id'@'%'` : 비밀번호 abc


- MySQL은 범위가 가장 작은 것을 먼저 선택한다.
  - 두 계정 정보 가운데, 범위가 좁은 것은 `192.168.0.10`이기 때문에, IP가 명시된 계정 정보를 이용해 이 사용자를 인증한다.
    - 따라서, IP가 192.168.0.10인 PC에서 `svc_id`라는 아이디와 `abc`라는 비밀번호로 로그인하면 비밀번호가 일치하지 않는다는 이유로 접속이 거절된다.
    - 중첩된 계정을 생성하지 않도록 주의하자 


## 사용자 계정 관리
### 시스템 계정과 일반 계정 
- SYSTEM_USER 권한 여부
  - 시스템 계정 : DB 서버관리자를 위한 계정 
    - 시스템 계정과 일반 계정을 관리 
    - 다른 세션(connection) 또는 그 세션에서 실행 중인 쿼리를 강제 종료 
    - 스토어드 프로그램 생성 시 DEFINER를 타 사용자로 설정 
  - 일반 계정  : 응용 프로그램이나 개발자를 위한 계정 


- MySQL 서버 내장 계정
![img.png](img.png)
  - `'root'@'localhost''` 
  - `'mysql.sys'@'localhost''` : sys 스키마 객체 (뷰, 함수, 프로시저들)들의 DEFINER로 사용되는 계정 
  - `'mysql.session'@'localhost''` : MySQL 플러그인이 서버로 접근할 때 사용되는 계정 
  - `'mysql.infoschema'@'localhost''` : information_schema에 정의된 뷰의 DEFINER로 사용되는 계정


### 계정 생성 
- 계정 생성 : `create user`
  - 계정의 인증 방식과 비밀번호 
  - 비밀번호 관련 옵션(비밀번호 유효 기간, 비밀번호 이력 개수, 비밀번호 재사용 불가 기간)
  - 기본 역할(role)
  - ssl 옵션
  - 계정 잠금 여부 
- 권한 부여 : `grant`


### 계정 생성 옵션 
```mysql
create user 'user'@'%'
identified with 'mysql_native_password' by 'password'
require none 
password expire interval 30 day
account unlock 
password history default 
password reuse interval default 
password require current default;
```
#### identified with
- 인증 방식과 비밀번호 설정 <br/><br/>
- `identified with '인증 방식(플러그인)' by 'password'` <br/><br/>
- 인증 플러그인 종류
  - native pluggable authentication : 비밀번호에 대한 해시(sha-1) 값을 저장해두고, 클라이언트가 보낸 값과 해시 값 비교 <br/><br/>
  - caching SHA-2 pluggable authentication : 암호화 해시값 생성을 위해 sha-2(256비트) 알고리즘 사용. 해시값의 보안에 더 중점을 둔 방식.
    - native authentication -> 입력이 동일 해시값 출력
    - caching sha-2 -> 내부적으로 salt 키 활용. 수천 번의 해시 계산을 수행해서 결과를 만들어냄. 동일한 키에 대해서도 결과가 달라짐.
    - 해시 결괏값을 메모리에 캐시에서 사용하기 때문에 `caching`이라는 이름이 붙음 
    - 해당 인증 방식은 SSL/TLS 또는 rsa 키페어를 반드시 사용해야 한다. -> SSL 옵션 활성화 필요 <br/><br/>
  - PAM pluggable authentication : 유닉스나 리눅스 패스워드 또는 LDAP(Lightweight Directory Access Protocol)같은 외부 인증을 사용할 수 있게 해주는 방식 <br/><br/> 
  - LDAP pluggable authentication : LDAP를 이용한 외부 인증을 사용할 수 있게 해주는 인증 방식 <br/><br/>
- MySQL 8.0의 기본 -> Caching SHA-2

<br>

#### require
- 서버 접속 시 암호화된 SSL/TLS 채널을 사용할지 여부 설정 
- 별도로 설정하지 않을 경우 비암호화 채널로 연결함 
- 옵션을 SSL로 설정하지 않아도, Caching SHA-2 Authentication 인증 방식을 사용하면 암호화된 채널만으로 서버에 접속할 수 있음 

<br>

#### password expire
- 비밀번호의 유효 기간 설정 
- 별도로 명싲하지 않으면 `default_password_liftime`시스템 변수에 저장된 기간으로 유효 기간이 설정 됨 
- 추가 옵션 
  - expire : 계정 생성과 동시에 비밀번호 만료 처리 
  - expire never : 만료 없음 
  - expire default : `default_password_liftime`시스템 변수에 저장된 기간으로 유효 기간이 설정
  - expire interval n day : 비밀번호 유효 기간을 오늘부터 n일자로 설정 

<br>

#### password history
- 한번 사용했던 비밀번호를 재사용하지 못하게 설정 
- 추가 옵션 
  - history default : `password_history` 시스템 변수에 저장된 개수만큼 비밀번호 이력 저장. 이력에 남아있는 비밀번호 재사용 불가
  - history n : 최근 n개까지의 이력만 저장 

<br>

#### password reuse interval
- 한 번 사용했던 비밀번호의 재사용 금지 기간 설정
- 별도로 명시하지 않을 경우 `password_reuse_interval` 시스템 변수에 저장된 기간으로 설정 됨 
- 추가 옵션 
  - default : `password_reuse_interval` 시스템 변수에 저장된 기간으로 설정
  - n day : n일자 이후에 비밀번호 재사용 가능하도록 설정 

<br>

#### password require
- 비밀번호가 만료되어 새로운 비밀번호로 변경할 때 현재 비밀번호를 필요로 할지 말지 결정하는 옵션 
- 명시되지 않을 경우 `password_require_current`시스템 변수의 값으로 설정 됨 
- 추가 옵션
  - current : 비밀번호 변경 시 현재 비밀번호를 먼저 입력하도록 설정 
  - optional : 비밀번호 변경 시 현재 비밀번호 입력하지 않아도 되도록 설정
  - default : `password_require_current`시스템 변수의 값으로 설정

<br>

#### account lock/unlock
- 계정 생성 시, 또는 alter user 명령을 사용해 계정 정보를 변경할 때 계정을 사용하지 못하게 잠글지 여부 결정 
- lock : 계정 사용 못하게 잠금
- unlock : 잠긴 계정을 다시 사용 가능 상태로 잠금 해제

<br>

### 고수준 비밀번호
- 비밀번호 글자 조합을 강제하거나 금칙어를 설정하는 기능이 있음 
- 유효성 체크 규칙을 적용하려면 `validate_password` 컴포넌트 이용 
```mysql
## validate_password 컴포넌트 설치
install component 'file://component_validate_password';

## 컴포넌트 확인 
select * from mysql.component;
```

- 설치되면 컴포넌트에서 제공하는 시스템 변수들을 확인할 수 있음<br/><br/> 
- 비밀번호 정책 
  - low : 길이만 검증 
  - medium : 길이 검증 & 숫자와 대소문자, 특수문자 배합 검증
  - strong : medium 레벨의 검증 모두 수행 & 금칙어 포함 여부 검증<br/><br/>
 

<br>

### 이중 비밀번호 
- 비밀번호는 서비스가 실행 중인 상태에서 변경이 불가능하다.
- 이중 비밀번호(Dual Password) : 2개의 비밀번호 중 하나만 일치하면 로그인이 통과되는 것을 의미 
  - primary : 최근에 설정된 비밀번호 
  - secondary : 이전 비밀번호 
```mysql
alter user 'root'@'localhost' identified by 'old_password';

# 비밀번호를 변경하면서 기존 비밀번호를 세컨더리 비밀번호로 설정  
alter user 'root'@'localhost' identified by 'new_password' retain current password;
```

- 위와 같이 설정  후, 배포 및 재시작을 실행 
  - 재시작이 완료되면 세컨더리 비밀번호는 삭제 

```mysql
alter user 'root'@'localhost' discard old password;
```

<br>

## 권한 
### 정적 권한 
- 글로벌 권한
  - 데이터베이스나 테이블 이외의 객체에 적용되는 권한 
  - grant 명령에서 특정 객체를 명시하면 안됨 <br/><br/>
- 객체 권한 
  - 데이터베이스나 테이블을 제어하는 데 필요한 권한 
  - grant 명령으로 권한을 부여할 때 반드시 특정 객체 명시 <br/><br/>
- all(or all privileges)
  - 글로벌과 객체 권한 두 가지 용도로 사용 가능 
  - 특정 객체에 all 권한이 부여되면 객체에 적용될 수 있는 모든 객체 권한을 부여함 
  - 글로벌로 all이 사용되면 글로벌 수준에서 가능한 모든 권한을 부여 <br/><br/>

<br>

### 동적 권한 
- MySQL 서버의 컴포넌트나 플러그인이 설치되면 그 때 등록되는 권한 

<br>

### 권한 부여 
```mysql
# 글로벌 권한
grant super on *.* to 'user'@'localhost';

# db 권한 : 모든 db or 특정 db (테이블, 스토어드 프로그램 포함) / 테이블 명시 불가 
grant event on *.* to 'user'@'localhost';
grant event on employees.* to 'user'@'localhost';

# 테이블 권한 
grant select, insert, update, delete on *.* to 'user'@'localhost';
grant select, insert, update, delete on employees.* to 'user'@'localhost';
grant select, insert, update, delete on employees.department to 'user'@'localhost';

# 특정 컬럼에 대해서만 권한 부여 : delete는 안됨  
grant select, insert, update(dept_name) on employees.department to 'user'@'localhost';
```

<br>

## 역할 
- 권한을 묶어서 역할로 사용 가능 
- 역할은 계정과 똑같은 모습을 하고 있음 

```mysql
# 빈 껍데기만 있는 역할 정의 
create role role_emp_read, role_emp_write;

# grant 명령으로 역할에 대해 권한 부여 
grant select on employees.* to role_emp_read;
grant insert, update, delete on employees.* to role_emp_write;

# 계정에 역할 부여 
grant role_emp_read to reader@'127.0.0.1';
grant role_emp_read, role_emp_write to writer@'127.0.0.1';

# 역할을 활성화 해야 사용할 수 있다. 하지만 이는 로그아웃 후 재로그인하면 해제 됨  
set role 'role_emp_read';

# 역할 자동 활성화 여부 : 매번 활성화 하지 않아도 로그인과 동시에 부여된 역할이 자동 활성화 
set global activate_all_roles_on_login=on;
```

- MySQL 서버 내부적으로 역할과 계정은 동일한 객체로 취급 됨 
  - 하나의 사용자 계정에 다른 사용자 계정이 가진 권한을 병합해서 권한 제어가 가능해졌을 뿐 
  - 역할은 account_locked 필드가 Y임 
  - 