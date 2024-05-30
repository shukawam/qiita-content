---
title: 今更ながらSpring Data JPAを触ってみる
tags:
  - Java
  - spring
  - spring-data-jpa
  - SpringBoot
private: false
updated_at: '2021-09-28T20:32:13+09:00'
id: 6e379df031dccebddd36
organization_url_name: null
slide: false
ignorePublish: false
---
# 始めに

[Doma](https://doma.readthedocs.io/en/2.19.2/)以外の OR Mapper を触ったことがなかったので、ふと思い立ち興味半分くらいで触ってみました。Spring Boot アプリケーションへの導入方法 ～ 簡単な API 作成で感じた Doma2 との比較について書きたいと思います。



# 環境

- IDE VSCode
- Java 11.0.6
- Spring Boot  2.3.1
- PostgreSQL 11.6

# 前提

簡単な API を作成して、色々と触ってみたいと思います。作成する API は以下の様。

| エンドポイント               | Http Method | 概要                                       | 備考                               |
| ---------------------------- | ----------- | ------------------------------------------ | ---------------------------------- |
| `/api/employee/{employeeId}` | GET         | 従業員IDに一致する従業員の情報を取得する。 |                                    |
| `/api/employee`              | GET         | 従業員情報を取得する。                     | 検索条件による絞り込みも実施する。 |
| `/api/employee`              | POST        | 従業員情報を登録する。                     |                                    |
| `/api/employee/{employeeId}` | PUT         | 従業員情報を更新する。                     |                                    |
| `/api/employee/{employeeId}` | DELETE      | 従業員情報を削除する。                     |                                    |



# API作成

## （一応）アプリケーションのひな型作成

[Spring Initializer Java Support](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-spring-initializr)というVSCodeプラグインを使用してアプリケーションのひな型を作ります。
このプラグイン自体は、[Spring Boot Extension Pack](https://marketplace.visualstudio.com/items?itemName=Pivotal.vscode-boot-dev-pack)に含まれているので[Spring Boot Extension Pack](https://marketplace.visualstudio.com/items?itemName=Pivotal.vscode-boot-dev-pack)をインストールしてもらえば十分です。
対話形式で、ひな型を作成します。コマンドパレットから、「Spring Initializr: Generate a Gradle Project」を選択する。

![create-application-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/aec0c4b6-1b85-6153-6d93-9ae8aac2301e.png)

Java を選択する。

![create-application-02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/ed29c0fe-c482-14f2-288f-482792075054.png)

パッケージ名を入力する。今回は、デフォルトで`com.example`のままとする。

![create-application-03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/b37e7952-2dfb-5fe3-0d32-3f1cbfbc4263.png)

プロジェクト名を入力する。お好きな名前をどうぞ。（私は、employee-api としました。）

![create-application-04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/baac0d27-d07a-1569-3a7b-ada72e7d86aa.png)

Spring Boot のバージョンを選択する。（2.3.1を選択）

![create-application-05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d60d7509-37cd-3137-01a1-1c33fc7f9658.png)

依存ライブラリを選択する。簡単な API を作成したかっただけなので以下のライブラリを選択しています。

- Spring Boot DevTools
- Lombok, Spring Web
- Spring Data JPA
- PostgreSQL Driver
- (Lombok)

![create-application-06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/a163012a-6cc2-cef7-faa7-fe13b9525f2c.png)

## `application.properties`に接続情報を定義する

```properties
spring.jpa.database=postgresql
spring.datasource.platform=postgres
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.url=jdbc:postgresql://localhost:5432/sample
spring.datasource.username=postgres
spring.datasource.password=postgres
```

## Entityクラスを定義する

テーブルの共通項目として、`insert_date`, `update_date`を定義しています。

```CommonEntity.java
/**
 * テーブルの共通項目を定義したクラスです。</br>
 * 全てのEntityクラスはこのクラスを継承して作成します。
 */
@MappedSuperclass
@Getter
@Setter
public class CommonEntity {

  /** データ登録日時 */
  @Column(name = "insert_date")
  @Temporal(TemporalType.DATE)
  private Date insertdate;

  /** データ更新日時 */
  @Column(name = "update_date")
  @Temporal(TemporalType.DATE)
  private Date updateDate;

  /**
   * データ登録前に共通的に実行されるメソッド
   */
  @PrePersist
  public void preInsert() {
    Date date = new Date();
    setInsertdate(date);
    setUpdateDate(date);
  }

  /**
   * データ更新前に共通的に実行されるメソッド
   */
  @PreUpdate
  public void preUpdate() {
    setUpdateDate(new Date());
  }

}
```

テーブル共通項目を定義した`CommonEntity`を継承し、業務用の Entity クラスを作成します。

```EmployeeEntity.java
@Entity
@Table(name = "employee")
@Getter
@Setter
public class EmployeeEntity extends CommonEntity {

  /** 従業員ID */
  @Id
  @Column(name = "id")
  @GeneratedValue(strategy = GenerationType.AUTO)
  private Integer employeeId;
  
  /** 従業員名 */
  @Column(name = "name")
  private String employeeName;

  /** 年齢 */
  @Column(name = "age")
  private Integer age;

  /** 役職ID */
  @Column(name = "position_id")
  private String positionId;

  /** 所属部署ID */
  @Column(name = "department_id")
  private String departmentId;

}
```

## Repository インタフェースを定義する

`org.springframework.data.jpa.repository.JpaRepository`を継承したインタフェースを定義します。

```EmployeeRepository.java
package com.example.employeeapi.employee;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface EmployeeRepository extends JpaRepository<EmployeeEntity, Integer> {

}
```

### （参考）`JpaRepository`

基本的なCRUD操作を扱えるようなメソッドが用意されています。このインタフェースを継承したインタフェース（今回の例だと、`EmployeeRepository`）では、業務仕様等で予め用意されたメソッドで不十分な場合に独自のメソッドを定義することができます。（Join してレコードを取得するなど）

```JpaRepository.java
/*
 * Copyright 2008-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.springframework.data.jpa.repository;

import java.util.List;

import javax.persistence.EntityManager;

import org.springframework.data.domain.Example;
import org.springframework.data.domain.Sort;
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.repository.query.QueryByExampleExecutor;

/**
 * JPA specific extension of {@link org.springframework.data.repository.Repository}.
 *
 * @author Oliver Gierke
 * @author Christoph Strobl
 * @author Mark Paluch
 */
@NoRepositoryBean
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#findAll()
	 */
	@Override
	List<T> findAll();

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.PagingAndSortingRepository#findAll(org.springframework.data.domain.Sort)
	 */
	@Override
	List<T> findAll(Sort sort);

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#findAll(java.lang.Iterable)
	 */
	@Override
	List<T> findAllById(Iterable<ID> ids);

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#save(java.lang.Iterable)
	 */
	@Override
	<S extends T> List<S> saveAll(Iterable<S> entities);

	/**
	 * Flushes all pending changes to the database.
	 */
	void flush();

	/**
	 * Saves an entity and flushes changes instantly.
	 *
	 * @param entity
	 * @return the saved entity
	 */
	<S extends T> S saveAndFlush(S entity);

	/**
	 * Deletes the given entities in a batch which means it will create a single {@link Query}. Assume that we will clear
	 * the {@link javax.persistence.EntityManager} after the call.
	 *
	 * @param entities
	 */
	void deleteInBatch(Iterable<T> entities);

	/**
	 * Deletes all entities in a batch call.
	 */
	void deleteAllInBatch();

	/**
	 * Returns a reference to the entity with the given identifier. Depending on how the JPA persistence provider is
	 * implemented this is very likely to always return an instance and throw an
	 * {@link javax.persistence.EntityNotFoundException} on first access. Some of them will reject invalid identifiers
	 * immediately.
	 *
	 * @param id must not be {@literal null}.
	 * @return a reference to the entity with the given identifier.
	 * @see EntityManager#getReference(Class, Object) for details on when an exception is thrown.
	 */
	T getOne(ID id);

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.query.QueryByExampleExecutor#findAll(org.springframework.data.domain.Example)
	 */
	@Override
	<S extends T> List<S> findAll(Example<S> example);

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.query.QueryByExampleExecutor#findAll(org.springframework.data.domain.Example, org.springframework.data.domain.Sort)
	 */
	@Override
	<S extends T> List<S> findAll(Example<S> example, Sort sort);
}

```



## Service, Controller クラスを定義する

先ほど定義した`EmployeeRepository`を使用するServiceクラスとそれを呼び出すControllerクラスを定義します。

```EmployeeService.java
package com.example.employeeapi.employee;

import java.util.ArrayList;
import java.util.List;

import javax.transaction.Transactional;

import com.example.employeeapi.employee.dto.Employee;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
@Transactional
public class EmployeeService {

  @Autowired
  private EmployeeRepository employeeRepository;

  public Employee getEmployeeById(String employeeId) {
    EmployeeEntity entity = employeeRepository.findById(Integer.parseInt(employeeId)).get();
    Employee employee = new Employee();
    copyEntityToBean(entity, employee);
    return employee;
  }

  public List<Employee> getEmployeeList() {
    List<Employee> employees = new ArrayList<>();
    List<EmployeeEntity> employeeEntityList = employeeRepository.findAll();
    employeeEntityList.forEach(entity -> {
      Employee employee = new Employee();
      copyEntityToBean(entity, employee);
      employees.add(employee);
    });
    return employees;
  }

  public Employee createEmployee(Employee employee) {
    EmployeeEntity entity = new EmployeeEntity();
    copyBeanToEntityForInsert(employee, entity);
    EmployeeEntity createdEntity = employeeRepository.save(entity);
    Employee newEmployee = new Employee();
    copyEntityToBean(createdEntity, newEmployee);
    return newEmployee;
  }

  public Employee updateEmployee(Employee employee) {
    EmployeeEntity entity = new EmployeeEntity();
    copyBeanToEntityForUpdate(employee, entity);
    EmployeeEntity updatedEntity = employeeRepository.save(entity);
    Employee updatedEmployee = new Employee();
    copyEntityToBean(updatedEntity, updatedEmployee);
    return updatedEmployee;
  }

  public boolean deleteEmployeeById(String employeeId) {
    employeeRepository.deleteById(Integer.parseInt(employeeId));
    return true;
  }

  private void copyEntityToBean(EmployeeEntity entity, Employee employee) {
    // サンプルのため、簡略的にコピーをする。
    // 綺麗にやるのであれば、BeanUtils#copyPropertiesなどを使用してください。
    employee.setId(String.valueOf(entity.getEmployeeId()));
    employee.setName(entity.getEmployeeName());
    employee.setAge(String.valueOf(entity.getAge()));
    employee.setPositionId(entity.getPositionId());
    employee.setDepartmentId(entity.getDepartmentId());
    employee.setInsertDate(String.valueOf(entity.getInsertdate()));
    employee.setUpdateDate(String.valueOf(entity.getUpdateDate()));
  }

  private void copyBeanToEntityForInsert(Employee employee, EmployeeEntity entity) {
    // サンプルのため、簡略的にコピーをする。
    // 綺麗にやるのであれば、BeanUtils#copyPropertiesなどを使用してください。
    if (!"".equals(employee.getName())) {
      entity.setEmployeeName(employee.getName());
    }
    if (!"".equals(employee.getAge())) {
      entity.setAge(Integer.parseInt(employee.getAge()));
    }
    if (!"".equals(employee.getPositionId())) {
      entity.setPositionId(employee.getPositionId());
    }
    if (!"".equals(employee.getDepartmentId())) {
      entity.setDepartmentId(employee.getDepartmentId());
    }
  }

  private void copyBeanToEntityForUpdate(Employee employee, EmployeeEntity entity) {
    // サンプルのため、簡略的にコピーをする。
    // 綺麗にやるのであれば、BeanUtils#copyPropertiesなどを使用してください。
    entity.setEmployeeId(Integer.parseInt(employee.getId()));
    copyBeanToEntityForInsert(employee, entity);
  }

}
```

```EmployeeController.java
package com.example.employeeapi.employee;

import java.util.List;

import com.example.employeeapi.common.dto.HttpResponseDto;
import com.example.employeeapi.employee.dto.Employee;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class EmployeeController {

  @Autowired
  private EmployeeService employeeService;

  @GetMapping(path = "/api/employee/{employeeId}")
  public HttpResponseDto getEmployeeById(@PathVariable("employeeId") String employeeId) {
    HttpResponseDto httpResponseDto = new HttpResponseDto();
    Employee employee = employeeService.getEmployeeById(employeeId);
    httpResponseDto.setHttpStatus(HttpStatus.OK);
    httpResponseDto.setResponseData(employee);
    return httpResponseDto;
  }

  @GetMapping(path = "/api/employee")
  public HttpResponseDto getEmployeeList() {
    HttpResponseDto httpResponseDto = new HttpResponseDto();
    List<Employee> employees = employeeService.getEmployeeList();
    httpResponseDto.setHttpStatus(HttpStatus.OK);
    httpResponseDto.setResponseData(employees);
    return httpResponseDto;
  }

  @PostMapping(path = "/api/employee")
  public HttpResponseDto createEmployee(@RequestBody Employee employee) {
    HttpResponseDto httpResponseDto = new HttpResponseDto();
    Employee newEmployee = employeeService.createEmployee(employee);
    httpResponseDto.setHttpStatus(HttpStatus.CREATED);
    httpResponseDto.setResponseData(newEmployee);
    return httpResponseDto;
  }

  @PutMapping(path = "/api/employee/{employeeId}")
  public HttpResponseDto updateEmployee(@PathVariable("employeeId") String emplyeeId, @RequestBody Employee employee) {
    HttpResponseDto httpResponseDto = new HttpResponseDto();
    employee.setId(emplyeeId);
    Employee updatedEmployee = employeeService.updateEmployee(employee);
    httpResponseDto.setHttpStatus(HttpStatus.CREATED);
    httpResponseDto.setResponseData(updatedEmployee);
    return httpResponseDto;
  }

  @DeleteMapping(path = "/api/employee/{employeeId}")
  public HttpResponseDto deleteEmployee(@PathVariable("employeeId") String employeeId) {
    HttpResponseDto httpResponseDto = new HttpResponseDto();
    if (employeeService.deleteEmployeeById(employeeId)) {
      httpResponseDto.setHttpStatus(HttpStatus.OK);
      httpResponseDto.setMessage("delete success.");
    } else {
      // do something
    }
    return httpResponseDto;
  }
}
```

（2020/09/04 追記）
また、クライアントへ値を返却するためのクラスを定義します。

```Employee.java
package com.example.demo.employee;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class Employee {

	private String id;

	private String name;

	private String age;

	private String departmentId;

	private String positionId;

	private String insertDate;

	private String updateDate;
}

```

```HttpResponseDto.java
package com.example.demo.common;

import org.springframework.http.HttpStatus;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class HttpResponseDto {

	private HttpStatus httpStatus;

	private String message;

	private Object responseData;
}

```

# 他のOR Mapperとの比較（主観）

他のOR Mapperとの比較と書きましたが、Domaとの比較です。

- 良いと感じた点
  - 簡単なCRUD操作であれば、提供されているAPIを使用するだけで実現できる。
    - Domaも検索（SELECT）以外は、APIが提供されているが、検索処理の場合は単純なクエリーであってもSQLファイルを作成する必要がある。
      - `JpaRepository`から提供されているAPI程度であれば[Doma Gen](https://doma-gen.readthedocs.io/en/2.6.1/gen/)で何とかなりそうな気もするが、、
  - 個人的によく使用している[TypeORM](https://typeorm.io/#/)というTypeScript用のOR Mapperに使用感が似ている。（TypeORMがJPAを意識して作ったのかな、、詳しい方いたら教えてください。）
- いまいちだと感じた点
  - 簡単なCRUD操作程度しか作っていないので、現状特に不満はありません。
  - ただし、元はJPAなのでしっかりと学習してから採用しないと痛い目を見そうな予感

# （補足）Database構築

検証用のDBはDockerベースで構築しています。コピペで動くのでDB作るのめんどいって方は使ってください。

```
$ tree
.
├── docker-compose.yml
└── init-script
    ├── 01_create_table.sql
    └── 02_insert_data.sql
```



```docker-compose.yml
version: '3'
volumes:
  db_data:
services:
  database:
    image: postgres:11.6
    container_name: postgres
    ports:
      - 5432:5432
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./init-script:/docker-entrypoint-initdb.d
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: sample

```

```01_create_table.sql
create table department (
  --部署コード
  id varchar(3) primary key,
  -- 部署名
  name varchar(50),
  -- データ投入日
  insert_date date,
  -- データ更新日
  update_date date
);

create table "position" (
  -- 役職ID
  id varchar(2) primary key,
  -- 役職名
  name varchar(20),
  -- データ投入日
  insert_date date,
  -- データ更新日
  update_date date
);

-- table生成
create table "employee" (
  -- 従業員番号
  id serial primary key,
  -- 従業員名
  name varchar(50),
  -- 年齢
  age integer,
  -- 役職
  position_id varchar(2) references position(id),
  -- 所属部署id
  department_id varchar(3) references department(id),
  -- データ投入日
  insert_date date,
  -- データ更新日
  update_date date
);
```



```02_insert_data.sql
insert into department (id, name, insert_date, update_date)
values ('001', '人事部', '2020-06-17', '2020-06-17');
insert into department (id, name, insert_date, update_date)
values ('002', '総務部', '2020-06-17', '2020-06-17');
insert into department (id, name, insert_date, update_date)
values ('003', '開発部', '2020-06-17', '2020-06-17');
insert into department (id, name, insert_date, update_date)
values ('004', '広報部', '2020-06-17', '2020-06-17');
insert into position (id, name, insert_date, update_date)
values ('01', '部長', '2020-06-17', '2020-06-17');
insert into position (id, name, insert_date, update_date)
values ('02', '課長', '2020-06-17', '2020-06-17');
insert into position (id, name, insert_date, update_date)
values ('03', '一般', '2020-06-17', '2020-06-17');
insert into employee (
    name,
    age,
    position_id,
    department_id,
    insert_date,
    update_date
  )
values (
    'しゃっちょさん',
    50,
    '01',
    '001',
    '2020-06-17',
    '2020-06-17'
  );
insert into employee (
    name,
    age,
    position_id,
    department_id,
    insert_date,
    update_date
  )
values (
    'ぶっちょさん',
    46,
    '02',
    '001',
    '2020-06-17',
    '2020-06-17'
  );
insert into employee (
    name,
    age,
    position_id,
    department_id,
    insert_date,
    update_date
  )
values (
    'かっちょさん',
    30,
    '03',
    '001',
    '2020-06-17',
    '2020-06-17'
  );
insert into employee (
    name,
    age,
    position_id,
    department_id,
    insert_date,
    update_date
  )
values (
    'ぱんぴーさん',
    30,
    '03',
    '002',
    '2020-06-17',
    '2020-06-17'
  );
```



# 終わりに

実際のユースケースを想像して、もう少し実務チックなAPIを作ってみたいと思います。



# 参考

- [SpringBoot + Spring JPAでデータベースに接続する](https://qiita.com/t-iguchi/items/685c0a1bb9b0e8ec68d2#comments)

- [spring boot + spring jpaでデータベースに接続しCRUD操作](https://qiita.com/kuro227/items/a16e22ac12afe7442a3d#update--delete)

- [Java ORマッパー選定のポイント](https://www.slideshare.net/masatoshitada7/java-or-jsug)
