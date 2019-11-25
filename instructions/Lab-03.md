---
lab:
    title: 'Azure로 온-프레미스 PostgreSQL 데이터베이스 마이그레이션'
    module: '모듈 3: Azure SQL DB for PostgreSQL로 온-프레미스 PostgreSQL 마이그레이션'
---

# 랩: Azure로 온-프레미스 PostgreSQL 데이터베이스 마이그레이션

## 개요

이 랩에서는 이 모듈에서 파악한 정보를 토대로 PostgreSQL 데이터베이스를 Azure로 마이그레이션합니다. 모듈 전체의 내용을 활용하기 위해 마이그레이션 2회를 수행합니다. 우선 Azure에서 실행 중인 가상 머신으로 온-프레미스 PostgreSQL 데이터베이스를 전송하는 오프라인 마이그레이션을 수행합니다. 그 후에는 가상 머신에서 실행 중인 데이터베이스를 Azure Database for PostgreSQL로 전송하는 온라인 마이그레이션을 수행합니다.

또한 데이터베이스를 사용하는 샘플 애플리케이션을 다시 구성하고 실행하여 각 마이그레이션 후 데이터베이스가 올바르게 작동하는지 확인합니다.

## 목표

이 랩을 완료하면 다음 작업을 수행할 수 있습니다.

1. 온-프레미스 PostgreSQL 데이터베이스를 Azure 가상 머신으로 전송하는 오프라인 마이그레이션 수행

1. 가상 머신에서 실행 중인 PostgreSQL 데이터베이스를 Azure Database for PostgreSQL로 전송하는 온라인 마이그레이션 수행

## 시나리오

여러분은 AdventureWorks라는 조직에서 데이터베이스 개발자로 근무하고 있습니다. AdventureWorks는 10년 이상 최종 소비자 및 유통업체에 자전거와 자전거 부품을 직접 판매해 왔습니다. AdventureWorks의 시스템은 현재 온-프레미스 데이터 센터에 있는 PostgreSQL을 사용하여 실행되는 데이터베이스에 정보를 저장합니다. AdventureWorks는 합리적인 하드웨어 운영을 위해 Azure로 데이터베이스를 이동할 계획입니다. 여러분은 이 마이그레이션 수행 업무를 맡게 되었습니다.

처음에는 데이터 위치만 Azure 가상 머신에서 실행되는 PostgreSQL 데이터베이스로 변경하기로 했습니다. 이 방식을 사용하면 데이터베이스를 전혀 또는 거의 변경하지 않아도 되므로 위험성이 낮기 때문입니다. 하지만 이 방식에서는 데이터베이스와 연관된 일상적인 모니터링 및 관리 태스크를 대부분 계속 수행해야 합니다. 또한 AdventureWorks의 고객층 변화 과정도 고려해야 합니다. 창업 초기에 AdventureWorks는 소규모 지역 사업체였지만 현재는 규모가 확대되어 전 세계에서 사업을 운영하고 있습니다. 즉, 전 세계 어디서나 고객이 제품을 구매할 수 있으므로 데이터베이스를 쿼리하는 고객의 대기 시간을 최소화해야 합니다. 다른 지역에 있는 가상 머신에 PostgreSQL 복제를 구현할 수는 있지만 이 경우에도 관리 오버헤드가 발생합니다.

그러므로 이 방식을 사용하는 대신 가상 머신에서 시스템 실행을 시작한 후 장기적인 솔루션을 고려할 예정입니다. 즉, Azure Database for PostgreSQL로 데이터베이스를 마이그레이션하고자 합니다. 이 PaaS 전략을 도입하는 경우 시스템 유지 관리와 연관된 대다수 작업을 수행할 필요가 없게 됩니다. 또한 시스템을 쉽게 확장할 수 있으며 읽기 전용 복제본을 추가하여 전 세계 어디서나 고객을 지원할 수 있습니다. 그리고 Microsoft에서 제공하는 가용성 보장 SLA도 적용됩니다.

## 설정

온-프레미스 환경의 기존 PostgreSQL 데이터베이스에 Azure로 마이그레이션하려는 데이터가 포함되어 있습니다. 랩을 시작하기 전에 초기 마이그레이션 대상으로 사용할 PostgreSQL을 실행하는 Azure 가상 머신을 만들어야 합니다. 이 가상 머신을 만들고 구성하는 스크립트는 별도로 제공됩니다. 다음 단계에 따라 이 스크립트를 다운로드하여 실행하세요.

1. 수업용으로 실행 중인 **LON-DEV-01** 가상 머신에 로그인합니다. 사용자 이름은 **azureuser** 이고 암호는 **Pa55w.rd** 입니다.

2. 브라우저를 사용하여 Azure Portal에 로그인합니다.

3. Azure Cloud Shell 창을 엽니다. **Bash** 셸을 실행 중인지 확인합니다.

4. 스크립트와 샘플 데이터베이스가 포함되어 있는 리포지토리를 아직 복제하지 않았으면 복제합니다.

```bash
git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
```

5. *workshop/migration_samples/setup* 폴더로 이동합니다.

```bash
cd ~/workshop/migration_samples/setup
```

