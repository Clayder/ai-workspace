---
name: test-builder
description: >-
  Use ao criar testes unitários, de integração ou builders de teste seguindo
  Clean Architecture com Kotlin. Ative para termos como: "criar teste",
  "escrever teste unitário", "teste de integração", "criar builder de teste",
  "test fixture", "massa de teste", "testar use case", "testar entidade",
  "testar repositório", "Testcontainers", "PostgresIntegrationTest",
  "builder de domínio", ou qualquer tarefa relacionada a cobertura de testes.
argument-hint: "[classe ou componente a ser testado]"
user-invocable: true
disable-model-invocation: false
---

# Como criar Testes e Builders — Clean Architecture (Kotlin)

> Consulte `/domain-language` para os agregados e Value Objects do projeto
> ao criar builders com defaults realistas do domínio.

## Separação de source sets

| Tipo | Localização |
|---|---|
| Testes de unidade | `src/test/kotlin/` |
| Testes de integração | `src/integrationTest/kotlin/` |
| Builders reutilizáveis | `src/testFixtures/kotlin/testbuilders/` |
| Cenários BDD | `src/integrationTest/resources/features/` |

Plugin Gradle `java-test-fixtures` **obrigatório** em todo módulo com builders.

**Cobertura mínima:** `domain ≥ 90%`, `application ≥ 90%`, `infrastructure ≥ 90%`

---

## Template: Builder de entidade de domínio

```kotlin
// src/testFixtures/kotlin/testbuilders/{Agregado}Builder.kt
object {Agregado}Builder {

    fun build(
        id: {Agregado}Id = {Agregado}Id.generate(),
        userId: UserId = UserId(UUID.randomUUID()),
        // demais campos com defaults realistas de negócio
        createdAt: Instant = Instant.now(),
        deletedAt: Instant? = null,
    ): {Agregado} = {Agregado}(
        id = id,
        userId = userId,
        createdAt = createdAt,
        deletedAt = deletedAt,
    )
}
```

## Template: Builder de Input de Use Case

```kotlin
// src/testFixtures/kotlin/testbuilders/{Nome}InputBuilder.kt
object {Nome}InputBuilder {

    fun build(
        userId: UserId = UserId(UUID.randomUUID()),
        // demais campos com defaults
    ) = {Nome}Input(userId = userId /*, ... */)
}
```

---

## Template: Teste de unidade (Use Case)

```kotlin
class {Nome}UseCaseTest {

    private val repository = mockk<{Agregado}Repository>()
    private val useCase = {Nome}UseCase(repository)

    @Test
    fun `deve {comportamento} quando {condição}`() {
        // Arrange
        val input = {Nome}InputBuilder.build()
        val entity = {Agregado}Builder.build(userId = input.userId)
        every { repository.findById(any()) } returns entity
        every { repository.save(any()) } returns entity

        // Act
        val output = useCase.execute(input)

        // Assert
        assertThat(output.{campo}).isEqualTo({esperado})
        verify(exactly = 1) { repository.save(any()) }
    }

    @Test
    fun `deve lançar {Excecao} quando {condição de erro}`() {
        val input = {Nome}InputBuilder.build()
        every { repository.findById(any()) } returns null

        assertThrows<{Excecao}> { useCase.execute(input) }
        verify(exactly = 0) { repository.save(any()) }
    }
}
```

---

## Template: Base de testes de integração (Testcontainers)

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

## Template: Teste de repositório

```kotlin
class {Agregado}RepositoryTest : PostgresIntegrationTest() {

    @Autowired private lateinit var repository: {Agregado}Repository

    @Test
    fun `deve persistir e recuperar por userId`() {
        val userId = UserId(UUID.randomUUID())
        val entity = {Agregado}Builder.build(userId = userId)
        repository.save(entity)

        val found = repository.findAllByUserId(userId)

        assertThat(found).hasSize(1)
        assertThat(found.first().id).isEqualTo(entity.id)
    }

    @Test
    fun `não deve retornar registros soft-deletados`() {
        val userId = UserId(UUID.randomUUID())
        val entity = {Agregado}Builder.build(userId = userId, deletedAt = Instant.now())
        repository.save(entity)

        val found = repository.findAllByUserId(userId)

        assertThat(found).isEmpty()
    }
}
```

> Ativar `spring.jpa.show-sql=true` nos testes de integração para detectar N+1.

---

## Checklist

- [ ] Builder em `src/testFixtures/kotlin/testbuilders/{Agregado}Builder.kt`
- [ ] `object` com parâmetros nomeados e defaults realistas do domínio
- [ ] Testes de unidade em `src/test/kotlin/` — sem I/O, sem Spring
- [ ] Testes de integração em `src/integrationTest/kotlin/` com Testcontainers
- [ ] `spring.jpa.show-sql=true` ativo nos testes de integração
- [ ] Plugin `java-test-fixtures` habilitado
- [ ] Cobertura: `domain ≥ 90%`, `application ≥ 90%`, `infrastructure ≥ 90%`