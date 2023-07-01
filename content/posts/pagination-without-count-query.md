---
title: "Pagination without count query in Spring Data JPA"
date: 2022-02-19T14:44:55+03:00
draft: false
---

Hi, in this post I’m going to show you how to implement pagination without count query. By default, Spring Data JPA doesn’t provide pagination method without count query when one want to use Specification<T> interface in methods. Yes, there are methods that return Slice<T>, but there isn’t any method that returns Slice<T> when we want to use Specification<T>. In this post, we’re going to implement our own Repository<T>.

If we take a look at SimpleJpaRepository’s readPage method:

```java
protected <S extends T> Page<S> readPage(TypedQuery<S> query, final Class<S> domainClass, Pageable pageable, @Nullable Specification<S> spec) {
    if (pageable.isPaged()) {
        query.setFirstResult((int)pageable.getOffset());
        query.setMaxResults(pageable.getPageSize());
    }

    return PageableExecutionUtils.getPage(query.getResultList(), pageable, () -> {
        return executeCountQuery(this.getCountQuery(spec, domainClass));
    });
}
```

At the end of the method, method executes the count query, and this can be costly for large datasets.
Slice interface holds only previous and next page information. There is no total page count information in Slice as it should be.↳

First we need to create our own interface for custom repository:

```java
@NoRepositoryBean
public interface ExtendedRepository<T, ID extends Serializable> extends JpaRepositoryImplementation<T, ID> {
    Slice<T> findAllAsSlice(Specification<T> specification, Pageable pageable);
}
```

We marked the interface as NoRepositoryBean which means that Spring Data JPA won’t try to provide the implementation of this interface.

After declaring the interface, we can implement the interface:

```java
public class SimpleExtendedRepository<T, ID extends Serializable> extends SimpleJpaRepository<T, ID> implements ExtendedRepository<T, ID> {

    public SimpleExtendedRepository(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {
        super(entityInformation, entityManager);
    }

    @Override
    public Slice<T> findAllAsSlice(Specification<T> specification, Pageable pageable) {
        TypedQuery<T> query = getQuery(specification, pageable);
        return pageable.isUnpaged() ? new SliceImpl<>(query.getResultList()) : readSlice(query, pageable, specification);
    }

    private Slice<T> readSlice(TypedQuery<T> query, Pageable pageable, Specification<T> specification){
        if (pageable.isPaged()){
            query.setFirstResult((int) pageable.getOffset());
            query.setMaxResults(pageable.getPageSize() + 1); // We should get 1 more row to understand there is a next page or not
        }
        List<T> content = query.getResultList();
        boolean hasNextPage = content.size() > pageable.getPageSize();
        if (content.size() > pageable.getPageSize()){ // If the result set contains 1 more row than the desired page count, we normalize the result set
            content = content.subList(0, pageable.getPageSize());
        }
        return new SliceImpl<>(content, pageable, hasNextPage);
    }
}
```

After concrete implementation, we need to tell the Spring Data JPA that it should use SimpleExtendedRepository class when creating implementations of Repository interfaces. To do that just simply add the repositoryBaseClass property to the @EnableJpaRepositories annotation:

```java
@SpringBootApplication
@EnableJpaRepositories(repositoryBaseClass = SimpleExtendedRepository.class)
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

And that’s it, now you can start to use our custom method by implementing ExtendedRepository interface like following:

```java
@Repository
public interface EmployeeRepository extends ExtendedRepository<Employee, Integer> {

}
```

This is the end of the post, hope you enjoyed! Please feel free to contact me if you have any suggestions, questions, concerns.