6. 다음과 같이 *create_postgresql_vm.sh* 스크립트를 실행합니다. 리소스 그룹의 이름과 가상 머신을 저장할 위치를 매개 변수로 지정합니다. 리소스 그룹은 아직 없는 경우 자동으로 만들어집니다. *eastus*, *uksouth* 등과 같이 근처의 위치를 지정합니다.

```bash
bash create_postgresql_vm.sh [resource group name] [location]
```

스크립트를 실행하려면 10분 정도 걸립니다. 스크립트 실행 시에는 많은 출력이 생성되며, 출력 완료 시에는 새 가상 머신의 IP 주소와 **Setup Complete** 메시지가 표시됩니다. IP 주소를 적어 두세요.
7. IP 주소를 적어 두세요.

> [!참고]
> 연습을 진행하려면 이 IP 주소가 필요합니다.

## 연습 1: Azure 가상 머신으로 온-프레미스 데이터베이스 마이그레이션

이 연습에서는 다음 태스크를 수행합니다.

1. 온-프레미스 데이터베이스 검토
2. 데이터베이스를 쿼리하는 샘플 애플리케이션 실행
3. Azure 가상 머신으로 데이터베이스를 전송하는 오프라인 마이그레이션 수행
4. Azure 가상 머신에서 데이터베이스 확인
5. Azure 가상 머신의 데이터베이스용으로 샘플 애플리케이션 다시 구성 및 테스트

### 태스크 1: 온-프레미스 데이터베이스 검토

1. 수업용으로 실행 중인 **LON-DEV-01** 가상 머신에 로그인합니다. 사용자 이름은 **azureuser** 이고 암호는 **Pa55w.rd** 입니다.

    이 가상 머신은 온-프레미스 환경을 시뮬레이션하며, PostgreSQL 서버를 실행하고 있습니다. 이 서버가 마이그레이션해야 하는 AdventureWorks 데이터베이스를 호스트합니다.

2. 수업용으로 실행 중인 **LON-DEV-01** 가상 머신의 화면 왼쪽에 있는 **즐겨찾기** 모음에서 **pgAdmin4** 유틸리티를 클릭합니다.

3. **저장된 암호 잠금 해제** 대화 상자에서 암호 **Pa55w.rd** 를 입력한 다음 **확인** 을 클릭합니다.

4. **pgAdmin4** 창에서 **서버**, **LON-DEV-01**, **데이터베이스**, **adventureworks**, **스키마** 를 차례로 확장합니다.

5. **sales** 스키마에서 **테이블** 을 확장합니다.

6. **salesorderheader** 테이블을 마우스 오른쪽 단추로 클릭하고 **스크립트**, **SELECT 스크립트** 를 차례로 클릭합니다.

7. **F5** 키를 눌러 쿼리를 실행합니다. 행 31,465개가 반환됩니다.

8. 테이블 목록에서 **salesorderdetail** 테이블을 마우스 오른쪽 단추로 클릭하고 **스크립트**, **SELECT 스크립트** 를 차례로 클릭합니다.

9. **F5** 키를 눌러 쿼리를 실행합니다. 행 121,317개가 반환됩니다.

10. 데이터베이스의 여러 스키마에서 다른 테이블의 데이터를 몇 분 정도 검색해 봅니다.

### 태스크 2: 데이터베이스를 쿼리하는 샘플 애플리케이션 실행

1. **LON-DEV-01** 가상 머신의 즐겨찾기 모음에서 **터미널** 을 클릭하여 터미널 창을 엽니다.

1. 터미널 창에서 데모용 샘플 코드를 다운로드합니다. 메시지가 표시되면 암호로 **Pa55w.rd** 를 입력합니다.

```bash
sudo rm -rf ~/workshop
git clone  https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
```

1. *~/workshop/migration_samples/code/postgresql/AdventureWorksQueries* 폴더로 이동합니다.

```bash
cd ~/workshop/migration_samples/code/postgresql/AdventureWorksQueries
```

이 폴더에는 쿼리를 실행하여 *adventureworks* 데이터베이스 내 여러 테이블의 행 수를 계산하는 샘플 앱이 포함되어 있습니다.

1. 앱을 실행합니다.

```bash
dotnet run
```

앱에서 다음 출력을 생성합니다.

```text
Querying AdventureWorks database
SELECT COUNT(*) FROM production.vproductanddescription
1764

SELECT COUNT(*) FROM purchasing.vendor
104

SELECT COUNT(*) FROM sales.specialoffer
16

SELECT COUNT(*) FROM sales.salesorderheader
31465

SELECT COUNT(*) FROM sales.salesorderdetail
121317

SELECT COUNT(*) FROM person.person
19972
```

### 태스크 3: Azure 가상 머신으로 데이터베이스를 전송하는 오프라인 마이그레이션 수행

앞에서 adventureworks 데이터베이스에 포함된 데이터를 살펴보았으므로 이제 Azure의 가상 머신에서 실행 중인 PostgreSQL 서버로 데이터베이스를 마이그레이션할 수 있습니다. 여기서는 백업 및 복원 명령을 사용하여 이 작업을 오프라인 태스크로 수행합니다.

