# kval


__________________________________________________________________________________________________________________________________________________________
BuildGRANDLE
__________________________________________________________________________________________________________________________________________________________
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
}

android {
    namespace = "com.android.room"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.android.room"
        minSdk = 29
        targetSdk = 34
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
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
    }
    buildFeatures {
        viewBinding = true
    }
}

dependencies {

    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.11.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")
    implementation("androidx.compose.ui:ui-graphics-android:1.6.2")
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")

    val roomVersion = "2.6.1"
    implementation("androidx.room:room-ktx:$roomVersion")
    kapt("androidx.room:room-compiler:$roomVersion")
}

__________________________________________________________________________________________________________________________________________________________
DataBase
__________________________________________________________________________________________________________________________________________________________

class DataBase {
    @Database(entities = [Entity.User::class,Entity.Fav::class, Entity.Record::class], version = 1)
    abstract class AppDatabase: RoomDatabase() {
        abstract fun userDao(): DAO.UserDao
        abstract fun favDao(): DAO.FavDao
        abstract fun recordDao(): DAO.RecordDao
    }
}
interface DAO {
    @Dao
    interface UserDao{
        @Insert
        fun insertUser(user: Entity.User)
        @Query("SELECT * from User where login = :login and password =:password")
        fun checkUser(login: String, password: String): Entity.User?
        @Query("SELECT * from User where login = :login")
        fun checkLogin(login: String): Entity.User?
        @Query("SELECT * from User where idUser = :id")
        fun getUser(id: Int): Entity.User
    }
    @Dao
    interface FavDao{
        @Insert
        fun insertFav(fav: Entity.Fav)
        @Delete
        fun deleteFav(fav: Entity.Fav)
        @Query("SELECT * FROM Record WHERE idRecord IN (SELECT idRecord FROM Fav WHERE idUser = :idUser)")
        fun getFavRecords(idUser: Int): List<Entity.Record>?
        @Query("SELECT * from Fav where idRecord = :idRecord and idUser =:idUser")
        fun checkFav(idRecord: Int, idUser: Int): Entity.Fav?
    }
    @Dao
    interface RecordDao{
        @Query("SELECT * FROM Record")
        fun getAllRecords(): List<Entity.Record>
    }
}
class Entity {
    @androidx.room.Entity
    data class User(
        @PrimaryKey(autoGenerate = true)
        val idUser: Int = 0,
        val login: String,
        val password: String,
    )
    @androidx.room.Entity
    data class Fav(
        @PrimaryKey(autoGenerate = true)
        val idFav: Int = 0,
        val idUser: Int,
        val idRecord: Int,
    )
    @androidx.room.Entity
    data class Record(
        @PrimaryKey(autoGenerate = true)
        val idRecord: Int = 0,
        val startCity: String,
        val startCityCode: String,
        val endCity : String,
        val endCityCode: String,
        val startDate: String,
        val endDate: String,
        val price: Int,
        val searchToken: String
    )
}

__________________________________________________________________________________________________________________________________________________________
AutActivity
__________________________________________________________________________________________________________________________________________________________

class AutActivity : AppCompatActivity() {
    private lateinit var binding: ActivityAutBinding
    private lateinit var userDatabase: DataBase.AppDatabase
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityAutBinding.inflate(layoutInflater)
        setContentView(binding.root)

        userDatabase = Room.databaseBuilder(
            applicationContext,
            DataBase.AppDatabase::class.java,
            "user_database"
        ).build()

        binding.btAction.setOnClickListener {
            if (binding.etLogin.text.isEmpty() || binding.etPassword.text.isEmpty()){
                Toast.makeText(this, "Введены не все данные", Toast.LENGTH_SHORT).show()
            }
            else{
                CoroutineScope(Dispatchers.IO).launch {
                    val result = userDatabase.userDao().checkUser(binding.etLogin.text.toString(), binding.etPassword.text.toString())
                    withContext(Dispatchers.Main) {
                        if (result != null){
                            val intent = Intent(this@AutActivity, MainActivity::class.java)
                            intent.putExtra("id", result.idUser)
                            startActivity(intent)
                            finish()
                        }
                        else {
                            Toast.makeText(this@AutActivity, "Пользователя не существует", Toast.LENGTH_SHORT).show()
                        }
                    }
                }
            }
        }

        binding.btBack.setOnClickListener {
            val intent = Intent(this, RegActivity::class.java)
            startActivity(intent)
            finish()
        }

    }
}
__________________________________________________________________________________________________________________________________________________________
FavActivity
__________________________________________________________________________________________________________________________________________________________

