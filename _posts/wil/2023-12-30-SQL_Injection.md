---
title: SQL Injection이란 무엇이며 어떻게 방어할 수 있을까?
excerpt: SQL 인젝션은 입력 폼을 통해 악의적인 스크립트 또는 SQL쿼리를 삽입함으로써 이루어집니다. 이를 방어하기 위해 매개변수화된 쿼리, ORM사용, 이스케이프 문자 사용, 입력 값 정제 등의 방법을 사용할 수 있습니다.
categories: [What I Learned]
tags: [SQL-Injection, sql공격, 보안, sql인젝션, 매개변수화쿼리, ORM, 이스케이프]
image:
  path: /assets/img/wil/covers/xss.png
---
## 🔍 SQL Injection

많은 사이트에서 폼을 이용한 로그인, 게시글 작성 등을 많이 볼 수 있습니다. SQL Injection은 입력 폼에 SQL 쿼리를 작성하는 것으로, 이러한 간단해 보이는 행위가 데이터베이스 시스템에 큰 영향을 줄 수 있습니다. (e.g 패스워드와 신용카드와 같은 개인정보 탈취)



### ✏️ SQL Injection 과정

1. HTTP request를 사용하는 입력 폼을 통해 사용자로부터 입력을 받는다.

2. 이때, 코드에 SQL 코드를 삽입한다. 예를 들어 `' OR 1 = 1'`과 같은 코드를 삽입하여 WHERE 절을 항상 참으로 만든다.

3. 웹 서버는 사용자의 요청을 받아 데이터베이스에 전달한다.

4. 데이터베이스는 전달받은 SQL 쿼리를 실행한다.

5. 데이터베이스는 쿼리의 결과를 반환한다. SQL Injection에 의해 노출되면 안 되는 데이터가 노출되고 이를 악용한다.

예를 들어, 로그인 폼에서 사용자 이름과 비밀번호를 입력하는 경우를 생각해 보겠습니다. 사용자 이름 필드에 `' OR '1'='1`와 같은 입력을 하면, SQL 쿼리는 `'SELECT * FROM users WHERE username = '' OR '1'='1'`로 변형됩니다. 이 쿼리는 항상 참이 되어 비밀번호 검증 없이 로그인이 허용될 수 있습니다.

## 🔍 SQL Injection 방어

SQL Injection을 방어하는 방법에 대해 알아보겠습니다.



### ✏️ 매개 변수화 쿼리 (Parameterized Statements) 사용

만약 다음과 같이 쿼리문을 작성하면 데이터베이스가 실행되기 전에 이미 구조화 되고, 항상 참이 되는 쿼리를 던짐으로써 모든 사용자에 대한 정보를 출력할 수 있습니다.

~~~java
Connection conn = DriverManager.getConnection(URL, ID, PASS);
Statement stmt = conn.createStatement();

// concat을 이용한 쿼리 작성
String sql = "SELECT * FROM users WHERE email = '" + email + "'";

// 쿼리 실행
ResultSet results = stmt.executeQuery(sql);
~~~




이를 해결하기 위해 매개 변수화 쿼리를 사용할 수 있습니다. 매개 변수화 쿼리란 쿼리에 매개 변수(placeholder)를 사용하여 동적으로 값을 전달하는 것을 말합니다. 다음 코드처럼 PreparedStatement를 이용하면 SQL Injection을 방지할 수 있습니다.

~~~java
// 데이터 베이스 연결
Connection conn = DriverManager.getConnection(URL, ID, PASS);

// 쿼리문 준비
String sql = "SELECT * FROM user WHERE email = ?";

// prepared statement 준비
PreparedStatement stmt = conn.prepareStatement(sql);

// 파라미터 값 바인딩
stmt.setString(1, "user@naver.com");
~~~

여기서 의문이 stmt.setString(1, "'1'='1'") 넘기면 결국엔 SQL Injection이 되는거 아닌가 싶은데, 그렇지 않습니다.
 - 데이터베이스 시스템은 매개 변수화된 쿼리는 문자열로 해석하지 않고, 값으로 취급합니다.
 - 좀 더 자세히 말하면 매개 변수화된 쿼리는 SQL 쿼리문은 쿼리로서 전달되고 이후에 동적으로 값을 집어넣습니다.
 - 즉, DBMS는 쿼리와 파라미터를 분리할 수 있으며 SQL Injection을 방지할 수 있습니다.



### ✏️ Object Relational Mapping (ORM) 사용

ORM이란 객체와 관계형 데이터베이스 간의 변환을 자동화해주는 기술을 말하며, 흔히 사용되는 JPA가 ORM이라고 할 수 있습니다. 
ORM을 사용하는 경우 쿼리문을 작성하지 않고 이 ORM 기술이 보통 엔티티와 그래프 탐색을 분석해서 작성해줍니다. 이때, 대부분의 ORM 기술은 매개 변수화 쿼리를 사용합니다.
하지만 ORM을 사용하더라도 쿼리를 직접 작성하는 경우에는 SQL Injection에 노출될 수 있습니다.(특히 문자열을 concat할 때 조심해야 합니다.)



### ✏️ 이스케이프 문자 사용

쿼리에 악용될 문자열(역슬래시, 작은 따옴표 등)을 이스케이프 문자열로 바꾸는 방법도 있으며 많은 언어들이 표준 함수로 기능을 제공하고 있습니다. 하지만 SQL Injection이 따옴표나 역슬래시로만 이루어지지 않기 때문에 완전히 막아주지는 못합니다.

`"SELECT * FROM user WHERE id = " + id;`

문자열 연결을 통해 id값을 그대로 연결하기 때문에 `'1; DROP TABLE user'` 와 같은 SQL Injection을 할 수 있습니다.



### ✏️ 입력 값 정제

입력 값에서 잠재적인 악성 코드나 취약한 문자열을 제거하는 것도 한 가지 방법이 될 수 있습니다.

문자열 필터링 
- 입력 값에 적합하지 않은 문자나 문자열 패턴을 필터링하여 제거한다. (HTML 태그, 스크립트 코드, SQL 예약어 등) <br>
- 형식 검사 : 입력 값의 범위 또는 제약 조건을 검사한다.

~~~java
public static String refineInput(String input) {
    if (input == null) {
        return null;
    }

    // SQL 예약어 필터링
    String[] sqlKeywords = {"SELECT", "INSERT", "DELETE", "UPDATE", "DROP", "EXECUTE", "UNION", "ALTER", "CREATE"};
    for (String keyword : sqlKeywords) {
        input = input.replaceAll("(?i)" + keyword, ""); // 대소문자 구분 없이 제거
    }

    // HTML 태그 제거
    input = input.replaceAll("<[^>]*>", "");

    // 스크립트 코드 제거 (옵션)
    input = input.replaceAll("(?i)<script.*?>.*?</script>", ""); // 대소문자 구분 없이 스크립트 태그 제거

    // 입력 값의 형식 및 범위 검사 (예: 이메일, 날짜 형식 검사)
    // 예시: 이메일 형식 검사
    if (!Pattern.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,6}$", input)) {
        // 형식이 맞지 않으면 빈 문자열 반환 또는 오류 처리
        return "";
    }

    return input;
}
~~~