> [!참고]
> 온라인에서 데이터를 마이그레이션하려는 경우에는 온-프레미스 데이터베이스에서 Azure 가상 머신에서 실행 중인 데이터베이스로의 복제를 구성할 수 있습니다.

1. 터미널 창에서 다음 명령을 실행하여 *adventureworks* 데이터베이스의 백업을 생성합니다.

```bash
pg_dump adventureworks -U azureuser -Fc > adventureworks_backup.bak
```

1. Azure 가상 머신에서 대상 데이터베이스를 만듭니다. [nn.nn.nn.nn]은 이 랩의 설정 단계에서 만든 가상 머신의 IP 주소로 바꿉니다. **azureuser** 의 암호를 입력하라는 메시지가 표시되면 암호 **Pa55w.rd** 를 입력합니다.

```azurecli
createdb -h [nn.nn.nn.nn] -U azureuser --password adventureworks
```

1. pg_restore 명령을 사용하여 새 데이터베이스에 백업을 복원합니다.

```bash
pg_restore -d adventureworks -h [nn.nn.nn.nn] -Fc -U azureuser --password adventureworks_backup.bak
```

이 명령을 실행하려면 몇 분 정도 걸립니다.

### 태스크 4: Azure 가상 머신에서 데이터베이스 확인

1. 다음 명령을 실행하여 Azure 가상 머신의 데이터베이스에 연결합니다. 가상 머신에서 실행되는 PostgreSQL 서버의 *azureuser* 사용자 암호는 **Pa55w.rd** 입니다.

```bash
psql -h [nn.nn.nn.nn] -U azureuser adventureworks
```

1. 다음 쿼리를 실행합니다.

```SQL
SELECT COUNT(*) FROM sales.salesorderheader;
```

이 쿼리가 행 31,465개(온-프레미스 데이터베이스의 행 수)를 반환하는지 확인합니다.

1. *sales.salesorderdetail* 테이블의 행 수를 쿼리합니다.

```SQL
SELECT COUNT(*) FROM sales.salesorderdetail;
```

이 테이블에는 행 121,317개가 포함되어 있어야 합니다.

1. **\q** 명령을 실행하여 *psql* 유틸리티를 닫습니다.

1. **pgAdmin4** 도구로 전환합니다.

1. 왼쪽 창에서 **서버** 를 마우스 오른쪽 단추로 클릭하고 **만들기**, **서버** 를 차례로 클릭합니다.

1. **만들기 - 서버** 대화 상자에서 **일반** 탭의 **이름** 상자에 **가상 머신** 을 입력한 다음 **연결** 탭을 클릭합니다.

1. 다음 세부 정보를 입력하고 **저장** 을 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 호스트 이름/주소 | *[nn.nn.nn.nn]* |
    | 포트 | 5432 |
    | 유지 관리 데이터베이스 | postgres |
    | 사용자 이름 | azureuser |
    | 암호 | Pa55w.rd |
    | 암호 저장 | 선택 |
    | 역할 | *비워 둠* |
    | 서비스 | *비워 둠* |

1. **pgAdmin4** 창 왼쪽 창의 **서버** 아래에서 **가상 머신** 을 확장합니다.

1. **데이터베이스**, **adventureworks** 를 차례로 확장하고 데이터베이스의 스키마와 테이블을 찾아봅니다. 테이블이 온-프레미스 데이터베이스의 테이블과 같아야 합니다.

### 태스크 5: Azure 가상 머신의 데이터베이스용으로 샘플 애플리케이션 다시 구성 및 테스트

1. **터미널** 창으로 돌아옵니다.

1. *nano* 편집기에서 테스트 애플리케이션의 App.config 파일을 엽니다.

```bash
nano App.config
```

1. **ConnectionString** 설정의 값을 변경하여 **localhost** 를 Azure 가상 머신의 IP 주소로 바꿉니다. 파일 내용이 다음과 같아야 합니다.

```XML
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <appSettings>
      <add key="ConnectionString" value="Server=nn.nn.nn.nn;Database=adventureworks;Port=5432;User Id=azureuser;Password=Pa55w.rd;" />
    </appSettings>
</configuration>
```

이제 애플리케이션이 Azure 가상 머신에서 실행되는 데이터베이스에 연결됩니다.

1. 파일을 저장하고 편집기를 닫으려면 Esc 키를 누른 후 Ctrl+X를 누릅니다. 메시지가 표시되면 Enter 키를 눌러 변경 내용을 저장합니다.

1. 애플리케이션을 빌드하고 실행합니다.

```bash
dotnet run
```

애플리케이션이 정상적으로 실행되며 각 테이블의 행 수를 이전과 동일하게 반환하는지 확인합니다.

지금까지 온-프레미스 데이터베이스를 Azure 가상 머신으로 마이그레이션하고 새 데이터베이스를 사용하도록 애플리케이션을 다시 구성했습니다.

## 연습 2: Azure Database for PostgreSQL로의 온라인 마이그레이션 수행

이 연습에서는 다음 태스크를 수행합니다.