class FavActivity : AppCompatActivity() {
    private lateinit var binding: ActivityFavBinding
    private lateinit var userDatabase: DataBase.AppDatabase
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityFavBinding.inflate(layoutInflater)
        setContentView(binding.root)
        userDatabase = Room.databaseBuilder(
            applicationContext,
            DataBase.AppDatabase::class.java,
            "user_database"
        ).build()
        val id = intent.getIntExtra("id", 0)

        CoroutineScope(Dispatchers.IO).launch {
            withContext(Dispatchers.IO) {

                val recyclerView: RecyclerView = findViewById(R.id.recyclerViewFav)
                recyclerView.layoutManager = LinearLayoutManager(this@FavActivity)

                val myFavList : List<Entity.Record> = userDatabase.favDao().getFavRecords(id)!!
                val adapter = AdapterFav(myFavList, id, userDatabase)

                recyclerView.adapter = adapter

            }
        }
        binding.btBackMain.setOnClickListener {
            val intent = Intent(this@FavActivity, MainActivity::class.java)
            intent.putExtra("id", id)
            startActivity(intent)
            finish()
        }
    }
}

__________________________________________________________________________________________________________________________________________________________
LentaActivity 
__________________________________________________________________________________________________________________________________________________________

class LentaActivity : AppCompatActivity() {

    private lateinit var binding: ActivityLentaBinding
    private lateinit var userDatabase: DataBase.AppDatabase

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityLentaBinding.inflate(layoutInflater)
        setContentView(binding.root)

        userDatabase = Room.databaseBuilder(
            applicationContext,
            DataBase.AppDatabase::class.java,
            "user_database"
        ).build()
        val id = intent.getIntExtra("id", 0)

        CoroutineScope(Dispatchers.IO).launch {

            val recyclerView : RecyclerView = findViewById(R.id.recyclerView)
            recyclerView.layoutManager = LinearLayoutManager(this@LentaActivity)

            val myDataList: List<Entity.Record> = userDatabase.recordDao().getAllRecords()
            val adapter = Adapter(myDataList, id, userDatabase)
            recyclerView.adapter = adapter
        }

        binding.btBackMain.setOnClickListener {
            val intent = Intent(this, MainActivity::class.java)
            intent.putExtra("id", id)
            startActivity(intent)
            finish()
        }
    }
}


__________________________________________________________________________________________________________________________________________________________
MainActivity 
__________________________________________________________________________________________________________________________________________________________

class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private lateinit var userDatabase: DataBase.AppDatabase

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding =ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        //id пользователя, который авторизовался
        val id = intent.getIntExtra("id",0)

        //инициализация бд
        userDatabase = Room.databaseBuilder(
            applicationContext,
            DataBase.AppDatabase::class.java,
            "user_database"
        ).build()
        CoroutineScope(Dispatchers.IO).launch {
            val user = userDatabase.userDao().getUser(id)

            binding.etUserName.text = "Имя пользователя: ${user.login}"
        }

        binding.btExit.setOnClickListener {
            val intent = Intent(this, AutActivity::class.java)
            startActivity(intent)
            finish()
        }

        binding.btLenta.setOnClickListener {
            val intent = Intent(this, LentaActivity::class.java)
            intent.putExtra("id", id)
            startActivity(intent)
            finish()
        }

        binding.btFav.setOnClickListener {
            val intent = Intent(this, FavActivity::class.java)
            intent.putExtra("id", id)
            startActivity(intent)
            finish()
        }
    }
}

__________________________________________________________________________________________________________________________________________________________
RegActivity
__________________________________________________________________________________________________________________________________________________________

class RegActivity : AppCompatActivity() {
    private lateinit var binding: ActivityRegBinding
    private lateinit var userDatabase: DataBase.AppDatabase
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityRegBinding.inflate(layoutInflater)
        setContentView(binding.root)

        userDatabase = Room.databaseBuilder(
            applicationContext,
            DataBase.AppDatabase::class.java, "user_database"
        ).build()

