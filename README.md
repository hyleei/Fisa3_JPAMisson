# [우리FISA 3기 클라우드 엔지니어링] JPA 학습 컨텐츠
---

## 팀원 소개
|<img src="https://avatars.githubusercontent.com/u/79884688?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/104816148?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/127733525?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|
|박장우<br/>[@Lisiant](https://github.com/Lisiant)|박현서<br/>[@hyleei](https://github.com/hyleei)|구동길<br/>[@dkac0012](https://github.com/dkac0012)|

# 🏨 클래스 구조

### `Team.java`

```java
// Getter, Setter, Constructor Annotations...
@ToString
@Entity
public class Team {

	@Id
	@Column(name = "team_id")
	private long teamId;

	@NonNull
	@Column(name = "team_name", length = 20)
	private String teamName;

	@OneToMany(mappedBy = "teamId")
	public List<Member> members = new ArrayList<>();

}

```

### `Member.java`

```java
// Getter, Setter, Constructor Annotations...
@ToString
@Entity
public class Member {

	@Id
	@Column(name="member_id")
	private long memberId;
	
	@NonNull
	@Column(length = 20, nullable = false) 
	private String name;
	
	@NonNull
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name="team_id")  
	private Team team;
	
}

```

위 클래스 구조를 참고하여, 다음 문제를 해결하세요.

---

# 🥇 문제 1

```java
@Test
public void joinTest() {

	EntityManager em = null;
	
	try {
		em = DBUtil.getEntityManager();

		String jpql = "SELECT m FROM Member m";
		em.createQuery(jpql, Member.class).getResultStream().forEach(System.out::println);  
		
	} catch (Exception e) {
		e.printStackTrace();
	} finally {
		if (em != null) {
			em.close();
			em = null;
		}
	}
}
```

다음 테스트 메서드를 실행했을 경우  **`StackOverFlowError`** 가 발생합니다. 그 이유에 대해 기술하고, 해결책을 제시하세요.

- 힌트 : Entity의 **`@ToString`** 어노테이션, **`순환 참조`**

## 💡정답 및 해설

### ❓원인

- 순환 참조: `Team` 과 `Member` 는 서로 참조하고 있습니다.
    - `Team`의 `members` 필드와 `Member`의 `team` 필드
- 이 때 `@ToString` 어노테이션이 `Team`과 `Member` 엔티티 모두에 사용되면서,  `toString()` 호출이 끝없이 이어지게 됩니다.

### 🔧 해결 방법

- `@ToString` 어노테이션의 `exclude` 옵션 사용
    
    `Team.java`
    
    ```java
    // ...
    @ToString(exclude = "members") // members 필드를 toString 출력에서 제외
    @Entity
    public class Team {
    		// ...
    }
    ```
    
    `Member.java`
    
    ```java
    // ...
    @ToString(exclude = "team") // team 필드를 toString 출력에서 제외
    @Entity
    public class Member {
    		// ...
    }
    ```
    
    둘 중 하나의 엔티티에 exclude option을 추가합니다.
    

---

# 🥈문제 2

위의 `member.java` code에서 FetchType을

`@ManyToOne(fetch = FetchType.LAZY)` 로 실행했을 때와 

`@ManyToOne(fetch = FetchType.EAGER)` 으로 실행했을 때, 각 select문의 개수를 구하세요.

```java
public class RunTest {

  @Test
  public void test() {
      EntityManager em = DBUtil.getEntityManager();
      EntityTransaction tx = null;

      try {
          tx = em.getTransaction();
          tx.begin();

          em.createQuery("SELECT m FROM Member m", Member.class)
            .getResultList();

          tx.commit();

      } catch (Exception e) {
          if (tx != null && tx.isActive()) {
              tx.rollback();
          }
          e.printStackTrace();
      } finally {
          if (em != null) {
              em.close();
          }
      }
  }
}

```

## 💡정답

- `@ManyToOne(fetch = FetchType.EAGER)`  : 2회

```
Hibernate: select member4x0_.member_id as member_i1_0_, 
                member4x0_.name as name2_0_, 
                member4x0_.team_id as team_id3_0_ 
           from ce_member member4x0_
					 
Hibernate: select team4x0_.team_id as team_id1_1_0_, 
                team4x0_.team_name as team_nam2_1_0_ 
           from Team4 team4x0_ where team4x0_.team_id=?
```

- `@ManyToOne(fetch = FetchType.EAGER)`  : 1회

```
Hibernate: select member4x0_.member_id as member_i1_0_, 
              member4x0_.name as name2_0_, 
              member4x0_.team_id as team_id3_0_ 
           from ce_member member4x0_
```

## 📖 해설

`EAGER` 옵션은 연관된 엔티티를  모두 불러옵니다. 

따라서, 조회에 필요한 정보 외에 연관된 모든 데이터를 로드하기 위한 추가 쿼리가 발생합니다.

`LAZY` 옵션은 조회에 필요한 엔티티만 불러옵니다.

따라서, 기본 `SELECT` 쿼리만 실행되며 기타 엔티티에 대한 쿼리는 실행되지 않습니다.