1. Azure 가상 머신에서 실행되는 PostgreSQL 서버 구성 및 스키마 내보내기
2. Azure Database for PostgreSQL 서버 및 데이터베이스 만들기
3. 대상 데이터베이스로 스키마 가져오기
4. Database Migration Service를 사용하여 온라인 마이그레이션 수행
5. 데이터 수정 및 새 데이터베이스로의 단독형 마이그레이션 수행
6. Azure Database for PostgreSQL에서 데이터베이스 확인
7. Azure Database for PostgreSQL의 데이터베이스용으로 샘플 애플리케이션 다시 구성 및 테스트

### 태스크 1: Azure 가상 머신에서 실행되는 PostgreSQL 서버 구성 및 스키마 내보내기

1. 웹 브라우저를 사용하여 Azure Portal로 돌아옵니다.

1. Azure Cloud Shell 창을 엽니다. **Bash** 셸을 실행 중인지 확인합니다.

1. 스크립트와 샘플 데이터베이스가 포함되어 있는 리포지토리를 아직 복제하지 않았으면 복제합니다.

```bash
git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
```

1. *~/workshop/migration_samples/setup/postgresql/adventureworks* 폴더로 이동합니다.

```bash
cd ~/workshop/migration_samples/setup/postgresql/adventureworks
```

1. PostgreSQL 서버를 실행하는 Azure 가상 머신에 연결합니다. 다음 명령에서 *nn.nn.nn.nn* 은 가상 머신의 IP 주소로 바꾸세요. 메시지가 표시되면 암호 **Pa55w.rdDemo** 를 입력합니다.

```bash
ssh azureuser@nn.nn.nn.nn
```

1. 가상 머신에서 *root* 계정으로 전환합니다. *azureuser* 사용자의 암호를 입력하라는 메시지가 표시되면 **Pa55w.rdDemo** 를 입력합니다.

```bash
sudo bash
```

1. */etc/postgresql/10/main* 디렉터리로 이동합니다.

```bash
cd /etc/postgresql/10/main
```

1. *nano* 편집기를 사용하여 *postgresql.conf* 파일을 엽니다.

```bash
nano postgresql.conf
```

1. 파일 아래쪽으로 스크롤하여 다음 매개 변수가 구성되었는지 확인합니다.

```text
listen-addresses = '*'
wal_level = logical
max_replication_slots = 5
max_wal_senders = 10
```

1. 파일을 저장하고 편집기를 닫으려면 Esc 키를 누른 후 Ctrl+X를 누릅니다. 메시지가 표시되면 Enter 키를 눌러 변경 내용을 저장합니다.

1. PostgreSQL 서비스를 다시 시작합니다.

```bash
service postgresql restart
```

1. Postgres가 올바르게 시작되었는지 확인합니다.

```bash
service postgresql status
```

서비스가 실행 중인 경우 다음과 같은 메시지가 표시됩니다.

```text
 postgresql.service - PostgreSQL RDBMS
  Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
   Active: active (exited) since Fri 2019-08-23 12:47:02 UTC; 2min 3s ago
   Process: 115562 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
  Main PID: 115562 (code=exited, status=0/SUCCESS)

Aug 23 12:47:02 postgreSQLVM systemd[1]: Starting PostgreSQL RDBMS...
Aug 23 12:47:02 postgreSQLVM systemd[1]: Started PostgreSQL RDBMS.
```

1. *root* 계정에서 로그아웃하고 *azureuser* 계정으로 돌아옵니다.

```bash
exit
```

1. bash 프롬프트에서 다음 명령을 실행하여 **adventureworks** 데이터베이스용 스키마를 **adventureworks_schema.sql** 파일로 내보냅니다.

```bash
pg_dump -o  -d adventureworks -s > adventureworks_schema.sql
```

1. 가상 머신에서 연결을 끊고 Cloud Shell 프롬프트로 돌아옵니다.

```bash
exit
```

1. Cloud Shell에서 가상 머신의 스키마 파일을 복사합니다. 여기서 *nn.nn.nn.nn* 은 가상 머신의 IP 주소로 바꿉니다. 메시지가 표시되면 암호 **Pa55w.rdDemo** 를 입력합니다.

```bash
scp azureuser@nn.nn.nn.nn:~/adventureworks_schema.sql adventureworks_schema.sql
```

## 태스크 2. Azure Database for PostgreSQL 서버 및 데이터베이스 만들기

1. Azure Portal로 전환합니다.

2. **+ 리소스 만들기** 를 클릭합니다.

3. **마켓플레이스 검색** 상자에 **Azure Database for PostgreSQL** 을 입력하고 Enter 키를 누릅니다.

4. **Azure Database for PostgreSQL** 페이지에서 **만들기** 를 클릭합니다.

5. **Azure Database for PostgreSQL 배포 옵션 선택** 페이지의 **Single Server** 상자에서 **만들기** 를 클릭합니다.