        binding.btAction.setOnClickListener {
            if (binding.etLogin.text.isEmpty() || binding.etPassword.text.isEmpty()){
                Toast.makeText(this, "Все поля должны быть заполнены!", Toast.LENGTH_SHORT).show()
            }
            else{
                CoroutineScope(Dispatchers.IO).launch {
                    val userExist = userDatabase.userDao().checkLogin(binding.etLogin.text.toString())
                    if (userExist != null){
                        Toast.makeText(this@RegActivity, "Логин не доступен", Toast.LENGTH_SHORT).show()
                    }
                    else{
                        val user = Entity.User(
                            login = binding.etLogin.text.toString(),
                            password = binding.etPassword.text.toString(),
                        )
                        userDatabase.userDao().insertUser(user)
                        withContext(Dispatchers.Main){
                            Toast.makeText(this@RegActivity, "Запись добавлена", Toast.LENGTH_SHORT).show()
                        }
                    }
                }
            }
        }
        binding.btBack.setOnClickListener {
            val intent = Intent(this, AutActivity::class.java)
            startActivity(intent)
            finish()
        }

    }
}


__________________________________________________________________________________________________________________________________________________________
activity_aut
__________________________________________________________________________________________________________________________________________________________

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".AutActivity">

    <EditText
        android:id="@+id/etLogin"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:ems="10"
        android:hint="Login"
        android:inputType="text"
        android:minHeight="48dp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>

    <EditText
        android:id="@+id/etPassword"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:ems="10"
        android:hint="Password"
        android:inputType="text"
        android:minHeight="48dp"
        app:layout_constraintEnd_toEndOf="@+id/etLogin"
        app:layout_constraintStart_toStartOf="@+id/etLogin"
        app:layout_constraintTop_toBottomOf="@+id/etLogin"/>

    <Button
        android:id="@+id/btBack"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:text="To registrartion"
        android:background="@android:color/transparent"
        android:textColor="@android:color/black"
        app:layout_constraintEnd_toEndOf="@+id/btAction"
        app:layout_constraintStart_toStartOf="@+id/btAction"
        app:layout_constraintTop_toBottomOf="@+id/btAction" />

    <Button
        android:id="@+id/btAction"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:text="Login"
        app:layout_constraintEnd_toEndOf="@+id/etPassword"
        app:layout_constraintStart_toStartOf="@+id/etPassword"
        app:layout_constraintTop_toBottomOf="@+id/etPassword" />

</androidx.constraintlayout.widget.ConstraintLayout>


__________________________________________________________________________________________________________________________________________________________
activity_fav
__________________________________________________________________________________________________________________________________________________________

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".FavActivity">
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewFav"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="8dp"
        app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
        tools:listitem="@layout/item_layout">
    </androidx.recyclerview.widget.RecyclerView>

    <Button
        android:id="@+id/btBackMain"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Back"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>


__________________________________________________________________________________________________________________________________________________________
activity_lenta
__________________________________________________________________________________________________________________________________________________________

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".LentaActivity">
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="8dp"
        app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
        tools:listitem="@layout/item_layout">
    </androidx.recyclerview.widget.RecyclerView>

    <Button
        android:id="@+id/btBackMain"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Back"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="1.0" />

</androidx.constraintlayout.widget.ConstraintLayout>

__________________________________________________________________________________________________________________________________________________________
activity_main
__________________________________________________________________________________________________________________________________________________________


<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/etUserName"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/etUserSurname"
        app:layout_constraintVertical_bias="0.068" />

    <Button
        android:id="@+id/btLenta"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Lenta"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toStartOf="@+id/btExit"
        app:layout_constraintHorizontal_bias="0.575"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.443" />

    <Button
        android:id="@+id/btFav"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Fav"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.432"
        app:layout_constraintStart_toEndOf="@+id/btExit"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.443" />

    <Button
        android:id="@+id/btExit"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Exit"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toStartOf="@+id/btFav"
        app:layout_constraintStart_toEndOf="@+id/btLenta"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.443" />

    <TextView
        android:id="@+id/etUserSurname"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="156dp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>


__________________________________________________________________________________________________________________________________________________________
activity_reg
__________________________________________________________________________________________________________________________________________________________

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".RegActivity">


    <EditText
        android:id="@+id/etLogin"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:ems="10"
        android:hint="Login"
        android:inputType="text"
        android:minHeight="48dp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>

    <EditText
        android:id="@+id/etPassword"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:ems="10"
        android:hint="Password"
        android:inputType="text"
        android:minHeight="48dp"
        app:layout_constraintEnd_toEndOf="@+id/etLogin"
        app:layout_constraintStart_toStartOf="@+id/etLogin"
        app:layout_constraintTop_toBottomOf="@+id/etLogin" />


    <Button
        android:id="@+id/btAction"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:text="Registration"
        app:layout_constraintEnd_toEndOf="@+id/etPassword"
        app:layout_constraintStart_toStartOf="@+id/etPassword"
        app:layout_constraintTop_toBottomOf="@+id/etPassword" />

    <Button
        android:id="@+id/btBack"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:background="@android:color/transparent"
        android:textColor="@android:color/black"
        android:text="to Login"
        app:layout_constraintEnd_toEndOf="@+id/btAction"
        app:layout_constraintStart_toStartOf="@+id/btAction"
        app:layout_constraintTop_toBottomOf="@+id/btAction" />
