# 17 — Android Networking & Storage

---

## Q1. Explain Retrofit.

**Answer:**

```kotlin
// API interface
interface ApiService {
    @GET("users")
    suspend fun getUsers(@Query("page") page: Int): Response<UserListResponse>

    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User

    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): User

    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: Int, @Body user: User): User

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: Int): Response<Unit>

    @Multipart
    @POST("upload")
    suspend fun uploadImage(@Part image: MultipartBody.Part): UploadResponse

    @Headers("Cache-Control: max-age=3600")
    @GET("config")
    suspend fun getConfig(): Config
}

// Setup with OkHttp
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenManager))
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    })
    .connectTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .cache(Cache(cacheDir, 10 * 1024 * 1024)) // 10MB cache
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/v1/")
    .client(okHttpClient)
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val api = retrofit.create(ApiService::class.java)
```

---

## Q2. How do you handle auth token refresh with OkHttp?

**Answer:**

```kotlin
class AuthInterceptor(private val tokenManager: TokenManager) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer ${tokenManager.accessToken}")
            .build()
        return chain.proceed(request)
    }
}

class TokenAuthenticator(
    private val tokenManager: TokenManager,
    private val authApi: AuthApi
) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        // Called when 401 received
        synchronized(this) {
            val newToken = runBlocking {
                authApi.refreshToken(tokenManager.refreshToken)
            }
            return if (newToken != null) {
                tokenManager.saveTokens(newToken)
                response.request.newBuilder()
                    .header("Authorization", "Bearer ${newToken.accessToken}")
                    .build()
            } else {
                null // give up, trigger logout
            }
        }
    }
}

val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenManager))
    .authenticator(TokenAuthenticator(tokenManager, authApi))
    .build()
```

---

## Q3. Explain Room relationships.

**Answer:**

```kotlin
// One-to-Many
@Entity
data class Author(@PrimaryKey val authorId: Long, val name: String)

@Entity(foreignKeys = [ForeignKey(entity = Author::class, parentColumns = ["authorId"],
    childColumns = ["bookAuthorId"], onDelete = ForeignKey.CASCADE)])
data class Book(@PrimaryKey val bookId: Long, val title: String, val bookAuthorId: Long)

data class AuthorWithBooks(
    @Embedded val author: Author,
    @Relation(parentColumn = "authorId", entityColumn = "bookAuthorId")
    val books: List<Book>
)

@Query("SELECT * FROM Author")
fun getAuthorsWithBooks(): Flow<List<AuthorWithBooks>>

// Many-to-Many (junction table)
@Entity(primaryKeys = ["playlistId", "songId"])
data class PlaylistSongCrossRef(val playlistId: Long, val songId: Long)

data class PlaylistWithSongs(
    @Embedded val playlist: Playlist,
    @Relation(
        parentColumn = "playlistId",
        entityColumn = "songId",
        associateBy = Junction(PlaylistSongCrossRef::class)
    )
    val songs: List<Song>
)
```

---

## Q4. How do you implement caching with OkHttp?

**Answer:**

```kotlin
// HTTP cache (respects Cache-Control headers)
val cache = Cache(File(context.cacheDir, "http_cache"), 50L * 1024 * 1024) // 50MB

val client = OkHttpClient.Builder()
    .cache(cache)
    .addInterceptor(CacheInterceptor())
    .addNetworkInterceptor(NetworkCacheInterceptor())
    .build()

// Force cache when offline
class CacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()
        if (!isNetworkAvailable()) {
            request = request.newBuilder()
                .cacheControl(CacheControl.FORCE_CACHE)
                .build()
        }
        return chain.proceed(request)
    }
}

// Set cache duration for network responses
class NetworkCacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        val cacheControl = CacheControl.Builder()
            .maxAge(5, TimeUnit.MINUTES)
            .build()
        return response.newBuilder()
            .header("Cache-Control", cacheControl.toString())
            .build()
    }
}
```

---

## Q5. Explain EncryptedSharedPreferences.

**Answer:**

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val securePrefs = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

// Use like regular SharedPreferences
securePrefs.edit()
    .putString("auth_token", token)
    .putString("refresh_token", refreshToken)
    .apply()

val token = securePrefs.getString("auth_token", null)
```