6. **Single Server** 페이지에서 다음 세부 정보를 입력하고 **검토 + 만들기** 를 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 리소스 그룹 | 앞에서 이 랩의 *설정* 태스크를 수행하여 Azure 가상 머신을 만들 때 지정했던 것과 같은 리소스 그룹 사용 |
    | 서버 이름 | **adventureworks*nnn***. 여기서 *nnn* 은 고유한 서버 이름을 만들기 위해 선택한 접미사입니다. |
    | 데이터 원본 | 없음 |
    | 관리 사용자 이름 | awadmin |
    | 암호 | Pa55w.rdDemo |
    | 암호 확인 | Pa55w.rdDemo |
    | 위치 | 가장 가까운 위치 선택 |
    | 버전 | 10 |
    | 컴퓨팅 + 스토리지 | **서버 구성** 을 클릭하고 **기본** 가격 책정 계층을 선택한 다음 **확인** 을 클릭합니다. |

7. **검토 + 만들기** 페이지에서 **만들기** 를 클릭합니다. 계속하기 전에 서비스가 만들어질 때까지 기다립니다.

8. 서비스가 만들어지면 Azure Portal에서 서비스 관련 페이지로 이동한 다음 **연결 보안** 을 클릭합니다.

9. **연결 보안 페이지** 에서 **Azure 서비스 방문 허용** 을 **켜기** 로 설정합니다.

10. 방화벽 규칙 목록에서 **VM** 규칙을 추가하고 **시작 IP 주소** 및 **끝 IP 주소** 를 앞에서 만든 PostgreSQL 서버를 실행하는 가상 머신의 IP 주소로 설정합니다.

11. **클라이언트 IP 추가** 를 클릭하여 온-프레미스 서버 역할을 하는 **LON-DEV-01** 가상 머신이 Azure Database for PostgreSQL에 연결할 수 있도록 설정합니다. 나중에 재구성된 클라이언트 애플리케이션을 실행할 때 이 액세스 권한이 필요합니다.

12. **저장** 을 클릭하고 방화벽 규칙이 업데이트될 때까지 기다립니다.

13. Cloud Shell 프롬프트에서 다음 명령을 실행하여 Azure Database for PostgreSQL 서비스에서 새 데이터베이스를 만듭니다. *[nnn]* 은 Azure Database for PostgreSQL 서비스를 만들 때 사용한 접미사로 바꿉니다. *[리소스 그룹]* 은 서비스용으로 지정한 리소스 그룹 이름으로 바꿉니다.

```bash
az postgres db create \
    --name azureadventureworks \
    --server-name adventureworks[nnn] \
    --resource-group [resource group]
```

데이터베이스가 올바르게 만들어지면 다음과 같은 메시지가 표시됩니다.

```text
{
    "charset": "UTF8",
    "collation": "English_United States.1252",
    "id": "/subscriptions/nnnnnnnnnnnnnnnnnnnnnn/resourceGroups/nnnnnnnn/providers/Microsoft.DBforPostgreSQL/servers/adventureworksnnn/databases/azureadventureworks",
    "name": "azureadventureworks",
    "resourceGroup": "nnnnnnnn",
    "type": "Microsoft.DBforPostgreSQL/servers/databases"
}
```

### 태스크 3: 대상 데이터베이스로 스키마 가져오기

1. Cloud Shell에서 다음 명령을 실행하여 azureadventureworks[nnn] 서버에 연결합니다. *[nnn]* 인스턴스 2개는 서비스의 접미사로 바꿉니다. 사용자 이름에는 *@adventureworks[nnn]* 접미사가 있습니다. 암호 프롬프트에서 **Pa55w.rdDemo** 를 입력합니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U awadmin@adventureworks[nnn] -d postgres
```

2. 다음 명령을 실행하여 *azureuser* 사용자를 만들고 이 사용자의 암호를 *Pa55w.rd* 로 설정합니다. 세 번째 문은 *azureuser* 사용자에게 *azureadventureworks* 데이터베이스에서 개체를 만들고 관리하는 데 필요한 권한을 부여합니다. *azure_pg_admin* 역할을 사용하면 *azureuser* 사용자가 데이터베이스에서 확장을 설치하고 사용할 수 있습니다.

```SQL
CREATE ROLE azureuser WITH LOGIN;
ALTER ROLE azureuser PASSWORD 'Pa55w.rd';
GRANT ALL PRIVILEGES ON DATABASE azureadventureworks TO azureuser;
GRANT azure_pg_admin TO azureuser;
```

3. **\q** 명령을 실행하여 *psql* 유틸리티를 닫습니다.

4. Azure Database for PostgreSQL 서비스에서 실행 중인 **azureadventureworks** 데이터베이스로 **adventureworks** 데이터베이스용 스키마를 가져옵니다. 가져오기는 *azureuser* 사용자로 수행해야 하므로 메시지가 표시되면 암호 **Pa55w.rd** 를 입력합니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f adventureworks_schema.sql
```

각 항목이 만들어질 때는 일련의 메시지가 표시됩니다. 스크립트가 오류 없이 완료되어야 합니다.

