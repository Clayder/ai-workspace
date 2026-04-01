---
name: bdd
description: >-
  Use when creating BDD scenarios with Cucumber following Clean Architecture with Kotlin.
  Activate for terms like: "create feature", "BDD scenario", "Cucumber", "Gherkin",
  "critical flow", "end-to-end test", "create .feature", "write scenario",
  "given", "when", "then", "step definition", or any task involving
  behavior specification in natural language.
argument-hint: "[business flow to specify]"
user-invocable: true
disable-model-invocation: false
---

# How to create BDD scenarios with Cucumber — Clean Architecture (Kotlin)

> Consult `/domain-language` for the critical flows already identified in the project
> and the aggregates involved in each scenario.

## When to use BDD

Only for **critical end-to-end flows**. Simple modules are covered by
`application`-layer integration tests — no dedicated `.feature` file needed.

---

## Location and naming

```
src/integrationTest/resources/features/{aggregate}/
{business_flow}.feature
```

**One `.feature` per business flow — never per individual endpoint.**

---

## Cucumber configuration

```kotlin
// src/integrationTest/kotlin/CucumberTestRunner.kt
@Suite
@IncludeEngines("cucumber")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "stepdefinitions")
@ConfigurationParameter(key = FEATURES_PROPERTY_NAME, value = "src/integrationTest/resources/features")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, value = "pretty, html:build/reports/cucumber.html")
class CucumberTestRunner

// src/integrationTest/kotlin/CucumberSpringConfiguration.kt
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class CucumberSpringConfiguration : PostgresIntegrationTest()
```

---

## `.feature` template (Brazilian Portuguese — mandatory)

Feature files **must** be written in Brazilian Portuguese. This is a project convention.

```gherkin
# language: pt

Funcionalidade: {Business flow name}
  Como {persona}
  Quero {action}
  Para {business objective}

  Contexto:
    Dado que estou autenticado como "{email}"
    E {shared precondition between scenarios}

  Cenário: {Happy path}
    Quando {user action}
    Então {expected domain result}

  Cenário: {Error case}
    Dado que {condition that causes the error}
    Quando {user action}
    Então devo receber o erro "{message}" com código "{PREFIX_NNN}"
```

---

## Step Definitions template

```kotlin
// src/integrationTest/kotlin/stepdefinitions/{Aggregate}Steps.kt
class {Aggregate}Steps(
    @Autowired private val restTemplate: TestRestTemplate,
) {
    private lateinit var authToken: String
    private var lastResponse: ResponseEntity<*>? = null

    @Dado("que estou autenticado como {string}")
    fun authenticatedAs(email: String) {
        val response = restTemplate.postForEntity(
            "/api/v1/auth/login",
            mapOf("email" to email, "password" to "test-password"),
            Map::class.java,
        )
        authToken = response.body?.get("token") as String
    }

    @Quando("{ação em PT-BR}")
    fun action(/* parameters */) {
        val headers = HttpHeaders().apply { setBearerAuth(authToken) }
        lastResponse = restTemplate.exchange(
            "/api/v1/{endpoint}",
            HttpMethod.POST,
            HttpEntity(/* body */, headers),
            Map::class.java,
        )
    }

    @Então("devo receber o erro {string} com código {string}")
    fun shouldReceiveError(message: String, code: String) {
        assertThat(lastResponse?.statusCode?.value()).isIn(422, 403, 404, 400)
        val body = lastResponse?.body as Map<*, *>
        assertThat(body["code"]).isEqualTo(code)
    }
}
```

---

## Best practices

| Do | Avoid |
|---|---|
| Independent scenarios | Scenarios that depend on execution order |
| Business-language data | Technical data in steps (raw IDs, cents) |
| `Contexto` for shared preconditions | Repeating preconditions in each scenario |
| One `.feature` per business flow | One `.feature` per API endpoint |
| Happy path + at least 1 error scenario | Success-only scenarios |

---

## Checklist

- [ ] `.feature` in `src/integrationTest/resources/features/{aggregate}/`
- [ ] `# language: pt` at the top of the file
- [ ] `Funcionalidade` header with persona and business objective
- [ ] `Contexto` with shared preconditions
- [ ] Steps in PT-BR with `Dado`, `Quando`, `Então`
- [ ] Business-language data
- [ ] Happy path + at least 1 error scenario
- [ ] Step definitions in `src/integrationTest/kotlin/stepdefinitions/`
- [ ] Tests running with Testcontainers