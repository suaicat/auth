### Авторизация ГУАП

Примерный алгоритм реализации кастомной авторизации в системе личного кабинета ГУАП. Представлю на Kotlin, но при желании можно адаптировать на любой язык.


Установка библиотек


```kotlin
%useLatestDescriptors
%use ktor-client
%use serialization
```

Модель для получения токена


```kotlin
@Serializable
data class TokenResponse(
    @SerialName("access_token")  val accessToken:  String,
    @SerialName("id_token")      val idToken:      String,
    @SerialName("refresh_token") val refreshToken: String,
    @SerialName("expires_in")    val expiresIn:    Int,
    @SerialName("refresh_expires_in") val refreshExpiresIn: Int,
    @SerialName("token_type")    val tokenType:    String
)
```

Объявляем основные переменные


```kotlin
val CLIENT_ID = "prosuaiApi"
val URL = "https://sso.guap.ru:8443/realms/master/protocol/openid-connect/"
val login = System.getenv("login")!!
val password = System.getenv("password")!!

```

#### Простая реализация

Инициализация клиента


```kotlin
import io.ktor.client.HttpClient
import io.ktor.client.plugins.contentnegotiation.ContentNegotiation
import io.ktor.client.plugins.defaultRequest
import io.ktor.client.plugins.logging.LogLevel
import io.ktor.client.plugins.logging.Logging
import io.ktor.serialization.kotlinx.json.json

val client = HttpClient {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true }) // Устанавливаем десериализацию из JSON в объект
    }

    defaultRequest {
        url(URL) // Устанавливаем ссылку по умолчанию
    }
}
```


```kotlin
import io.ktor.client.call.body
import io.ktor.client.request.forms.submitForm
import io.ktor.client.statement.bodyAsText
import io.ktor.client.statement.readBytes
import io.ktor.http.Parameters
import kotlinx.coroutines.runBlocking

suspend fun HttpClient.getToken(login: String, password: String): TokenResponse {
    return client.submitForm(
        "token",
        formParameters = Parameters.build {
            append("client_id", CLIENT_ID)
            append("grant_type", "password")
            append("username", login)
            append("password", password)
            append("scope", "openid email profile roles")
        }
    ).body()
}

val token = runBlocking{client.getToken(login, password)}
// token // Ваш токен будет виден всем если вы поделитесь notebook
```

Получим данные пользователя


```kotlin
import io.ktor.client.request.get
import io.ktor.client.request.headers
import io.ktor.http.HeaderValue
import io.ktor.http.Headers
import io.ktor.http.HttpHeaders
import io.ktor.http.parameters

val user = runBlocking {
    val resp = client.get("userinfo"){
        headers{
            append(HttpHeaders.Authorization, """Bearer ${token.accessToken}""")
        }
    }
    println(resp.status.value)
    return@runBlocking resp.bodyAsText()
}
// user

```

    200