5. 다음 명령을 실행합니다. *findkeys.sql* 스크립트는 또 다른 SQL 스크립트인 *dropkeys.sql* 을 생성합니다. 이 스크립트는 **azureadventureworks** 데이터베이스의 테이블에서 모든 외래 키를 제거합니다. *dropkeys.sql* 스크립트는 잠시 후에 실행할 예정입니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f findkeys.sql -o dropkeys.sql -t
```

시간이 되면 텍스트 편집기를 사용해 *dropkeys.sql* 스크립트를 살펴볼 수 있습니다.

6. 다음 명령을 실행합니다. *createkeys.sql* 스크립트는 또 다른 SQL 스크립트인 *addkeys.sql* 을 생성합니다. 이 스크립트는 모든 외래 키를 다시 만듭니다. 데이터베이스를 마이그레이션한 후에 *addkeys.sql* 스크립트를 실행할 것입니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f createkeys.sql -o addkeys.sql -t
```

7. *dropkeys.sql* 스크립트를 실행합니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f dropkeys.sql
```

외래 키가 삭제되면 일련의 **ALTER TABLE** 메시지가 표시됩니다.

8. psql 유틸리티를 다시 시작하여 *azureadventureworks* 데이터베이스에 연결합니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
```

9. 다음 쿼리를 실행하여 나머지 외래 키의 세부 정보를 찾습니다.

```SQL
SELECT constraint_type, table_schema, table_name, constraint_name
FROM information_schema.table_constraints
WHERE constraint_type = 'FOREIGN KEY';
```

이 쿼리는 빈 결과 집합을 반환해야 합니다. 하지만 외래 키가 아직 있으면 각 외래 키에 대해 다음 명령을 실행합니다.

```SQL
ALTER TABLE [table_schema].[table_name] DROP CONSTRAINT [constraint_name];
```

10. 남아 있는 외래 키를 제거한 후 다음 SQL 문을 실행하여 데이터베이스의 트리거를 표시합니다.

```bash
SELECT trigger_name
FROM information_schema.triggers;
```

이 쿼리도 데이터베이스에 트리거가 없음을 나타내는 빈 결과 집합을 반환해야 합니다. 데이터베이스에 트리거가 있으면 데이터를 마이그레이션하기 전에 트리거를 사용하지 않도록 설정했다가 나중에 다시 사용하도록 설정해야 합니다.

11. **\q** 명령을 실행하여 *psql* 유틸리티를 닫습니다.

## 태스크 4.  Database Migration Service를 사용하여 온라인 마이그레이션 수행

1. Azure Portal로 다시 전환합니다.

2. **모든 서비스**, **구독** 을 차례로 클릭하고 자신의 구독을 클릭합니다.

3. 구독 페이지의 **설정** 에서 **리소스 공급자** 를 클릭합니다.

4. **이름으로 필터링** 상자에 **DataMigration** 을 입력한 다음 **Microsoft.DataMigration** 을 클릭합니다.

5. **Microsoft.DataMigration** 이 등록되어 있지 않으면 **등록** 을 클릭하고 **상태** 가 **등록됨** 으로 변경될 때까지 기다립니다. 변경된 상태를 확인하려면 **새로 고침** 을 클릭해야 할 수 있습니다.

6. **리소스 만들기** 를 클릭하고 **마켓플레이스 검색** 상자에 **Azure Database Migration Service** 를 입력한 후에 Enter 키를 누릅니다.

7. **Azure Database Migration Service** 페이지에서 **만들기** 를 클릭합니다.

8. **Migration Service 만들기** 페이지에서 다음 세부 정보를 입력하고 **만들기** 를 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 서비스 이름 | adventureworks_migration_service |
    | 구독 | 자신의 구독 선택 |
    | 리소스 그룹 선택 | Azure Database for PostgreSQL 서비스 및 Azure 가상 머신에 사용한 것과 같은 리소스 그룹 지정 |
    | 위치 | 가장 가까운 위치 선택 |
    | 가상 네트워크 | 설정 과정에서 만든 **postgresqlvnet/posgresqlvmSubnet** 가상 네트워크 선택 |
    | 가격 책정 계층 | 프리미엄(vCore 4개) |

9. Database Migration Service가 만들어지는 동안 기다립니다. 몇 분 정도 걸립니다.

10. Azure Portal에서 Database Migration Service 페이지로 이동합니다.

11. **새 마이그레이션 프로젝트** 를 클릭합니다.

12. **새 마이그레이션 프로젝트** 페이지에서 다음 세부 정보를 입력하고 **활동 만들기 및 실행** 을 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 프로젝트 이름 | adventureworks_migration_project |
    | 원본 서버 유형 | PostgreSQL |
    | PostgreSQL의 대상 데이터베이스 | Azure Database for PostgreSQL |
    | 활동 유형 선택 | 온라인 데이터 마이그레이션 |

13. **마이그레이션 마법사** 가 시작되면 **원본 세부 정보 추가** 페이지에서 다음 세부 정보를 입력하고 **저장** 을 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 원본 서버 이름 | nn.nn.nn.nn *(PostgreSQL을 실행하는 Azure 가상 머신의 IP 주소)* |
    | 서버 포트 | 5432 |
    | 데이터베이스 | adventureworks |
    | 사용자 이름 | azureuser |
    | 암호 | Pa55w.rd |

