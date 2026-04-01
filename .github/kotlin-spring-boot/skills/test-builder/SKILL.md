---
name: test-builder
description: >-
  Use when creating unit tests, integration tests, or test builders following
  Clean Architecture with Kotlin. Activate for terms like: "create test",
  "write unit test", "integration test", "create test builder",
  "test fixture", "test data", "test use case", "test entity",
  "test repository", "Testcontainers", "PostgresIntegrationTest",
  "domain builder", or any task related to test coverage.
argument-hint: "[class or component to be tested]"
user-invocable: true
disable-model-invocation: false
---

# How to create Tests and Builders — Clean Architecture (Kotlin)

> Consult `/domain-language` for the project's aggregates and Value Objects
> when creating builders with realistic domain defaults.

## Source set separation

| Type | Location |
|---|---|
| Unit tests | `src/test/kotlin/` |
| Integration tests | `src/integrationTest/kotlin/` |
| Reusable builders | `src/testFixtures/kotlin/testbuilders/` |
| BDD scenarios | `src/integrationTest/resources/features/` |

Gradle `java-test-fixtures` plugin **required** in every module with builders.

**Minimum coverage:** `domain ≥ 90%`, `application ≥ 90%`, `infrastructure ≥ 90%`

---

## Template: domain entity builder

```kotlin
// src/testFixtures/kotlin/testbuilders/{Aggregate}Builder.kt
object {Aggregate}Builder {

    fun build(
        id: {Aggregate}Id = {Aggregate}Id.generate(),
        userId: UserId = UserId(UUID.randomUUID()),
        // other fields with realistic business defaults
        createdAt: Instant = Instant.now(),
        deletedAt: Instant? = null,
    ): {Aggregate} = {Aggregate}(
        id = id,
        userId = userId,
        createdAt = createdAt,
        deletedAt = deletedAt,
    )
}
```

## Template: Use Case Input builder

```kotlin
// src/testFixtures/kotlin/testbuilders/{Name}InputBuilder.kt
object {Name}InputBuilder {

    fun build(
        userId: UserId = UserId(UUID.randomUUID()),
        // other fields with defaults
    ) = {Name}Input(userId = userId /*, ... */)
}
```

---

## Template: unit test (Use Case)

```kotlin
class {Name}UseCaseTest {

    private val repository = mockk<{Aggregate}Repository>()
    private val useCase = {Name}UseCase(repository)

    @Test
    fun `should {behavior} when {condition}`() {
        // Arrange
        val input = {Name}InputBuilder.build()
        val entity = {Aggregate}Builder.build(userId = input.userId)
        every { repository.findById(any()) } returns entity
        every { repository.save(any()) } returns entity

        // Act
        val output = useCase.execute(input)

        // Assert
        assertThat(output.{field}).isEqualTo({expected})
        verify(exactly = 1) { repository.save(any()) }
    }

    @Test
    fun `should throw {Exception} when {error condition}`() {
        val input = {Name}InputBuilder.build()
        every { repository.findById(any()) } returns null

        assertThrows<{Exception}> { useCase.execute(input) }
        verify(exactly = 0) { repository.save(any()) }
    }
}
```

---

## Template: integration test base (Testcontainers)

```kotlin
// src/integrationTest/kotlin/PostgresIntegrationTest.kt
@SpringBootTest
@Testcontainers
abstract class PostgresIntegrationTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine").apply {
            withDatabaseName("test_db")
            withUsername("test")
            withPassword("test")
        }

        @JvmStatic
        @DynamicPropertySource
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
}
```

## Template: repository test

```kotlin
class {Aggregate}RepositoryTest : PostgresIntegrationTest() {

    @Autowired private lateinit var repository: {Aggregate}Repository

    @Test
    fun `should persist and retrieve by userId`() {
        val userId = UserId(UUID.randomUUID())
        val entity = {Aggregate}Builder.build(userId = userId)
        repository.save(entity)

        val found = repository.findAllByUserId(userId)

        assertThat(found).hasSize(1)
        assertThat(found.first().id).isEqualTo(entity.id)
    }

    @Test
    fun `should not return soft-deleted records`() {
        val userId = UserId(UUID.randomUUID())
        val entity = {Aggregate}Builder.build(userId = userId, deletedAt = Instant.now())
        repository.save(entity)

        val found = repository.findAllByUserId(userId)

        assertThat(found).isEmpty()
    }
}
```

> Enable `spring.jpa.show-sql=true` in integration tests to detect N+1 queries.

---

## Checklist

- [ ] Builder in `src/testFixtures/kotlin/testbuilders/{Aggregate}Builder.kt`
- [ ] `object` with named parameters and realistic domain defaults
- [ ] Unit tests in `src/test/kotlin/` — no I/O, no Spring
- [ ] Integration tests in `src/integrationTest/kotlin/` with Testcontainers
- [ ] `spring.jpa.show-sql=true` enabled in integration tests
- [ ] `java-test-fixtures` plugin enabled
- [ ] Coverage: `domain ≥ 90%`, `application ≥ 90%`, `infrastructure ≥ 90%`