# 1. はじめに
&nbsp;&nbsp;&nbsp; Androidアプリで、APIを連携してデータを取ってくる一連の流れがよく忘れてしまうので、書いてみました。<br>
&nbsp;&nbsp;&nbsp; 基本、変数や関数、ファイル名にある「Books」を適宜必要な名前に変更して頂けると使えるようになっています。

# 2. 使用するAPIについて
　本記事では、APIとして[Google Books API](https://developers.google.com/books/docs/v1/reference/volumes?hl=ja)を採用しています。

# 3. 実装方法
## 3-1. アーキテクチャ
```
<UI Layer>
 UI elements
 　↓
 State holders（ViewModelが該当）
　 ↓
<Data Layer>
 Repositories（network packageが該当）
 　↓
 Data sources（data packageと、model packageが該当）
```
## 3-2. プロジェクト構造
```
com.example.bookshelf
├── data
      ├── AppContainer.kt
      └── BooksRepository.kt　　　
├── model
      └── Books.kt
├── network
      └── ApiService.kt
└── ui
      └── screen
          ├── HomeScreen.kt（本記事では省略）
          └── BooksViewModel.kt
└── BooksApplication.kt
```


## 3-3. model package
&nbsp;&nbsp;&nbsp; このパッケージでは、アプリ側が受け取るデータクラスを実装します。変数名は、 JSON オブジェクトのキー名に対応する必要があります。
```Books.kt
import kotlinx.serialization.Serializable

@Serializable
data class BookList (
 val items: List<Book>
)

@Serializable
data class Book (
 val id: String,
 val volumeInfo: VolumeInfo
)

@Serializable
data class VolumeInfo (
 val title: String,
 val authors: List<String>,
 val imageLinks: ImageLinks
)

@Serializable
data class ImageLinks (
 val thumbnail: String
)
```


## 3-5. network package
&nbsp;&nbsp;&nbsp; このpackageでは、ApiServiceを作成します。ApiServiceは、RetrofitがHTTPリクエストを使用してサーバーと通信する方法を定義します。
　まず、必要なライブラリをプロジェクトに追加します。
```build.gradle.kts
// Retrofit 
implementation("com.squareup.retrofit2:retrofit:2.9.0")
```
　次に、ApiServiceを作成します。
```ApiService.kt
import com.example.boicebookshelf.model.BookList
import retrofit2.http.GET
import retrofit2.http.Path

interface BooksApiService {
 @GET("/books/v1/volumes?q={search terms}")
 suspend fun getBooks(
    @Path("search terms") terms: String
 ): BookList
}
```


## 3-4. data package
&nbsp;&nbsp;&nbsp; このpackageでは、repositoryとAppContainerを作成します。Repositoryは、実際にネットワークの呼び出しを行います。
```BooksRepository.kt
import com.example.boicebookshelf.model.BookList
import com.example.boicebookshelf.network.BooksApiService

interface BooksRepository {
    suspend fun getBooks(terms: String): BookList
}

class NetworkBooksRepository(
    private val booksApiService: BooksApiService
) : BooksRepository {
    override suspend fun getBooks(terms: String): BookList = booksApiService.getBooks(terms)
}
```
&nbsp;&nbsp;&nbsp; AppContainerは、実際にネットワークの呼び出しを行うリポジトリをアプリ全体に公開するアプリケーションコンテナです。
　まず、必要なライブラリをプロジェクトに追加します。
```build.gradle.kts(Module;app)
plugins {
  id("org.jetbrains.kotlin.plugin.serialization") version "1.8.10"
}
```
```build.gradle.kts
// Kotlin serialization 
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.5.1")

// Retrofit with Kotlin serialization Converter
implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
implementation("com.squareup.okhttp3:okhttp:4.11.0")
```
&nbsp;&nbsp;&nbsp; 次に、インターネットにアクセスするためのINTERNET権限を、マニフェストで宣言します。以下の行は、applicationタグの直前に追加します。
```manifests/AndroidManifest.xml
<uses-permission android:name="android.permission.INTERNET" />
```
&nbsp;&nbsp;&nbsp; 最後に、アプリケーションコンテナを作成します。
```AppContainer.kt
import com.example.boicebookshelf.network.BooksApiService
import retrofit2.Retrofit
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType

interface AppContainer {
    val booksRepository: BooksRepository
}

class DefaultAppContainer: AppContainer {
    // 想定外のキーを無視
    private val format = Json { ignoreUnknownKeys = true }

    private val baseUrl = "https://www.googleapis.com"

    // JSONレスポンスをKotlinオブジェクトに変換
    private val retrofit: Retrofit = Retrofit.Builder()
        .addConverterFactory(format.asConverterFactory("application/json".toMediaType()))
        .baseUrl(baseUrl)
        .build()

    // リフレクション
    private val retrofitService: BooksApiService by lazy {
        retrofit.create(BooksApiService::class.java)
    }

    override val booksRepository: BooksRepository by lazy {
        NetworkBooksRepository(retrofitService)
    }
}
```


## 3-6. コンテナをアプリにアタッチ
&nbsp;&nbsp;&nbsp; BooksApplication.ktをcom.example.boicebookshelfの直下に置きます。
```BooksApplication.kt
import android.app.Application
import com.example.boicebookshelf.data.AppContainer
import com.example.boicebookshelf.data.DefaultAppContainer

class BooksApplication: Application() {
    lateinit var container: AppContainer
    override fun onCreate() {
        super.onCreate()
        container = DefaultAppContainer()
    }
}
```
&nbsp;&nbsp;&nbsp;  AndroidManifestのapplicationセクションに、`android:name=".BooksApplication"`を追加します。
```xml:AndroidManifest.xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />
    <application
        android:name=".BooksApplication"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.BoiceBookShelf"
        tools:targetApi="31">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.BoiceBookShelf">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```


## 3-7. ViewModelにリポジトリを追加
```BooksViewModel.kt
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
import android.net.http.HttpException
import android.os.Build
import androidx.annotation.RequiresExtension
import androidx.compose.runtime.mutableStateOf
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.ViewModelProvider.AndroidViewModelFactory.Companion.APPLICATION_KEY
import androidx.lifecycle.viewModelScope
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import com.example.boicebookshelf.BooksApplication
import com.example.boicebookshelf.data.BooksRepository
import com.example.boicebookshelf.model.Book
import kotlinx.coroutines.launch
import java.io.IOException

sealed interface BooksUiState {
    data class Success(val books: List<Book>) : BooksUiState
    object Error : BooksUiState
    object Loading : BooksUiState
}

@RequiresExtension(extension = Build.VERSION_CODES.S, version = 7)
class BooksViewModel(private val booksRepository: BooksRepository): ViewModel() {
    var booksUiState: BooksUiState by mutableStateOf(BooksUiState.Loading)
        private set

    init {
        getBooks("")
    }

    @RequiresExtension(extension = Build.VERSION_CODES.S, version = 7)
    fun getBooks(terms: String) {
        viewModelScope.launch {
            booksUiState = BooksUiState.Loading
            booksUiState = try {
                val books = booksRepository.getBooks(terms)
                BooksUiState.Success(books.items)
            } catch (e: IOException) {
                BooksUiState.Error
            } catch (e: HttpException) {
                BooksUiState.Error
            }
        }
    }

    companion object {
        val Factory: ViewModelProvider.Factory = viewModelFactory {
            initializer {
                val application = (this[APPLICATION_KEY] as BooksApplication)
                val booksRepository = application.container.booksRepository
                BooksViewModel(booksRepository)
            }
        }
    }
}
```
&nbsp;&nbsp;&nbsp; sealed interfaceでは、enumクラスとは違い、複数のインスタンスを持つことができます。一方で、sealed interfaceの継承によって、型の種類を制限できるため、使い勝手が良いです。
参照元）https://karino2.github.io/kotlin-web-site-ja/docs/sealed-classes.html
:::


# 4. おわりに
&nbsp;&nbsp;&nbsp; ここまで、アーキテクチャからパッケージ構造、各パッケージ内でのコードを説明してきました。UIの説明は省き、APIからデータを取得する最低限のコードを書きました。次回は、APIを介して、データをサーバーに保存する方法を書きたいと思います。

# 5. 参考URL

https://developer.android.com/codelabs/basic-android-kotlin-compose-getting-data-internet?hl=ja#6

https://developer.android.com/codelabs/basic-android-kotlin-compose-add-repository?hl=ja#3