14. **대상 세부 정보** 페이지에서 다음 세부 정보를 입력하고 **저장** 을 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 대상 서버 이름 | adventureworks[nnn].postgres.database.azure.com |
    | 데이터베이스 | azureadventureworks |
    | 사용자 이름 | azureuser@adventureworks[nnn] |
    | 암호 | Pa55w.rd |

15. **대상 데이터베이스에 매핑** 페이지에서 **adventureworks** 데이터베이스를 선택하여 **azureadventureworks** 에 매핑합니다. **postgres** 데이터베이스의 선택을 취소합니다. **저장** 을 클릭합니다.

16. **마이그레이션 설정** 페이지에서 **adventureworksworks** 드롭다운을 확장하고 **고급 온라인 마이그레이션 설정 드롭다운** 을 확장한 후에 **병렬로 로드할 인스턴스의 최대 수** 가 5로 설정되어 있는지 확인하고 **저장** 을 클릭합니다.

17. **마이그레이션 요약** 페이지의 **활동 이름** 상자에 **AdventureWorks_Migration_Activity** 를 입력하고 **마이그레이션 실행** 을 클릭합니다.

18. **AdventureWorks_Migration_Activity** 페이지에서 15초 간격으로 **새로 고침** 을 클릭합니다. 마이그레이션 작업이 진행되면 작업 상태가 표시됩니다. **마이그레이션 세부 정보** 열의 상태가 **중단 준비 완료** 로 변경될 때까지 기다립니다.

19. Cloud Shell로 다시 전환합니다.

20. 다음 명령을 실행하여 **azureadventureworks** 데이터베이스에서 외래 키를 다시 만듭니다. **addkeys.sql** 스크립트는 앞에서 생성했습니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f addkeys.sql
```

외래 키가 추가되면 일련의 **ALTER TABLE** 문이 표시됩니다. *SpecialOfferProduct* 테이블 관련 오류가 표시될 수도 있는데 일단은 무시해도 됩니다. 이 오류는 UNIQUE 제약 조건이 올바르게 전송되지 않아서 발생하는 것입니다. 실제 환경에서는 다음 쿼리를 사용하여 원본 데이터베이스에서 이 제약 조건의 세부 정보를 검색해야 합니다.

```SQL
SELECT constraint_type, table_schema, table_name, constraint_name
FROM information_schema.table_constraints
WHERE constraint_type = 'UNIQUE';
```

그러고 나면 Azure Database for PostgreSQL의 대상 데이터베이스에서 이 제약 조건을 수동으로 복구할 수 있습니다.

다른 오류는 발생하지 않아야 합니다.

## 태스크 5. 데이터 수정 및 새 데이터베이스로의 단독형 마이그레이션 수행

1. Azure Portal의 **AdventureWorks_Migration_Activity** 페이지로 돌아옵니다.

1. **adventureworks** 데이터베이스를 클릭합니다.

1. **adventureworks** 페이지에서 모든 테이블의 상태가 **완료됨** 으로 표시되는지 확인합니다.

1. **증분 데이터 동기화** 를 클릭합니다. 모든 테이블의 상태가 **동기화** 로 표시되는지 확인합니다.

1. Cloud Shell로 다시 전환합니다.

1. 다음 명령을 실행하여 가상 머신에서 PostgreSQL을 사용하여 실행 중인 **adventureworks** 데이터베이스에 연결합니다.

```bash
psql -h nn.nn.nn.nn -U azureuser -d adventureworks
```

1. 다음 SQL 문을 실행하여 데이터베이스에서 주문 43659, 43660, 43661을 표시한 다음 제거합니다. 데이터베이스는 *salesorderheader* 테이블에서 관련 항목 삭제를 구현합니다. 그러면 *salesorderdetail* 테이블에서 해당 행이 자동으로 삭제됩니다.

```SQL
SELECT * FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
SELECT * FROM sales.salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
DELETE FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
```

1. **\q** 명령을 실행하여 *psql* 유틸리티를 닫습니다.

1. Azure Portal의 **adventureworks** 페이지로 돌아온 다음 *salesorderheader* 및 *salesorderdetail* 테이블의 페이지로 스크롤합니다. **새로 고침** 을 클릭합니다. *salesorderheader* 테이블에서는 행 3개가 삭제되었고 **sales.salesorderdetail** 테이블에서는 행 29개가 제거되었다는 메시지를 확인합니다.

1. **중단 시작** 을 클릭합니다.

1. **중단 완료** 페이지에서 **확인** 을 선택하고 **적용** 을 클릭합니다. 상태가 **완료됨** 으로 변경될 때까지 기다립니다.

1. Cloud Shell로 돌아옵니다.

1. 다음 명령을 실행하여 Azure Database for PostgreSQL 서비스를 통해 실행 중인 **azureadventureworks** 데이터베이스에 연결합니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
```

1. 다음 SQL 문을 실행하여 데이터베이스의 주문 및 주문 세부 정보를 표시합니다. 각 테이블의 첫 페이지가 표시되고 나면 실행을 끝냅니다. 이러한 쿼리는 데이터가 전송되었음을 표시하기 위해 실행하는 것입니다.

```SQL
SELECT * FROM sales.salesorderheader;
SELECT * FROM sales.salesorderdetail;
```

