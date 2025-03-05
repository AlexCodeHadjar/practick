package com.example.kval

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.gestures.detectTapGestures
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.NavController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.room.*
import com.google.gson.Gson
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

@Entity data class User(
    @PrimaryKey val id: Int? =null,
    val  email : String,
    val password: String
)
@Entity data class Item(
    @PrimaryKey(autoGenerate = true) val primaryId: Int=0,
    val itemId: Int,
    val userId:Int,
    val title: String,
    val info : String,
    val dopInfo:String
)


data class JsonItem(val id: Int, val title: String, val info: String, val dopInfo:String)

@Dao interface AppDao {
    @Insert suspend fun insertUser(user: User): Long
    @Query("SELECT * FROM user WHERE email = :email AND password = :password") suspend fun getUser(email: String, password: String): User?
    @Insert suspend fun insertItem(item: Item)
    @Query("SELECT * FROM item WHERE userId = :userId") suspend fun getFavorites(userId: Int): List<Item>
}

@Database(entities = [User::class, Item::class], version = 3)
abstract class AppDatabase : RoomDatabase() {
    abstract fun appDao(): AppDao
}

class AppViewModel(private val dao: AppDao, private val context: android.content.Context) : ViewModel() {
    var user by mutableStateOf<User?>(null)
    var items by mutableStateOf<List<JsonItem>>(emptyList())
    var favorites by mutableStateOf<List<Item>>(emptyList())
    var error by mutableStateOf<String?>(null)

    init { loadItems() }

    private fun showTemporaryError(message: String) {
        error = message
        viewModelScope.launch { delay(3000); error = null }
    }

    suspend fun register(email: String, password: String) {
        try {
            if (email.isEmpty() || password.isEmpty()) throw Exception("Поля пустые")
            dao.insertUser(User(email = email, password = password))
            login(email, password)
        } catch (e: Exception) { showTemporaryError(e.message ?: "Ошибка") }
    }

    suspend fun login(email: String, password: String) {
        try {
            if (email.isEmpty() || password.isEmpty()) throw Exception("Поля пустые")
            user = dao.getUser(email, password)
            if (user == null) showTemporaryError("Неверно") else favorites = dao.getFavorites(user!!.id!!)
        } catch (e: Exception) { showTemporaryError(e.message ?: "Ошибка") }
    }

    private fun loadItems() {

            val json = context.assets.open("data.json").bufferedReader().use { it.readText() }
            items = Gson().fromJson(json, Array<JsonItem>::class.java).toList()

    }

    fun addToFavorite(item: JsonItem) {
        user?.let { u ->
            viewModelScope.launch {
                try {
                    dao.insertItem(Item(itemId = item.id, userId = u.id!!, title = item.title, info = item.info, dopInfo = item.info))
                    favorites = dao.getFavorites(u.id)
                } catch (e: Exception) { showTemporaryError("Ошибка") }
            }
        } ?: showTemporaryError("Войдите")
    }
}

@Composable
fun App(database: AppDatabase, context: android.content.Context) {
    val navController = rememberNavController()
    val vm: AppViewModel = viewModel { AppViewModel(database.appDao(), context) }
    MaterialTheme {
        NavHost(navController, "auth") {
            composable("auth") { AuthScreen(vm, navController) }
            composable("profile") { ProfileScreen(vm, navController) }
            composable("items") { ItemsScreen(vm, navController) }
            composable("favorites") { FavoritesScreen(vm) }
        }
    }
}

@Composable
fun AuthScreen(vm: AppViewModel, nav: NavController) {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    val scope = rememberCoroutineScope()
    Column {
        Text("Вход")
        TextField(value = email, onValueChange = { email = it }, label = { Text("Email") })
        TextField(value = password, onValueChange = { password = it }, label = { Text("Пароль") })
        Button(onClick = { scope.launch { vm.register(email, password); if (vm.user != null) nav.navigate("profile") } }) { Text("Регистрация") }
        Button(onClick = { scope.launch { vm.login(email, password); if (vm.user != null) nav.navigate("profile") } }) { Text("Войти") }

    }
}

@Composable
fun ProfileScreen(vm: AppViewModel, nav: NavController) {
    Column {
        Text("Профиль")
        Text(vm.user?.email ?: "Нет данных")
        Button(onClick = { nav.navigate("items") }) { Text("Список") }
        Button(onClick = { nav.navigate("favorites") }) { Text("Избранное") }
    }
}


@Composable
fun ItemsScreen(vm: AppViewModel, nav: NavController) {
    Column {
        Text("Список")
        LazyColumn {
            items(vm.items) { item ->
                Column(
                    Modifier.pointerInput(Unit) {
                        detectTapGestures(onDoubleTap = { vm.addToFavorite(item) })
                    }
                ) {
                    Text(item.title)
                    Text(item.info, color = Color.Gray)
                    Text(item.dopInfo)
                }
            }
        }

    }
}




@Composable
fun FavoritesScreen(vm: AppViewModel) {
    Column {
        Text("Избранное")
        LazyColumn {
            items(vm.favorites) { item -> Text(item.title) }

        }
    }
}

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val db = Room.databaseBuilder(this, AppDatabase::class.java, "app-db")
            .fallbackToDestructiveMigration()
            .build()
        setContent { App(db, this) }
    }
}













plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    id("kotlin-kapt")
}

android {
    namespace = "com.example.kval"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.kval"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    kotlinOptions {
        jvmTarget = "11"
    }
    buildFeatures {
        compose = true
    }
}

dependencies {
    implementation("com.google.code.gson:gson:2.8.8")
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.activity.compose)
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.ui)
    implementation(libs.androidx.ui.graphics)
    implementation(libs.androidx.ui.tooling.preview)
    implementation(libs.androidx.material3)
    implementation (libs.androidx.room.runtime)
    implementation (libs.androidx.room.ktx)
    kapt(libs.androidx.room.compiler)
    implementation (libs.gson)
    implementation (libs.androidx.navigation.compose)
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.ui.test.junit4)
    debugImplementation(libs.androidx.ui.tooling)
    debugImplementation(libs.androidx.ui.test.manifest)
}