</androidx.constraintlayout.widget.ConstraintLayout>


__________________________________________________________________________________________________________________________________________________________
item_layout
__________________________________________________________________________________________________________________________________________________________


<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/tvStart_EndCityCode1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textStyle="bold"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tvStart_EndCityName1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:textSize="16sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="@id/tvStart_EndCityCode1"
        app:layout_constraintTop_toTopOf="@id/tvStart_EndCityCode1" />

    <TextView
        android:id="@+id/tvStartDate1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="@id/tvStart_EndCityName"
        app:layout_constraintTop_toBottomOf="@+id/tvStart_EndCityName" />

    <TextView
        android:id="@+id/tvEndDate1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="@id/tvStartDate"
        app:layout_constraintTop_toBottomOf="@+id/tvStartDate" />
    <TextView
        android:id="@+id/tvPrice1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintStart_toStartOf="@id/tvEndDate"
        app:layout_constraintTop_toBottomOf="@+id/tvEndDate" />

    <Button
        android:id="@+id/btFav11"
        android:layout_width="match_parent"
        android:layout_height="40dp"
        android:textColor="@color/white"
        app:layout_constraintEnd_toEndOf="@id/tvPrice1"
        app:layout_constraintStart_toStartOf="@id/tvPrice1"
        app:layout_constraintTop_toBottomOf="@+id/tvPrice"
        android:layout_marginLeft="220dp"/>

</LinearLayout>



__________________________________________________________________________________________________________________________________________________________
Android_Manifest
__________________________________________________________________________________________________________________________________________________________


<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.Room"
        tools:targetApi="31">
        <activity
            android:name=".LentaActivity"
            android:exported="false"
            android:noHistory="true"/>
        <activity
            android:name=".FavActivity"
            android:exported="false"
            android:noHistory="true"/>
        <activity
            android:name=".MainActivity"
            android:exported="false"
            android:noHistory="true" />
        <activity
            android:name=".RegActivity"
            android:exported="false"
            android:noHistory="true" />
        <activity
            android:name=".AutActivity"
            android:exported="true"
            android:noHistory="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>


INSERT INTO Record VALUES(1,'Москва','mow','Санкт-Петербург','led','2023-07-20T00:00:00Z','2023-07-25T00:00:00Z',2690,'MOW2007LED2507Y100');
INSERT INTO Record VALUES(2,'Москва','mow','Нижний Новгород','goj','2023-08-07T08:15:00Z','2023-08-13T09:10:00Z',3140,'MOW0708GOJ1308Y100');
INSERT INTO Record VALUES(3,'Москва','mow','Махачкала','mcx','2023-10-16T10:00:00Z','2023-10-20T12:00:00Z',4570,'MOW1610MCX2010Y100');
INSERT INTO Record VALUES(4,'Москва','mow','Калининград','kgd','2023-10-10T21:00:00Z','2023-10-15T13:00:00Z',4570,'MOW1010KGD1310Y100');
INSERT INTO Record VALUES(5,'Москва','mow','Казань','kzn','2023-07-21T15:00:00Z','2023-07-30T15:00:00Z',4760,'MOW2106KZN3006Y100');
INSERT INTO Record VALUES(6,'Москва','mow','Самара','kuf','2023-09-06T11:00:00Z','2023-09-11T11:00:00Z',4902,'MOW0609KUF1109Y100');
INSERT INTO Record VALUES(7,'Москва','mow','Краснодар','krr','2023-08-15T09:30:00Z','2023-08-23T10:30:00Z',4914,'MOW1504KRR2304Y100');
INSERT INTO Record VALUES(8,'Москва','mow','Екатеринбург','svx','2023-07-20T12:00:00Z','2023-07-26T12:00:00Z',5096,'MOW2006SVX2606Y100');
INSERT INTO Record VALUES(9,'Москва','mov','Волгоград','vog','2023-07-27T11:10:00Z','2023-08-10T10:20:00Z',5140,'MOW2706VOG1007Y100');
INSERT INTO Record VALUES(10,'Москва','mov','Пермь','pee','2023-07-09T21:00:00Z','2023-07-16T00:00:00Z',5140,'MOW0906PEE1606Y100');