1. 다음 SQL 문을 실행하여 주문 43659, 43660, 43661 및 해당 세부 정보를 표시합니다.

```SQL
SELECT * FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
SELECT * FROM sales.salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
```

두 쿼리에서 모두 행 0개가 반환되어야 합니다.

1. **\q** 명령을 실행하여 *psql* 유틸리티를 닫습니다.

### 태스크 6: Azure Database for PostgreSQL에서 데이터베이스 확인

1. 온-프레미스 컴퓨터로 사용되는 가상 머신으로 돌아옵니다.

1. **pgAdmin4** 도구로 전환합니다.

1. 왼쪽 창에서 **서버** 를 마우스 오른쪽 단추로 클릭하고 **만들기**, **서버** 를 차례로 클릭합니다.

1. **만들기 - 서버** 대화 상자에서 **일반** 탭의 **이름** 상자에 **Azure Database for PostgreSQL** 을 입력한 다음 **연결** 탭을 클릭합니다.

1. 다음 세부 정보를 입력하고 **SSL** 탭을 클릭합니다.

    | 속성  | 값  |
    |---|---|
    | 호스트 이름/주소 | adventureworks*[nnn]*.postgres.database.azure.com |
    | 포트 | 5432 |
    | 유지 관리 데이터베이스 | postgres |
    | 사용자 이름 | azureuser@adventureworks*[nnn]* |
    | 암호 | Pa55w.rd |
    | 암호 저장 | 선택 |
    | 역할 | *비워 둠* |
    | 서비스 | *비워 둠* |

1. **SSL** 탭에서 **SSL 모드** 를 **필수** 로 설정하고 **저장** 을 클릭합니다.

1. **pgAdmin4** 창 왼쪽 창의 **서버** 아래에서 **Azure Database for PostgreSQL** 을 확장합니다.

1. **데이터베이스**, **azureadventureworks** 를 차례로 확장하고 데이터베이스의 스키마와 테이블을 찾아봅니다. 테이블이 온-프레미스 데이터베이스의 테이블과 같아야 합니다.

### 태스크 7: Azure Database for PostgreSQL의 데이터베이스용으로 샘플 애플리케이션 다시 구성 및 테스트

1. **터미널** 창으로 돌아옵니다.

2. *nano* 편집기에서 테스트 애플리케이션의 App.config 파일을 엽니다.

```bash
nano App.config
```

3. **ConnectionString** 설정의 값을 변경하여 Azure 가상 머신의 IP 주소를 **adventureworks[nnn].postgres.database.azure.com** 으로 바꿉니다.  **Database** 는 **azureadventureworks** 로 변경합니다. **User Id** 는 **azureuser@adventureworks[nnn]** 으로 변경합니다. 연결 문자열 끝에 **Ssl Mode=Require;** 텍스트를 추가합니다. 파일 내용이 다음과 같아야 합니다.

```XML
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <appSettings>
      <add key="ConnectionString" value="Server=adventureworks[nnn].postgres.database.azure.com;Database=azureadventureworks;Port=5432;User Id=azureuser@adventureworks[nnn];Password=Pa55w.rd;Ssl Mode=Require;" />
    </appSettings>
</configuration>
```

이제 애플리케이션이 Azure 가상 머신에서 실행되는 데이터베이스에 연결됩니다.

4. 파일을 저장하고 편집기를 닫으려면 Esc 키를 누른 후 Ctrl+X를 누릅니다. 메시지가 표시되면 Enter 키를 눌러 변경 내용을 저장합니다.

5. 애플리케이션을 빌드하고 실행합니다.

```bash
dotnet run
```

애플리케이션이 데이터베이스에 연결되기는 하지만 오류가 발생하며 **materialized view "vproductanddescription" has not been populated** 라는 메시지가 표시됩니다. 데이터베이스에서 구체화된 보기를 새로 고쳐야 합니다.

6. 터미널 창에서 *psql* 유틸리티를 사용하여 azureadventureworks 데이터베이스에 연결합니다.

```bash
psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
```

7. 다음 쿼리를 실행하여 데이터베이스의 모든 구체화된 보기 목록을 표시합니다.

```SQL
SELECT schemaname, matviewname, matviewowner, ispopulated
FROM pg_matviews;
```

이 쿼리는 *person.vstateprovincecountryregion* 및 *production.vproductanddescription* 에 해당하는 행 2개를 반환해야 합니다. 두 보기의 *ispopulated* 열 상태는 *f* (아직 내용이 입력되지 않음)입니다.

8. 다음 문을 실행하여 두 보기에 내용을 입력합니다.

```SQL
REFRESH MATERIALIZED VIEW person.vstateprovincecountryregion;
REFRESH MATERIALIZED VIEW production.vproductanddescription;
```

9.  **\q** 명령을 실행하여 *psql* 유틸리티를 닫습니다.

10. 애플리케이션을 다시 실행합니다.

```bash
dotnet run
```

이제 애플리케이션이 정상적으로 실행되는지 확인합니다.

지금까지 Azure Database for PostgreSQL로 데이터베이스를 마이그레이션하고 새 데이터베이스를 사용하도록 애플리케이션을 다시 구성했습니다.
