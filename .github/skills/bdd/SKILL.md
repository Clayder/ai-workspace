---
name: bdd
description: >-
  Use ao criar cenários BDD com Cucumber seguindo Clean Architecture com Kotlin.
  Ative para termos como: "criar feature", "cenário BDD", "Cucumber", "Gherkin",
  "fluxo crítico", "teste end-to-end", "criar .feature", "escrever cenário",
  "dado que", "quando", "então", "step definition", ou qualquer tarefa que
  envolva especificação de comportamento em linguagem natural.
argument-hint: "[fluxo de negócio a especificar]"
user-invocable: true
disable-model-invocation: false
---

# Como criar cenários BDD com Cucumber — Clean Architecture (Kotlin)

> Consulte `/domain-language` para os fluxos críticos já identificados no projeto
> e os agregados envolvidos em cada cenário.

## Quando usar BDD

Apenas para **fluxos críticos end-to-end**. Módulos simples são cobertos por
testes de integração de camada `application` — sem `.feature` próprio.

---

## Localização e nomenclatura

```
src/integrationTest/resources/features/{agregado}/
{fluxo_de_negocio}.feature
```

**Um `.feature` por fluxo de negócio — nunca por endpoint individual.**

---

## Configuração Cucumber

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

## Template de `.feature` (PT-BR obrigatório)

```gherkin
# language: pt

Funcionalidade: {Nome do fluxo de negócio}
  Como {persona}
  Quero {ação}
  Para {objetivo de negócio}

  Contexto:
    Dado que estou autenticado como "{email}"
    E {pré-condição compartilhada entre cenários}

  Cenário: {Caminho feliz}
    Quando {ação do usuário}
    Então {resultado esperado no domínio}

  Cenário: {Caso de erro}
    Dado que {condição que causa o erro}
    Quando {ação do usuário}
    Então devo receber o erro "{mensagem}" com código "{PREFIXO_NNN}"
```

---

## Template de Step Definitions

```kotlin
// src/integrationTest/kotlin/stepdefinitions/{Agregado}Steps.kt
class {Agregado}Steps(
    @Autowired private val restTemplate: TestRestTemplate,
) {
    private lateinit var authToken: String
    private var lastResponse: ResponseEntity<*>? = null

    @Dado("que estou autenticado como {string}")
    fun queEstouAutenticadoComo(email: String) {
        val response = restTemplate.postForEntity(
            "/api/v1/auth/login",
            mapOf("email" to email, "password" to "senha-teste"),
            Map::class.java,
        )
        authToken = response.body?.get("token") as String
    }

    @Quando("{ação em PT-BR}")
    fun acao(/* parâmetros */) {
        val headers = HttpHeaders().apply { setBearerAuth(authToken) }
        lastResponse = restTemplate.exchange(
            "/api/v1/{endpoint}",
            HttpMethod.POST,
            HttpEntity(/* body */, headers),
            Map::class.java,
        )
    }

    @Então("devo receber o erro {string} com código {string}")
    fun devoReceberErro(mensagem: String, code: String) {
        assertThat(lastResponse?.statusCode?.value()).isIn(422, 403, 404, 400)
        val body = lastResponse?.body as Map<*, *>
        assertThat(body["code"]).isEqualTo(code)
    }
}
```

---

## Boas práticas

| Faça | Evite |
|---|---|
| Cenários independentes entre si | Cenários que dependem da ordem de execução |
| Dados em linguagem de negócio | Dados técnicos nos steps (IDs, centavos brutos) |
| `Contexto` para pré-condições compartilhadas | Repetir pré-condições em cada cenário |
| Um `.feature` por fluxo de negócio | Um `.feature` por endpoint da API |
| Cenário de caminho feliz + ao menos 1 de erro | Apenas cenários de sucesso |

---

## Checklist

- [ ] `.feature` em `src/integrationTest/resources/features/{agregado}/`
- [ ] `# language: pt` no topo do arquivo
- [ ] Cabeçalho `Funcionalidade` com persona e objetivo de negócio
- [ ] `Contexto` com pré-condições compartilhadas
- [ ] Steps em PT-BR com `Dado`, `Quando`, `Então`
- [ ] Dados em linguagem de negócio
- [ ] Caminho feliz + ao menos 1 cenário de erro
- [ ] Step definitions em `src/integrationTest/kotlin/stepdefinitions/`
- [ ] Testes rodando com Testcontainers