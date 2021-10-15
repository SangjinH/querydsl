# QueryDSL 공부하기

>- 설정
>
>- 사용법
>
>  - [`Boolean Builer`](`Boolean Builder`)
>
>  - ##### [`Where절`](`Where 절`)
>
>  - [`QueryProjection`](QueryProjection)

## 1.설정

`build.gradle` 폴더에 해당 부분들 추가. 
```text
plugins {
	//querydsl 추가
	id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
	}

dependencies {
	//querydsl 추가
	implementation 'com.querydsl:querydsl-jpa'

}

//querydsl 추가 시작
def querydslDir = "$buildDir/generated/querydsl"
querydsl {
	jpa = true
	querydslSourcesDir = querydslDir
}
sourceSets {
	main.java.srcDir querydslDir
}
configurations {
	querydsl.extendsFrom compileClasspath
}
compileQuerydsl {
	options.annotationProcessorPath = configurations.querydsl
}
//querydsl 추가 끝

```



## 2.실제 사용법

- Querydsl 을 이용하는데 가장 큰 장점은 `동적쿼리의 생성`이다.
   * 동적쿼리 생성의 방법 중 가장 대표적인 두 가지는 `BooleanBuilder` 와 `where절` 을 이용하는 것이다. 



### 	`Boolean Builder`

```java
    @Test
    public void dynamicQuery_BooleanBuilder() {
        String usernameParam = "member1";
        Integer ageParam = null;

        List<Member> result = searchMember1(usernameParam, ageParam);
        assertThat(result.size()).isEqualTo(1);
    }

    private List<Member> searchMember1(String usernameCond, Integer ageCond) {
        BooleanBuilder builder = new BooleanBuilder();
        if (usernameCond != null) {
            builder.and(member.username.eq(usernameCond));
        }

        if (ageCond != null) {
            builder.and(member.age.eq(ageCond));
        }

        return queryFactory
                .selectFrom(member)
                .where(builder)
                .fetch();
    }
```

- 위 코드 중 `searchMember1` 메서드가 동적쿼리를 생성하는 역할을 한다.

  `BooleanBuilder` 는 여러가지 조건을 조합할 수 있는 Type으로 Java코드의 장점을 갖고 있다. 

  따라서 인자로 들어온 `usernameCond` 와 `ageCond` 를 각각 `member.username.eq` 와 `member.age.eq` 로 찾은 후

  조립하는 과정이다. 

- **장점** 

  - 보다시피, 여러개의 조건을 한 번에 조립할 수 있다는 장점이 있고, 추가적인 조건을 등록하기 쉽다.

- **단점**

  - 가장 핵심적인 메소드, `searchMember1` 을 봤을 때 어떤 쿼리를 작성하는 것인지 마지막에 가서야 알 수 있다.

    한마디로, 한 번에 알아보기에는 어렵다. 



### 	`Where 절`

```java
@Test
    public void dynamicQuery_WhereParam() {
        String usernameParam = "member1";
        Integer ageParam = 10;

        List<Member> result = searchMember2(usernameParam, ageParam);
        assertThat(result.size()).isEqualTo(1);
    }

    private List<Member> searchMember2(String usernameCond, Integer ageCond) {
        return queryFactory
                .selectFrom(member)
                .where(usernameEq(usernameCond), ageEq(ageCond))
//                .where(allEq(usernameCond, ageCond))
                .fetch();
    }

    private BooleanExpression usernameEq(String usernameCond) {
        return usernameCond == null ? null : member.username.eq(usernameCond);}

    private BooleanExpression ageEq(Integer ageCond) {
        return ageCond == null ? null : member.age.eq(ageCond);}

    private BooleanExpression allEq(String usernameCond, Integer ageCond) {
        return usernameEq(usernameCond).and(ageEq(ageCond));
    }
```

- 위 내용 중 `selectMember2()` 메서드에 관련된 부분이다. 기존의 BooleanBuilder와는 다르게 

  기존 QueryFactory를 이용해 조회할 때, 조건절인 `where`절에 해당 관련 메서드를 만들어

  없다면 `null`을 반환해 결국은 아무런 영향도 미치지 않게끔 하는 방법이다.



- 장점

  - 기본적으로 코드를 봤을 때 길지 않다. 가독성이 좋다. 진짜 작동하는 코드가 무엇인지 한 눈에 확인할 수 있다.

  - 조합이 가능하다. 

    - 예로는 광고관련 BackEnd에서 검증하는 로직을 넣을때 

      `isServicable()` 이라는 메서드를 `isValid()` , `isExist()` 라는 메서드를 조합해,

      보여줘야할 광고를 찾을때, 유효한 기업인지와, 존재하는 기업인지에 대한 부분을 합쳐

      간결하게 표현할 수 있다. 

- 단점

  - `Null` 체크를 잘 해줘야한다. 모든 메서드마다..!



## QueryProjection

> 일단 QueryProjection은 Dto에 적용하는 개념이다. 
>
> 예로는 
>
> ```java
> @Data
> public class MemberTeamDto {
>     // columns ...
>     
>     @QueryProjection
>     public MemberTeamDto(Long memberId, ...) {
>         this.memberId = memberId;
>         // ...
>     }
> }
> ```
>
> 위 `@QueryProjection` 어노테이션을 달아주면 QueryFactory에서 바로 적용이 가능하다. 주의할 점은, 
>
> gradle에서 `compileQuerydsl` 을 실행시켜 `QMemberTeamDto` 를 만들고,
>
>  사용 `QueryFactory에서` 해당 `column과 이름을 맞춰줘야` 한다!

- ### 실제 사용 Repository 

```java
public List<MemberTeamDto> searchByBuiler(){
    // Boolean Builder로 여러 조건을 Binding 
    return queryFactory
        	.select(new QMemberTeamDto(
                member.id.as("memberId"),
                member.username,
                member.age,				// QMember.member
                team.id.as("teamId"),    // QTeam.team
                team.name.as("teamName")
            ))
        	.from(member)
        	.leftJoin(member.team, team)
        	.where(builder)
        	.fetch();
}
```

