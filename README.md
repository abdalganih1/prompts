بالتأكيد! بناءً على طلبك، سنقوم ببناء كل الملفات المطلوبة للطبقة الكاملة للوصول إلى البيانات (Data Access Layer) لتطبيقك، بالإضافة إلى أمثلة توضيحية لكيفية استخدامها في الواجهات والخدمات.

**تنبيه هام:**
*   **الأمان:** في هذا الكود، سيتم استخدام تجزئة بسيطة (hashing) لكلمات المرور `PasswordHasher`. في تطبيق حقيقي، **يجب استخدام مكتبة تجزئة كلمات مرور قوية ومقاومة لهجمات القوة الغاشمة (Brute-force attacks) مثل BCrypt أو PBKDF2 أو Argon2.**
*   **تشغيل العمليات على Thread منفصل:** عمليات قاعدة البيانات (خاصة القراءة والكتابة) يجب أن تتم دائمًا في **خلفية Thread** لتجنب تجميد واجهة المستخدم (ANR - Application Not Responding). في هذا الكود، سأظهر الاستدعاءات مباشرة. لتطبيق حقيقي، يجب أن تغلف هذه الاستدعاءات ضمن `AsyncTask` أو `Executors.newSingleThreadExecutor()` أو Coroutines (في Kotlin) أو RxJava.
*   **إدارة الاتصالات:** في هذا المثال، نقوم بفتح (`open()`) وإغلاق (`close()`) اتصال قاعدة البيانات في كل عملية DAO. في تطبيق أكبر، قد ترغب في إدارة الاتصال بطريقة مركزية أكثر (مثلاً، فتح الاتصال عند بدء التطبيق وإغلاقه عند إيقافه، أو استخدام Singleton للـ DAO).

---

### **بنية الملفات النهائية**

```
app/src/main
├── java/com/example/keywordlistenerjava/
│   │
│   ├── activity/
│   │   ├── LoginActivity.java
│   │   ├── RegisterActivity.java
│   │   ├── MainActivity.java
│   │   ├── SettingsActivity.java
│   │   └── AlertLogActivity.java
│   │
│   ├── adapter/
│   │   ├── AlertLogAdapter.java
│   │   └── KeywordLinkAdapter.java    // لعرض الروابط في SettingsActivity
│   │
│   ├── db/
│   │   ├── DatabaseHelper.java
│   │   ├── entity/
│   │   │   ├── User.java
│   │   │   ├── Keyword.java
│   │   │   ├── EmergencyNumber.java
│   │   │   ├── KeywordNumberLink.java // جديد
│   │   │   └── AlertLog.java
│   │   └── dao/
│   │       ├── UserDao.java
│   │       ├── KeywordDao.java
│   │       ├── EmergencyNumberDao.java // جديد
│   │       ├── KeywordNumberLinkDao.java // جديد
│   │       └── AlertLogDao.java
│   │
│   ├── service/
│   │   └── PorcupineService.java
│   │
│   ├── util/
│   │   ├── SharedPreferencesHelper.java
│   │   ├── PasswordHasher.java        // جديد (للتجزئة - انظر الملاحظة)
│   │   └── LocationHelper.java
│   │
│   └── (باقي الملفات مثل MainActivity القديمة، إذا لم تكن في activity/)
│
├── res/
│   ├── layout/
│   │   ├── activity_login.xml
│   │   ├── activity_register.xml
│   │   ├── activity_main.xml
│   │   ├── activity_settings.xml
│   │   ├── activity_alert_log.xml
│   │   ├── list_item_alert.xml
│   │   └── list_item_keyword_link.xml // لتصميم عنصر الربط في الإعدادات
│   │
│   ├── menu/
│   │   └── main_menu.xml
│   │
│   └── values/
│       ├── colors.xml
│       ├── strings.xml
│       └── themes.xml
│
└── assets/
    ├── marhaban_android.ppn
    └── porcupine_params_ar.pv
```

---

### **الجزء الأول: كلاسات الـ Entity (POJOs)**

#### `db/entity/User.java` (محدث)

```java
package com.example.keywordlistenerjava.db.entity;

public class User {
    private int userId;
    private String firstName;
    private String lastName;
    private String phoneNumber;
    private String residenceArea;
    private String passwordHash;
    private String registrationDate; // Stored as TEXT in SQLite (ISO 8601 format)
    private boolean isActive; // Stored as INTEGER (0 or 1) in SQLite

    // Constructors
    public User() {
    }

    // Getters and Setters
    public int getUserId() {
        return userId;
    }

    public void setUserId(int userId) {
        this.userId = userId;
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    public String getResidenceArea() {
        return residenceArea;
    }

    public void setResidenceArea(String residenceArea) {
        this.residenceArea = residenceArea;
    }

    public String getPasswordHash() {
        return passwordHash;
    }

    public void setPasswordHash(String passwordHash) {
        this.passwordHash = passwordHash;
    }

    public String getRegistrationDate() {
        return registrationDate;
    }

    public void setRegistrationDate(String registrationDate) {
        this.registrationDate = registrationDate;
    }

    public boolean isActive() {
        return isActive;
    }

    public void setActive(boolean active) {
        isActive = active;
    }
}
```

#### `db/entity/Keyword.java` (محدث)

```java
package com.example.keywordlistenerjava.db.entity;

public class Keyword {
    private int keywordId;
    private String keywordText;
    private Integer userId; // Use Integer wrapper to allow null
    private String ppnFileName;
    private boolean isDefault;
    private String addedDate; // Stored as TEXT in SQLite

    // Constructors
    public Keyword() {
    }

    // Getters and Setters
    public int getKeywordId() {
        return keywordId;
    }

    public void setKeywordId(int keywordId) {
        this.keywordId = keywordId;
    }

    public String getKeywordText() {
        return keywordText;
    }

    public void setKeywordText(String keywordText) {
        this.keywordText = keywordText;
    }

    public Integer getUserId() {
        return userId;
    }

    public void setUserId(Integer userId) { // Accepts null
        this.userId = userId;
    }

    public String getPpnFileName() {
        return ppnFileName;
    }

    public void setPpnFileName(String ppnFileName) {
        this.ppnFileName = ppnFileName;
    }

    public boolean isDefault() {
        return isDefault;
    }

    public void setDefault(boolean aDefault) {
        isDefault = aDefault;
    }

    public String getAddedDate() {
        return addedDate;
    }

    public void setAddedDate(String addedDate) {
        this.addedDate = addedDate;
    }
}
```

#### `db/entity/EmergencyNumber.java` (محدث)

```java
package com.example.keywordlistenerjava.db.entity;

public class EmergencyNumber {
    private int numberId;
    private String phoneNumber;
    private String numberDescription;
    private Integer userId; // Use Integer wrapper to allow null
    private boolean isDefault;
    private String addedDate; // Stored as TEXT in SQLite

    // Constructors
    public EmergencyNumber() {
    }

    // Getters and Setters
    public int getNumberId() {
        return numberId;
    }

    public void setNumberId(int numberId) {
        this.numberId = numberId;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    public String getNumberDescription() {
        return numberDescription;
    }

    public void setNumberDescription(String numberDescription) {
        this.numberDescription = numberDescription;
    }

    public Integer getUserId() {
        return userId;
    }

    public void setUserId(Integer userId) { // Accepts null
        this.userId = userId;
    }

    public boolean isDefault() {
        return isDefault;
    }

    public void setDefault(boolean aDefault) {
        isDefault = aDefault;
    }

    public String getAddedDate() {
        return addedDate;
    }

    public void setAddedDate(String addedDate) {
        this.addedDate = addedDate;
    }
}
```

#### `db/entity/KeywordNumberLink.java` (جديد)

```java
package com.example.keywordlistenerjava.db.entity;

public class KeywordNumberLink {
    private int linkId;
    private int keywordId;
    private int numberId;
    private int userId; // The user who owns this specific link
    private boolean isActive; // Whether this link is currently active
    private String createdAt; // Stored as TEXT in SQLite

    // Constructors
    public KeywordNumberLink() {
    }

    // Getters and Setters
    public int getLinkId() {
        return linkId;
    }

    public void setLinkId(int linkId) {
        this.linkId = linkId;
    }

    public int getKeywordId() {
        return keywordId;
    }

    public void setKeywordId(int keywordId) {
        this.keywordId = keywordId;
    }

    public int getNumberId() {
        return numberId;
    }

    public void setNumberId(int numberId) {
        this.numberId = numberId;
    }

    public int getUserId() {
        return userId;
    }

    public void setUserId(int userId) {
        this.userId = userId;
    }

    public boolean isActive() {
        return isActive;
    }

    public void setActive(boolean active) {
        isActive = active;
    }

    public String getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(String createdAt) {
        this.createdAt = createdAt;
    }
}
```

#### `db/entity/AlertLog.java` (محدث)

```java
package com.example.keywordlistenerjava.db.entity;

public class AlertLog {
    private int logId;
    private int userId;
    private String keywordUsed;
    private String alertDate; // Stored as TEXT (YYYY-MM-DD)
    private String alertTime; // Stored as TEXT (HH:MM:SS)
    private double latitude;
    private double longitude;
    private String mapLink;
    private Boolean isFalseAlarm; // Boolean wrapper for NULL, true, false

    // Constructors
    public AlertLog() {
    }

    // Getters and Setters
    public int getLogId() {
        return logId;
    }

    public void setLogId(int logId) {
        this.logId = logId;
    }

    public int getUserId() {
        return userId;
    }

    public void setUserId(int userId) {
        this.userId = userId;
    }

    public String getKeywordUsed() {
        return keywordUsed;
    }

    public void setKeywordUsed(String keywordUsed) {
        this.keywordUsed = keywordUsed;
    }

    public String getAlertDate() {
        return alertDate;
    }

    public void setAlertDate(String alertDate) {
        this.alertDate = alertDate;
    }

    public String getAlertTime() {
        return alertTime;
    }

    public void setAlertTime(String alertTime) {
        this.alertTime = alertTime;
    }

    public double getLatitude() {
        return latitude;
    }

    public void setLatitude(double latitude) {
        this.latitude = latitude;
    }

    public double getLongitude() {
        return longitude;
    }

    public void setLongitude(double longitude) {
        this.longitude = longitude;
    }

    public String getMapLink() {
        return mapLink;
    }

    public void setMapLink(String mapLink) {
        this.mapLink = mapLink;
    }

    public Boolean getIsFalseAlarm() {
        return isFalseAlarm;
    }

    public void setIsFalseAlarm(Boolean falseAlarm) {
        isFalseAlarm = falseAlarm;
    }
}
```

---

### **الجزء الثاني: `db/DatabaseHelper.java` (محدث وموسع)**

```java
package com.example.keywordlistenerjava.db;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.util.Log;

import androidx.annotation.Nullable;

import com.example.keywordlistenerjava.db.entity.Keyword;
import com.example.keywordlistenerjava.db.entity.EmergencyNumber;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class DatabaseHelper extends SQLiteOpenHelper {

    private static final String TAG = "DatabaseHelper";

    // Database Info
    private static final String DATABASE_NAME = "SecurityApp.db";
    private static final int DATABASE_VERSION = 1;

    // Table Names
    public static final String TABLE_USERS = "Users";
    public static final String TABLE_KEYWORDS = "Keywords";
    public static final String TABLE_EMERGENCY_NUMBERS = "EmergencyNumbers";
    public static final String TABLE_KEYWORD_NUMBER_LINKS = "KeywordNumberLinks";
    public static final String TABLE_ALERT_LOG = "AlertLog";

    // Common Columns
    public static final String COLUMN_COMMON_USER_ID = "user_id";

    // Users Table - Columns
    public static final String COLUMN_USER_ID = "user_id"; // Primary Key
    public static final String COLUMN_USER_FIRST_NAME = "first_name";
    public static final String COLUMN_USER_LAST_NAME = "last_name";
    public static final String COLUMN_USER_PHONE = "phone_number";
    public static final String COLUMN_USER_RESIDENCE = "residence_area";
    public static final String COLUMN_USER_PASSWORD_HASH = "password_hash";
    public static final String COLUMN_USER_REG_DATE = "registration_date"; // TEXT (ISO 8601)
    public static final String COLUMN_USER_IS_ACTIVE = "is_active"; // INTEGER (0=false, 1=true)

    // Keywords Table - Columns
    public static final String COLUMN_KEYWORD_ID = "keyword_id"; // Primary Key
    public static final String COLUMN_KEYWORD_TEXT = "keyword_text";
    // COLUMN_KEYWORD_USER_ID is COLUMN_COMMON_USER_ID
    public static final String COLUMN_KEYWORD_PPN_FILE = "ppn_file_name";
    public static final String COLUMN_KEYWORD_IS_DEFAULT = "is_default"; // INTEGER
    public static final String COLUMN_KEYWORD_ADDED_DATE = "added_date"; // TEXT

    // EmergencyNumbers Table - Columns
    public static final String COLUMN_NUMBER_ID = "number_id"; // Primary Key
    public static final String COLUMN_NUMBER_PHONE = "phone_number";
    public static final String COLUMN_NUMBER_DESC = "number_description";
    // COLUMN_NUMBER_USER_ID is COLUMN_COMMON_USER_ID
    public static final String COLUMN_NUMBER_IS_DEFAULT = "is_default"; // INTEGER
    public static final String COLUMN_NUMBER_ADDED_DATE = "added_date"; // TEXT

    // KeywordNumberLinks Table - Columns
    public static final String COLUMN_LINK_ID = "link_id"; // Primary Key
    public static final String COLUMN_LINK_KEYWORD_ID = "keyword_id"; // FK to Keywords
    public static final String COLUMN_LINK_NUMBER_ID = "number_id"; // FK to EmergencyNumbers
    public static final String COLUMN_LINK_USER_ID = "user_id"; // FK to Users
    public static final String COLUMN_LINK_IS_ACTIVE = "is_active"; // INTEGER
    public static final String COLUMN_LINK_CREATED_AT = "created_at"; // TEXT

    // AlertLog Table - Columns
    public static final String COLUMN_LOG_ID = "log_id"; // Primary Key
    public static final String COLUMN_LOG_USER_ID = "user_id"; // FK to Users
    public static final String COLUMN_LOG_KEYWORD_USED = "keyword_used";
    public static final String COLUMN_LOG_DATE = "alert_date"; // TEXT (YYYY-MM-DD)
    public static final String COLUMN_LOG_TIME = "alert_time"; // TEXT (HH:MM:SS)
    public static final String COLUMN_LOG_LATITUDE = "latitude"; // REAL
    public static final String COLUMN_LOG_LONGITUDE = "longitude"; // REAL
    public static final String COLUMN_LOG_MAP_LINK = "map_link"; // TEXT
    public static final String COLUMN_LOG_IS_FALSE_ALARM = "is_false_alarm"; // INTEGER (NULL, 0, 1)


    // SQL CREATE Statements
    private static final String CREATE_TABLE_USERS = "CREATE TABLE " + TABLE_USERS + " (" +
            COLUMN_USER_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
            COLUMN_USER_FIRST_NAME + " TEXT NOT NULL, " +
            COLUMN_USER_LAST_NAME + " TEXT NOT NULL, " +
            COLUMN_USER_PHONE + " TEXT UNIQUE NOT NULL, " +
            COLUMN_USER_RESIDENCE + " TEXT, " +
            COLUMN_USER_PASSWORD_HASH + " TEXT NOT NULL, " +
            COLUMN_USER_REG_DATE + " TEXT DEFAULT (strftime('%Y-%m-%d %H:%M:%S','now','localtime')), " +
            COLUMN_USER_IS_ACTIVE + " INTEGER DEFAULT 1 NOT NULL" +
            ");";

    private static final String CREATE_TABLE_KEYWORDS = "CREATE TABLE " + TABLE_KEYWORDS + " (" +
            COLUMN_KEYWORD_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
            COLUMN_KEYWORD_TEXT + " TEXT NOT NULL UNIQUE, " + // UNIQUE to prevent duplicate keywords system-wide
            COLUMN_COMMON_USER_ID + " INTEGER, " + // NULL for default keywords
            COLUMN_KEYWORD_PPN_FILE + " TEXT, " +
            COLUMN_KEYWORD_IS_DEFAULT + " INTEGER DEFAULT 0 NOT NULL, " +
            COLUMN_KEYWORD_ADDED_DATE + " TEXT DEFAULT (strftime('%Y-%m-%d %H:%M:%S','now','localtime')), " +
            "FOREIGN KEY(" + COLUMN_COMMON_USER_ID + ") REFERENCES " + TABLE_USERS + "(" + COLUMN_USER_ID + ") ON DELETE CASCADE" +
            ");";

    private static final String CREATE_TABLE_EMERGENCY_NUMBERS = "CREATE TABLE " + TABLE_EMERGENCY_NUMBERS + " (" +
            COLUMN_NUMBER_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
            COLUMN_NUMBER_PHONE + " TEXT NOT NULL UNIQUE, " + // UNIQUE to prevent duplicate numbers system-wide
            COLUMN_NUMBER_DESC + " TEXT NOT NULL, " +
            COLUMN_COMMON_USER_ID + " INTEGER, " + // NULL for default numbers
            COLUMN_NUMBER_IS_DEFAULT + " INTEGER DEFAULT 0 NOT NULL, " +
            COLUMN_NUMBER_ADDED_DATE + " TEXT DEFAULT (strftime('%Y-%m-%d %H:%M:%S','now','localtime')), " +
            "FOREIGN KEY(" + COLUMN_COMMON_USER_ID + ") REFERENCES " + TABLE_USERS + "(" + COLUMN_USER_ID + ") ON DELETE CASCADE" +
            ");";

    private static final String CREATE_TABLE_KEYWORD_NUMBER_LINKS = "CREATE TABLE " + TABLE_KEYWORD_NUMBER_LINKS + " (" +
            COLUMN_LINK_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
            COLUMN_LINK_KEYWORD_ID + " INTEGER NOT NULL, " +
            COLUMN_LINK_NUMBER_ID + " INTEGER NOT NULL, " +
            COLUMN_LINK_USER_ID + " INTEGER NOT NULL, " +
            COLUMN_LINK_IS_ACTIVE + " INTEGER DEFAULT 1 NOT NULL, " +
            COLUMN_LINK_CREATED_AT + " TEXT DEFAULT (strftime('%Y-%m-%d %H:%M:%S','now','localtime')), " +
            "FOREIGN KEY(" + COLUMN_LINK_KEYWORD_ID + ") REFERENCES " + TABLE_KEYWORDS + "(" + COLUMN_KEYWORD_ID + ") ON DELETE CASCADE, " +
            "FOREIGN KEY(" + COLUMN_LINK_NUMBER_ID + ") REFERENCES " + TABLE_EMERGENCY_NUMBERS + "(" + COLUMN_NUMBER_ID + ") ON DELETE CASCADE, " +
            "FOREIGN KEY(" + COLUMN_LINK_USER_ID + ") REFERENCES " + TABLE_USERS + "(" + COLUMN_USER_ID + ") ON DELETE CASCADE, " +
            "UNIQUE(" + COLUMN_LINK_KEYWORD_ID + ", " + COLUMN_LINK_NUMBER_ID + ", " + COLUMN_LINK_USER_ID + ")" + // Prevent duplicate links for same user
            ");";

    private static final String CREATE_TABLE_ALERT_LOG = "CREATE TABLE " + TABLE_ALERT_LOG + " (" +
            COLUMN_LOG_ID + " INTEGER PRIMARY KEY AUTOINCREMENT, " +
            COLUMN_LOG_USER_ID + " INTEGER NOT NULL, " +
            COLUMN_LOG_KEYWORD_USED + " TEXT NOT NULL, " +
            COLUMN_LOG_DATE + " TEXT NOT NULL, " + // Stored as YYYY-MM-DD
            COLUMN_LOG_TIME + " TEXT NOT NULL, " + // Stored as HH:MM:SS
            COLUMN_LOG_LATITUDE + " REAL NOT NULL, " +
            COLUMN_LOG_LONGITUDE + " REAL NOT NULL, " +
            COLUMN_LOG_MAP_LINK + " TEXT, " +
            COLUMN_LOG_IS_FALSE_ALARM + " INTEGER, " + // NULL, 0=false, 1=true
            "FOREIGN KEY(" + COLUMN_LOG_USER_ID + ") REFERENCES " + TABLE_USERS + "(" + COLUMN_USER_ID + ") ON DELETE CASCADE" +
            ");";


    public DatabaseHelper(@Nullable Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        Log.i(TAG, "onCreate: Creating database tables...");
        // Enable foreign key constraints
        db.execSQL("PRAGMA foreign_keys = ON;");

        db.execSQL(CREATE_TABLE_USERS);
        db.execSQL(CREATE_TABLE_KEYWORDS);
        db.execSQL(CREATE_TABLE_EMERGENCY_NUMBERS);
        db.execSQL(CREATE_TABLE_KEYWORD_NUMBER_LINKS);
        db.execSQL(CREATE_TABLE_ALERT_LOG);
        Log.i(TAG, "onCreate: Database tables created.");

        // Add default system-wide keywords and numbers
        addSystemDefaultData(db);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        Log.w(TAG, "onUpgrade: Upgrading database from version " + oldVersion + " to " + newVersion);
        // !!! IMPORTANT: This will delete all existing data. For a real app, implement data migration.
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_ALERT_LOG);
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_KEYWORD_NUMBER_LINKS);
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_EMERGENCY_NUMBERS);
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_KEYWORDS);
        db.execSQL("DROP TABLE IF EXISTS " + TABLE_USERS);
        onCreate(db); // Recreate tables
    }

    // This method adds default keywords and numbers that are part of the system, not specific to any user.
    private void addSystemDefaultData(SQLiteDatabase db) {
        Log.i(TAG, "addSystemDefaultData: Adding default system-wide keywords and numbers...");

        // Default Keywords (10 examples)
        // Ensure you have actual .ppn files for these in your assets folder!
        addKeyword(db, "طوارئ", null, "default_emergency.ppn", true);
        addKeyword(db, "اسعاف", null, "default_ambulance.ppn", true);
        addKeyword(db, "شرطة", null, "default_police.ppn", true);
        addKeyword(db, "عنف", null, "default_violence.ppn", true);
        addKeyword(db, "خطر", null, "default_danger.ppn", true);
        addKeyword(db, "نجده", null, "default_help.ppn", true);
        addKeyword(db, "اعتداء", null, "default_assault.ppn", true);
        addKeyword(db, "سرقه", null, "default_theft.ppn", true);
        addKeyword(db, "اختطاف", null, "default_kidnap.ppn", true);
        addKeyword(db, "حريق", null, "default_fire.ppn", true);


        // Default Emergency Numbers (10 examples)
        addEmergencyNumber(db, "112", "الشرطة/الطوارئ العامة", null, true);
        addEmergencyNumber(db, "110", "الإسعاف", null, true);
        addEmergencyNumber(db, "108", "الشرطة", null, true);
        addEmergencyNumber(db, "113", "الدفاع المدني", null, true);
        addEmergencyNumber(db, "09xxxxxxxxx", "مشفى دمشق", null, true); // Example
        addEmergencyNumber(db, "09xxxxxxxxx", "المحافظة", null, true); // Example
        addEmergencyNumber(db, "09xxxxxxxxx", "الهلال الأحمر", null, true); // Example
        addEmergencyNumber(db, "09xxxxxxxxx", "الدائرة الأمنية 1", null, true); // Example
        addEmergencyNumber(db, "09xxxxxxxxx", "الدائرة الأمنية 2", null, true); // Example
        addEmergencyNumber(db, "09xxxxxxxxx", "الدائرة الأمنية 3", null, true); // Example


        // --- Example of linking default keywords to default numbers for the system ---
        // This is if you want pre-defined system-wide links that apply to all users
        // Otherwise, each user will get their own default links upon registration.
        // For simplicity, we'll let the user registration handle default linking.
        Log.i(TAG, "addSystemDefaultData: Default data added.");
    }

    // Helper method to add a keyword
    // Returns the row ID of the inserted keyword
    public long addKeyword(SQLiteDatabase db, String text, Integer userId, String ppnFile, boolean isDefault) {
        ContentValues values = new ContentValues();
        values.put(COLUMN_KEYWORD_TEXT, text);
        if (userId != null && userId != 0) { // Check for non-null and non-zero userId
            values.put(COLUMN_COMMON_USER_ID, userId);
        } else {
            values.putNull(COLUMN_COMMON_USER_ID); // Set to NULL for default keywords
        }
        values.put(COLUMN_KEYWORD_PPN_FILE, ppnFile);
        values.put(COLUMN_KEYWORD_IS_DEFAULT, isDefault ? 1 : 0);
        values.put(COLUMN_KEYWORD_ADDED_DATE, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US).format(new Date()));
        return db.insert(TABLE_KEYWORDS, null, values);
    }

    // Helper method to add an emergency number
    // Returns the row ID of the inserted number
    public long addEmergencyNumber(SQLiteDatabase db, String phone, String description, Integer userId, boolean isDefault) {
        ContentValues values = new ContentValues();
        values.put(COLUMN_NUMBER_PHONE, phone);
        values.put(COLUMN_NUMBER_DESC, description);
        if (userId != null && userId != 0) { // Check for non-null and non-zero userId
            values.put(COLUMN_COMMON_USER_ID, userId);
        } else {
            values.putNull(COLUMN_COMMON_USER_ID); // Set to NULL for default numbers
        }
        values.put(COLUMN_NUMBER_IS_DEFAULT, isDefault ? 1 : 0);
        values.put(COLUMN_NUMBER_ADDED_DATE, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US).format(new Date()));
        return db.insert(TABLE_EMERGENCY_NUMBERS, null, values);
    }

    // Helper methods to get IDs of default keywords/numbers (useful for linking)
    public long getDefaultKeywordId(SQLiteDatabase db, String keywordText) {
        Cursor cursor = null;
        long id = -1;
        try {
            cursor = db.query(
                    TABLE_KEYWORDS,
                    new String[]{COLUMN_KEYWORD_ID},
                    COLUMN_KEYWORD_TEXT + " = ? AND " + COLUMN_KEYWORD_IS_DEFAULT + " = 1",
                    new String[]{keywordText},
                    null, null, null
            );
            if (cursor != null && cursor.moveToFirst()) {
                id = cursor.getLong(cursor.getColumnIndexOrThrow(COLUMN_KEYWORD_ID));
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting default keyword ID for " + keywordText, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return id;
    }

    public long getDefaultNumberId(SQLiteDatabase db, String phoneNumber) {
        Cursor cursor = null;
        long id = -1;
        try {
            cursor = db.query(
                    TABLE_EMERGENCY_NUMBERS,
                    new String[]{COLUMN_NUMBER_ID},
                    COLUMN_NUMBER_PHONE + " = ? AND " + COLUMN_NUMBER_IS_DEFAULT + " = 1",
                    new String[]{phoneNumber},
                    null, null, null
            );
            if (cursor != null && cursor.moveToFirst()) {
                id = cursor.getLong(cursor.getColumnIndexOrThrow(COLUMN_NUMBER_ID));
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting default number ID for " + phoneNumber, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return id;
    }
}
```

---

### **الجزء الثالث: كلاسات الـ DAO (Data Access Objects)**

#### `db/dao/UserDao.java` (محدث وموسع)

```java
package com.example.keywordlistenerjava.db.dao;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.util.Log;

import com.example.keywordlistenerjava.db.DatabaseHelper;
import com.example.keywordlistenerjava.db.entity.User;
import com.example.keywordlistenerjava.util.PasswordHasher; // لاستخدام تجزئة كلمة المرور

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class UserDao {
    private static final String TAG = "UserDao";
    private SQLiteDatabase db;
    private DatabaseHelper dbHelper;
    private Context context; // Keep context to pass to other DAOs for default linking

    public UserDao(Context context) {
        this.context = context;
        dbHelper = new DatabaseHelper(context);
    }

    public void open() {
        db = dbHelper.getWritableDatabase();
    }

    public void close() {
        dbHelper.close();
    }

    /**
     * Inserts a new user into the database and links them to default keywords and numbers.
     * @param user The user object to insert (password should be plain text here, will be hashed).
     * @return The user_id of the newly inserted row, or -1 if an error occurred.
     */
    public long registerUser(User user) {
        long userId = -1;
        db.beginTransaction(); // Start a transaction for atomicity
        try {
            // Hash the password
            String hashedPassword = PasswordHasher.hashPassword(user.getPasswordHash());

            ContentValues values = new ContentValues();
            values.put(DatabaseHelper.COLUMN_USER_FIRST_NAME, user.getFirstName());
            values.put(DatabaseHelper.COLUMN_USER_LAST_NAME, user.getLastName());
            values.put(DatabaseHelper.COLUMN_USER_PHONE, user.getPhoneNumber());
            values.put(DatabaseHelper.COLUMN_USER_RESIDENCE, user.getResidenceArea());
            values.put(DatabaseHelper.COLUMN_USER_PASSWORD_HASH, hashedPassword);
            values.put(DatabaseHelper.COLUMN_USER_REG_DATE, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US).format(new Date()));
            values.put(DatabaseHelper.COLUMN_USER_IS_ACTIVE, 1); // Default to active

            userId = db.insert(DatabaseHelper.TABLE_USERS, null, values);

            if (userId != -1) {
                // Link default keywords and numbers for the new user
                linkDefaultKeywordsAndNumbersToUser((int) userId);
                db.setTransactionSuccessful(); // Mark transaction as successful
                Log.i(TAG, "User registered and default links created successfully. User ID: " + userId);
            } else {
                Log.e(TAG, "Failed to insert user into database.");
            }
        } catch (Exception e) {
            Log.e(TAG, "Error registering user: " + e.getMessage(), e);
        } finally {
            db.endTransaction(); // End transaction (commit or rollback)
        }
        return userId;
    }

    /**
     * Authenticates a user by phone number and password.
     * @param phoneNumber The user's phone number.
     * @param password The plain text password.
     * @return The User object if authenticated, null otherwise.
     */
    public User authenticateUser(String phoneNumber, String password) {
        Cursor cursor = null;
        User user = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_USERS,
                    null, // all columns
                    DatabaseHelper.COLUMN_USER_PHONE + " = ?",
                    new String[]{phoneNumber},
                    null, null, null
            );

            if (cursor != null && cursor.moveToFirst()) {
                String storedHashedPassword = cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_PASSWORD_HASH));
                if (PasswordHasher.verifyPassword(password, storedHashedPassword)) {
                    user = cursorToUser(cursor);
                    Log.i(TAG, "User authenticated: " + phoneNumber);
                } else {
                    Log.w(TAG, "Authentication failed: Incorrect password for " + phoneNumber);
                }
            } else {
                Log.w(TAG, "Authentication failed: User not found with phone number " + phoneNumber);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error authenticating user: " + e.getMessage(), e);
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
        return user;
    }

    /**
     * Retrieves a user by their user ID.
     * @param userId The ID of the user.
     * @return A User object if found, otherwise null.
     */
    public User getUserById(int userId) {
        Cursor cursor = null;
        User user = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_USERS,
                    null, // all columns
                    DatabaseHelper.COLUMN_USER_ID + " = ?",
                    new String[]{String.valueOf(userId)},
                    null, null, null
            );

            if (cursor != null && cursor.moveToFirst()) {
                user = cursorToUser(cursor);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting user by ID: " + userId, e);
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
        return user;
    }

    // Helper method to convert Cursor to User object
    private User cursorToUser(Cursor cursor) {
        User user = new User();
        user.setUserId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_ID)));
        user.setFirstName(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_FIRST_NAME)));
        user.setLastName(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_LAST_NAME)));
        user.setPhoneNumber(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_PHONE)));
        user.setResidenceArea(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_RESIDENCE)));
        user.setPasswordHash(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_PASSWORD_HASH)));
        user.setRegistrationDate(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_REG_DATE)));
        user.setActive(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_USER_IS_ACTIVE)) == 1);
        return user;
    }

    // This method creates default links for a newly registered user
    private void linkDefaultKeywordsAndNumbersToUser(int userId) {
        KeywordDao keywordDao = new KeywordDao(context);
        EmergencyNumberDao numberDao = new EmergencyNumberDao(context);
        KeywordNumberLinkDao linkDao = new KeywordNumberLinkDao(context);

        try {
            keywordDao.open();
            numberDao.open();
            linkDao.open();

            // Get all default keywords and numbers
            List<Keyword> defaultKeywords = keywordDao.getAllDefaultKeywords();
            List<EmergencyNumber> defaultNumbers = numberDao.getAllDefaultNumbers();

            // Create links between default keywords and default numbers for the new user
            // This example creates a link between each default keyword and each default number.
            // You might want to customize this logic (e.g., link only specific keywords to specific numbers).
            for (Keyword keyword : defaultKeywords) {
                for (EmergencyNumber number : defaultNumbers) {
                    // Create link record
                    KeywordNumberLink link = new KeywordNumberLink();
                    link.setKeywordId(keyword.getKeywordId());
                    link.setNumberId(number.getNumberId());
                    link.setUserId(userId);
                    link.setActive(true); // Default links are active

                    long linkId = linkDao.addLink(link);
                    if (linkId != -1) {
                        Log.d(TAG, "Default link created: Keyword '" + keyword.getKeywordText() + "' to Number '" + number.getPhoneNumber() + "' for User ID " + userId);
                    } else {
                        Log.e(TAG, "Failed to create default link: Keyword '" + keyword.getKeywordText() + "' to Number '" + number.getPhoneNumber() + "' for User ID " + userId);
                    }
                }
            }
        } catch (Exception e) {
            Log.e(TAG, "Error linking default keywords/numbers to new user: " + e.getMessage(), e);
        } finally {
            linkDao.close();
            numberDao.close();
            keywordDao.close();
        }
    }
}
```

#### `db/dao/KeywordDao.java` (مفصل)

```java
package com.example.keywordlistenerjava.db.dao;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.util.Log;

import com.example.keywordlistenerjava.db.DatabaseHelper;
import com.example.keywordlistenerjava.db.entity.Keyword;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

public class KeywordDao {
    private static final String TAG = "KeywordDao";
    private SQLiteDatabase db;
    private DatabaseHelper dbHelper;

    private String[] allColumns = {
            DatabaseHelper.COLUMN_KEYWORD_ID,
            DatabaseHelper.COLUMN_KEYWORD_TEXT,
            DatabaseHelper.COLUMN_KEYWORD_USER_ID,
            DatabaseHelper.COLUMN_KEYWORD_PPN_FILE,
            DatabaseHelper.COLUMN_KEYWORD_IS_DEFAULT,
            DatabaseHelper.COLUMN_KEYWORD_ADDED_DATE
    };

    public KeywordDao(Context context) {
        dbHelper = new DatabaseHelper(context);
    }

    public void open() {
        db = dbHelper.getWritableDatabase();
    }

    public void close() {
        dbHelper.close();
    }

    /**
     * Adds a new keyword to the database.
     * @param keyword The Keyword object to insert.
     * @return The row ID of the newly inserted row, or -1 if an error occurred.
     */
    public long addKeyword(Keyword keyword) {
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COLUMN_KEYWORD_TEXT, keyword.getKeywordText());
        if (keyword.getUserId() != null && keyword.getUserId() != 0) {
            values.put(DatabaseHelper.COLUMN_KEYWORD_USER_ID, keyword.getUserId());
        } else {
            values.putNull(DatabaseHelper.COLUMN_KEYWORD_USER_ID);
        }
        values.put(DatabaseHelper.COLUMN_KEYWORD_PPN_FILE, keyword.getPpnFileName());
        values.put(DatabaseHelper.COLUMN_KEYWORD_IS_DEFAULT, keyword.isDefault() ? 1 : 0);
        values.put(DatabaseHelper.COLUMN_KEYWORD_ADDED_DATE, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US).format(new Date()));

        return db.insert(DatabaseHelper.TABLE_KEYWORDS, null, values);
    }

    /**
     * Retrieves a keyword by its ID.
     * @param keywordId The ID of the keyword.
     * @return A Keyword object if found, otherwise null.
     */
    public Keyword getKeywordById(int keywordId) {
        Cursor cursor = null;
        Keyword keyword = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_KEYWORDS,
                    allColumns,
                    DatabaseHelper.COLUMN_KEYWORD_ID + " = ?",
                    new String[]{String.valueOf(keywordId)},
                    null, null, null
            );
            if (cursor != null && cursor.moveToFirst()) {
                keyword = cursorToKeyword(cursor);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting keyword by ID: " + keywordId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return keyword;
    }

    /**
     * Retrieves a keyword by its text (case-insensitive for comparison).
     * This will get the first matching keyword.
     * @param keywordText The text of the keyword.
     * @return A Keyword object if found, otherwise null.
     */
    public Keyword getKeywordByText(String keywordText) {
        Cursor cursor = null;
        Keyword keyword = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_KEYWORDS,
                    allColumns,
                    DatabaseHelper.COLUMN_KEYWORD_TEXT + " = ?",
                    new String[]{keywordText}, // Exact match, consider using COLLATE NOCASE for case-insensitivity
                    null, null, null
            );
            if (cursor != null && cursor.moveToFirst()) {
                keyword = cursorToKeyword(cursor);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting keyword by text: " + keywordText, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return keyword;
    }

    /**
     * Retrieves all keywords associated with a specific user (including default ones if linked).
     * For now, this only retrieves keywords that have the user_id assigned.
     * To get all keywords a user *can* use (user-specific + default system-wide),
     * you'd need a more complex query involving UNION or separate calls.
     * @param userId The ID of the user.
     * @return A list of Keyword objects.
     */
    public List<Keyword> getAllKeywordsForUser(int userId) {
        List<Keyword> keywords = new ArrayList<>();
        Cursor cursor = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_KEYWORDS,
                    allColumns,
                    DatabaseHelper.COLUMN_KEYWORD_USER_ID + " = ? OR " + DatabaseHelper.COLUMN_KEYWORD_IS_DEFAULT + " = 1", // Get user's custom keywords AND default ones
                    new String[]{String.valueOf(userId)},
                    null, null, null
            );
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                keywords.add(cursorToKeyword(cursor));
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting all keywords for user: " + userId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return keywords;
    }

    /**
     * Retrieves all default (system-wide) keywords.
     * @return A list of Keyword objects.
     */
    public List<Keyword> getAllDefaultKeywords() {
        List<Keyword> keywords = new ArrayList<>();
        Cursor cursor = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_KEYWORDS,
                    allColumns,
                    DatabaseHelper.COLUMN_KEYWORD_IS_DEFAULT + " = 1",
                    null, null, null, null
            );
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                keywords.add(cursorToKeyword(cursor));
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting all default keywords.", e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return keywords;
    }


    /**
     * Deletes a keyword by its ID.
     * @param keywordId The ID of the keyword to delete.
     * @return The number of rows affected (should be 1 if successful).
     */
    public int deleteKeyword(int keywordId) {
        return db.delete(
                DatabaseHelper.TABLE_KEYWORDS,
                DatabaseHelper.COLUMN_KEYWORD_ID + " = ?",
                new String[]{String.valueOf(keywordId)}
        );
    }

    // Helper method to convert Cursor to Keyword object
    private Keyword cursorToKeyword(Cursor cursor) {
        Keyword keyword = new Keyword();
        keyword.setKeywordId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_KEYWORD_ID)));
        keyword.setKeywordText(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_KEYWORD_TEXT)));
        
        int userIdIndex = cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_KEYWORD_USER_ID);
        if (!cursor.isNull(userIdIndex)) {
            keyword.setUserId(cursor.getInt(userIdIndex));
        } else {
            keyword.setUserId(null); // For default keywords
        }

        keyword.setPpnFileName(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_KEYWORD_PPN_FILE)));
        keyword.setDefault(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_KEYWORD_IS_DEFAULT)) == 1);
        keyword.setAddedDate(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_KEYWORD_ADDED_DATE)));
        return keyword;
    }
}
```

#### `db/dao/EmergencyNumberDao.java` (مفصل)

```java
package com.example.keywordlistenerjava.db.dao;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.util.Log;

import com.example.keywordlistenerjava.db.DatabaseHelper;
import com.example.keywordlistenerjava.db.entity.EmergencyNumber;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

public class EmergencyNumberDao {
    private static final String TAG = "EmergencyNumberDao";
    private SQLiteDatabase db;
    private DatabaseHelper dbHelper;

    private String[] allColumns = {
            DatabaseHelper.COLUMN_NUMBER_ID,
            DatabaseHelper.COLUMN_NUMBER_PHONE,
            DatabaseHelper.COLUMN_NUMBER_DESC,
            DatabaseHelper.COLUMN_NUMBER_USER_ID,
            DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT,
            DatabaseHelper.COLUMN_NUMBER_ADDED_DATE
    };

    public EmergencyNumberDao(Context context) {
        dbHelper = new DatabaseHelper(context);
    }

    public void open() {
        db = dbHelper.getWritableDatabase();
    }

    public void close() {
        dbHelper.close();
    }

    /**
     * Adds a new emergency number to the database.
     * @param number The EmergencyNumber object to insert.
     * @return The row ID of the newly inserted row, or -1 if an error occurred.
     */
    public long addEmergencyNumber(EmergencyNumber number) {
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COLUMN_NUMBER_PHONE, number.getPhoneNumber());
        values.put(DatabaseHelper.COLUMN_NUMBER_DESC, number.getNumberDescription());
        if (number.getUserId() != null && number.getUserId() != 0) {
            values.put(DatabaseHelper.COLUMN_NUMBER_USER_ID, number.getUserId());
        } else {
            values.putNull(DatabaseHelper.COLUMN_NUMBER_USER_ID);
        }
        values.put(DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT, number.isDefault() ? 1 : 0);
        values.put(DatabaseHelper.COLUMN_NUMBER_ADDED_DATE, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US).format(new Date()));

        return db.insert(DatabaseHelper.TABLE_EMERGENCY_NUMBERS, null, values);
    }

    /**
     * Retrieves an emergency number by its ID.
     * @param numberId The ID of the number.
     * @return An EmergencyNumber object if found, otherwise null.
     */
    public EmergencyNumber getEmergencyNumberById(int numberId) {
        Cursor cursor = null;
        EmergencyNumber number = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_EMERGENCY_NUMBERS,
                    allColumns,
                    DatabaseHelper.COLUMN_NUMBER_ID + " = ?",
                    new String[]{String.valueOf(numberId)},
                    null, null, null
            );
            if (cursor != null && cursor.moveToFirst()) {
                number = cursorToEmergencyNumber(cursor);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting emergency number by ID: " + numberId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return number;
    }

    /**
     * Retrieves all emergency numbers associated with a specific user (including default ones if linked).
     * @param userId The ID of the user.
     * @return A list of EmergencyNumber objects.
     */
    public List<EmergencyNumber> getAllEmergencyNumbersForUser(int userId) {
        List<EmergencyNumber> numbers = new ArrayList<>();
        Cursor cursor = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_EMERGENCY_NUMBERS,
                    allColumns,
                    DatabaseHelper.COLUMN_NUMBER_USER_ID + " = ? OR " + DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT + " = 1",
                    new String[]{String.valueOf(userId)},
                    null, null, null
            );
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                numbers.add(cursorToEmergencyNumber(cursor));
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting all emergency numbers for user: " + userId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return numbers;
    }

    /**
     * Retrieves all default (system-wide) emergency numbers.
     * @return A list of EmergencyNumber objects.
     */
    public List<EmergencyNumber> getAllDefaultNumbers() {
        List<EmergencyNumber> numbers = new ArrayList<>();
        Cursor cursor = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_EMERGENCY_NUMBERS,
                    allColumns,
                    DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT + " = 1",
                    null, null, null, null
            );
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                numbers.add(cursorToEmergencyNumber(cursor));
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting all default numbers.", e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return numbers;
    }


    /**
     * Deletes an emergency number by its ID.
     * @param numberId The ID of the number to delete.
     * @return The number of rows affected (should be 1 if successful).
     */
    public int deleteEmergencyNumber(int numberId) {
        return db.delete(
                DatabaseHelper.TABLE_EMERGENCY_NUMBERS,
                DatabaseHelper.COLUMN_NUMBER_ID + " = ?",
                new String[]{String.valueOf(numberId)}
        );
    }

    // Helper method to convert Cursor to EmergencyNumber object
    private EmergencyNumber cursorToEmergencyNumber(Cursor cursor) {
        EmergencyNumber number = new EmergencyNumber();
        number.setNumberId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_ID)));
        number.setPhoneNumber(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_PHONE)));
        number.setNumberDescription(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_DESC)));
        
        int userIdIndex = cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_USER_ID);
        if (!cursor.isNull(userIdIndex)) {
            number.setUserId(cursor.getInt(userIdIndex));
        } else {
            number.setUserId(null); // For default numbers
        }

        number.setDefault(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT)) == 1);
        number.setAddedDate(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_ADDED_DATE)));
        return number;
    }
}
```

#### `db/dao/KeywordNumberLinkDao.java` (جديد ومفصل)

```java
package com.example.keywordlistenerjava.db.dao;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.util.Log;

import com.example.keywordlistenerjava.db.DatabaseHelper;
import com.example.keywordlistenerjava.db.entity.KeywordNumberLink;
import com.example.keywordlistenerjava.db.entity.Keyword; // Needed for linked keyword data
import com.example.keywordlistenerjava.db.entity.EmergencyNumber; // Needed for linked number data

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

public class KeywordNumberLinkDao {
    private static final String TAG = "KeywordNumberLinkDao";
    private SQLiteDatabase db;
    private DatabaseHelper dbHelper;

    private String[] allColumns = {
            DatabaseHelper.COLUMN_LINK_ID,
            DatabaseHelper.COLUMN_LINK_KEYWORD_ID,
            DatabaseHelper.COLUMN_LINK_NUMBER_ID,
            DatabaseHelper.COLUMN_LINK_USER_ID,
            DatabaseHelper.COLUMN_LINK_IS_ACTIVE,
            DatabaseHelper.COLUMN_LINK_CREATED_AT
    };

    public KeywordNumberLinkDao(Context context) {
        dbHelper = new DatabaseHelper(context);
    }

    public void open() {
        db = dbHelper.getWritableDatabase();
    }

    public void close() {
        dbHelper.close();
    }

    /**
     * Adds a new keyword-number link.
     * @param link The KeywordNumberLink object to insert.
     * @return The row ID of the newly inserted row, or -1 if an error occurred.
     */
    public long addLink(KeywordNumberLink link) {
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COLUMN_LINK_KEYWORD_ID, link.getKeywordId());
        values.put(DatabaseHelper.COLUMN_LINK_NUMBER_ID, link.getNumberId());
        values.put(DatabaseHelper.COLUMN_LINK_USER_ID, link.getUserId());
        values.put(DatabaseHelper.COLUMN_LINK_IS_ACTIVE, link.isActive() ? 1 : 0);
        values.put(DatabaseHelper.COLUMN_LINK_CREATED_AT, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US).format(new Date()));

        return db.insert(DatabaseHelper.TABLE_KEYWORD_NUMBER_LINKS, null, values);
    }

    /**
     * Retrieves all active links for a specific user.
     * @param userId The ID of the user.
     * @return A list of KeywordNumberLink objects.
     */
    public List<KeywordNumberLink> getAllActiveLinksForUser(int userId) {
        List<KeywordNumberLink> links = new ArrayList<>();
        Cursor cursor = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_KEYWORD_NUMBER_LINKS,
                    allColumns,
                    DatabaseHelper.COLUMN_LINK_USER_ID + " = ? AND " + DatabaseHelper.COLUMN_LINK_IS_ACTIVE + " = 1",
                    new String[]{String.valueOf(userId)},
                    null, null, null
            );
            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                links.add(cursorToLink(cursor));
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting all active links for user: " + userId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return links;
    }

    /**
     * Retrieves all active emergency phone numbers linked to a specific keyword for a user.
     * This method joins with Keywords and EmergencyNumbers tables to get full details.
     * @param userId The ID of the user.
     * @param keywordText The text of the keyword detected.
     * @return A list of EmergencyNumber objects linked to the keyword.
     */
    public List<EmergencyNumber> getEmergencyNumbersForKeyword(int userId, String keywordText) {
        List<EmergencyNumber> numbers = new ArrayList<>();
        Cursor cursor = null;
        try {
            String query = "SELECT T2." + DatabaseHelper.COLUMN_NUMBER_ID + ", " +
                    "T2." + DatabaseHelper.COLUMN_NUMBER_PHONE + ", " +
                    "T2." + DatabaseHelper.COLUMN_NUMBER_DESC + ", " +
                    "T2." + DatabaseHelper.COLUMN_NUMBER_USER_ID + ", " +
                    "T2." + DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT + ", " +
                    "T2." + DatabaseHelper.COLUMN_NUMBER_ADDED_DATE +
                    " FROM " + DatabaseHelper.TABLE_KEYWORD_NUMBER_LINKS + " AS T1" +
                    " JOIN " + DatabaseHelper.TABLE_EMERGENCY_NUMBERS + " AS T2 ON T1." + DatabaseHelper.COLUMN_LINK_NUMBER_ID + " = T2." + DatabaseHelper.COLUMN_NUMBER_ID +
                    " JOIN " + DatabaseHelper.TABLE_KEYWORDS + " AS T3 ON T1." + DatabaseHelper.COLUMN_LINK_KEYWORD_ID + " = T3." + DatabaseHelper.COLUMN_KEYWORD_ID +
                    " WHERE T1." + DatabaseHelper.COLUMN_LINK_USER_ID + " = ? AND T3." + DatabaseHelper.COLUMN_KEYWORD_TEXT + " = ? AND T1." + DatabaseHelper.COLUMN_LINK_IS_ACTIVE + " = 1;";

            cursor = db.rawQuery(query, new String[]{String.valueOf(userId), keywordText});

            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                EmergencyNumber number = new EmergencyNumber();
                number.setNumberId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_ID)));
                number.setPhoneNumber(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_PHONE)));
                number.setNumberDescription(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_DESC)));
                
                int numUserIdIndex = cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_USER_ID);
                if (!cursor.isNull(numUserIdIndex)) {
                    number.setUserId(cursor.getInt(numUserIdIndex));
                } else {
                    number.setUserId(null);
                }
                
                number.setDefault(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_IS_DEFAULT)) == 1);
                number.setAddedDate(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_NUMBER_ADDED_DATE)));
                numbers.add(number);
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting emergency numbers for keyword '" + keywordText + "' for user " + userId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return numbers;
    }


    /**
     * Activates or deactivates a specific link.
     * @param linkId The ID of the link to update.
     * @param isActive True to activate, false to deactivate.
     * @return The number of rows affected (should be 1 if successful).
     */
    public int updateLinkStatus(int linkId, boolean isActive) {
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COLUMN_LINK_IS_ACTIVE, isActive ? 1 : 0);

        return db.update(
                DatabaseHelper.TABLE_KEYWORD_NUMBER_LINKS,
                values,
                DatabaseHelper.COLUMN_LINK_ID + " = ?",
                new String[]{String.valueOf(linkId)}
        );
    }

    /**
     * Deletes a specific link.
     * @param linkId The ID of the link to delete.
     * @return The number of rows affected (should be 1 if successful).
     */
    public int deleteLink(int linkId) {
        return db.delete(
                DatabaseHelper.TABLE_KEYWORD_NUMBER_LINKS,
                DatabaseHelper.COLUMN_LINK_ID + " = ?",
                new String[]{String.valueOf(linkId)}
        );
    }
    
    // Check if a specific link exists for a user (useful before adding)
    public boolean linkExists(int userId, int keywordId, int numberId) {
        Cursor cursor = null;
        try {
            cursor = db.query(
                DatabaseHelper.TABLE_KEYWORD_NUMBER_LINKS,
                new String[]{DatabaseHelper.COLUMN_LINK_ID},
                DatabaseHelper.COLUMN_LINK_USER_ID + " = ? AND " +
                DatabaseHelper.COLUMN_LINK_KEYWORD_ID + " = ? AND " +
                DatabaseHelper.COLUMN_LINK_NUMBER_ID + " = ?",
                new String[]{String.valueOf(userId), String.valueOf(keywordId), String.valueOf(numberId)},
                null, null, null
            );
            return cursor != null && cursor.getCount() > 0;
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
    }


    // Helper method to convert Cursor to KeywordNumberLink object
    private KeywordNumberLink cursorToLink(Cursor cursor) {
        KeywordNumberLink link = new KeywordNumberLink();
        link.setLinkId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LINK_ID)));
        link.setKeywordId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LINK_KEYWORD_ID)));
        link.setNumberId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LINK_NUMBER_ID)));
        link.setUserId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LINK_USER_ID)));
        link.setActive(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LINK_IS_ACTIVE)) == 1);
        link.setCreatedAt(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LINK_CREATED_AT)));
        return link;
    }
}
```

#### `db/dao/AlertLogDao.java` (محدث وموسع)

```java
package com.example.keywordlistenerjava.db.dao;

import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.util.Log;

import com.example.keywordlistenerjava.db.DatabaseHelper;
import com.example.keywordlistenerjava.db.entity.AlertLog;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

public class AlertLogDao {
    private static final String TAG = "AlertLogDao";
    private SQLiteDatabase db;
    private DatabaseHelper dbHelper;
    private String[] allColumns = {
            DatabaseHelper.COLUMN_LOG_ID,
            DatabaseHelper.COLUMN_LOG_USER_ID,
            DatabaseHelper.COLUMN_LOG_KEYWORD_USED,
            DatabaseHelper.COLUMN_LOG_DATE,
            DatabaseHelper.COLUMN_LOG_TIME,
            DatabaseHelper.COLUMN_LOG_LATITUDE,
            DatabaseHelper.COLUMN_LOG_LONGITUDE,
            DatabaseHelper.COLUMN_LOG_MAP_LINK,
            DatabaseHelper.COLUMN_LOG_IS_FALSE_ALARM
    };

    public AlertLogDao(Context context) {
        dbHelper = new DatabaseHelper(context);
    }

    public void open() {
        db = dbHelper.getWritableDatabase();
    }

    public void close() {
        dbHelper.close();
    }

    /**
     * Inserts a new alert log into the database.
     * @param alertLog The AlertLog object to insert.
     * @return The row ID of the newly inserted row, or -1 if an error occurred.
     */
    public long addAlertLog(AlertLog alertLog) {
        ContentValues values = new ContentValues();
        values.put(DatabaseHelper.COLUMN_LOG_USER_ID, alertLog.getUserId());
        values.put(DatabaseHelper.COLUMN_LOG_KEYWORD_USED, alertLog.getKeywordUsed());
        values.put(DatabaseHelper.COLUMN_LOG_DATE, new SimpleDateFormat("yyyy-MM-dd", Locale.US).format(new Date()));
        values.put(DatabaseHelper.COLUMN_LOG_TIME, new SimpleDateFormat("HH:mm:ss", Locale.US).format(new Date()));
        values.put(DatabaseHelper.COLUMN_LOG_LATITUDE, alertLog.getLatitude());
        values.put(DatabaseHelper.COLUMN_LOG_LONGITUDE, alertLog.getLongitude());
        values.put(DatabaseHelper.COLUMN_LOG_MAP_LINK, alertLog.getMapLink());
        // isFalseAlarm is initially null, so no need to put it here

        return db.insert(DatabaseHelper.TABLE_ALERT_LOG, null, values);
    }

    /**
     * Retrieves all alert logs for a specific user, ordered by most recent first.
     * @param userId The ID of the user.
     * @return A list of AlertLog objects.
     */
    public List<AlertLog> getAllAlertsForUser(int userId) {
        List<AlertLog> alertLogs = new ArrayList<>();
        Cursor cursor = null;
        try {
            cursor = db.query(
                    DatabaseHelper.TABLE_ALERT_LOG,
                    allColumns,
                    DatabaseHelper.COLUMN_LOG_USER_ID + " = ?",
                    new String[]{String.valueOf(userId)},
                    null, null,
                    DatabaseHelper.COLUMN_LOG_DATE + " DESC, " + DatabaseHelper.COLUMN_LOG_TIME + " DESC"
            );

            cursor.moveToFirst();
            while (!cursor.isAfterLast()) {
                alertLogs.add(cursorToAlertLog(cursor));
                cursor.moveToNext();
            }
        } catch (Exception e) {
            Log.e(TAG, "Error getting all alerts for user: " + userId, e);
        } finally {
            if (cursor != null) cursor.close();
        }
        return alertLogs;
    }

    /**
     * Updates the 'is_false_alarm' status of an alert log.
     * This is typically done by an admin or user correcting a log.
     * @param logId The ID of the alert log to update.
     * @param isFalseAlarm True for false alarm, false for real, null to unset.
     * @return The number of rows affected (should be 1 if successful).
     */
    public int updateFalseAlarmStatus(int logId, Boolean isFalseAlarm) {
        ContentValues values = new ContentValues();
        if (isFalseAlarm == null) {
            values.putNull(DatabaseHelper.COLUMN_LOG_IS_FALSE_ALARM);
        } else {
            values.put(DatabaseHelper.COLUMN_LOG_IS_FALSE_ALARM, isFalseAlarm ? 1 : 0);
        }

        return db.update(
                DatabaseHelper.TABLE_ALERT_LOG,
                values,
                DatabaseHelper.COLUMN_LOG_ID + " = ?",
                new String[]{String.valueOf(logId)}
        );
    }

    /**
     * Gets the total count of alerts for a user.
     * @param userId The user's ID.
     * @return Total alert count.
     */
    public int getTotalAlertsCount(int userId) {
        Cursor cursor = null;
        int count = 0;
        try {
            cursor = db.rawQuery("SELECT COUNT(*) FROM " + DatabaseHelper.TABLE_ALERT_LOG + " WHERE " + DatabaseHelper.COLUMN_LOG_USER_ID + " = ?", new String[]{String.valueOf(userId)});
            if (cursor != null && cursor.moveToFirst()) {
                count = cursor.getInt(0);
            }
        } finally {
            if (cursor != null) cursor.close();
        }
        return count;
    }

    /**
     * Gets the count of real alerts for a user.
     * @param userId The user's ID.
     * @return Real alert count.
     */
    public int getRealAlertsCount(int userId) {
        Cursor cursor = null;
        int count = 0;
        try {
            cursor = db.rawQuery("SELECT COUNT(*) FROM " + DatabaseHelper.TABLE_ALERT_LOG + " WHERE " + DatabaseHelper.COLUMN_LOG_USER_ID + " = ? AND " + DatabaseHelper.COLUMN_LOG_IS_FALSE_ALARM + " = 0", new String[]{String.valueOf(userId)});
            if (cursor != null && cursor.moveToFirst()) {
                count = cursor.getInt(0);
            }
        } finally {
            if (cursor != null) cursor.close();
        }
        return count;
    }

    /**
     * Gets the count of false alerts for a user.
     * @param userId The user's ID.
     * @return False alert count.
     */
    public int getFalseAlertsCount(int userId) {
        Cursor cursor = null;
        int count = 0;
        try {
            cursor = db.rawQuery("SELECT COUNT(*) FROM " + DatabaseHelper.TABLE_ALERT_LOG + " WHERE " + DatabaseHelper.COLUMN_LOG_USER_ID + " = ? AND " + DatabaseHelper.COLUMN_LOG_IS_FALSE_ALARM + " = 1", new String[]{String.valueOf(userId)});
            if (cursor != null && cursor.moveToFirst()) {
                count = cursor.getInt(0);
            }
        } finally {
            if (cursor != null) cursor.close();
        }
        return count;
    }

    // Helper method to convert Cursor to AlertLog object
    private AlertLog cursorToAlertLog(Cursor cursor) {
        AlertLog alertLog = new AlertLog();
        alertLog.setLogId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_ID)));
        alertLog.setUserId(cursor.getInt(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_USER_ID)));
        alertLog.setKeywordUsed(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_KEYWORD_USED)));
        alertLog.setAlertDate(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_DATE)));
        alertLog.setAlertTime(cursor.getString(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_TIME)));
        alertLog.setLatitude(cursor.getDouble(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_LATITUDE)));
        alertLog.setLongitude(cursor.getDouble(cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_LONGITUDE)));
        
        int mapLinkIndex = cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_MAP_LINK);
        if (!cursor.isNull(mapLinkIndex)) {
            alertLog.setMapLink(cursor.getString(mapLinkIndex));
        } else {
            alertLog.setMapLink(null);
        }

        int isFalseAlarmIndex = cursor.getColumnIndexOrThrow(DatabaseHelper.COLUMN_LOG_IS_FALSE_ALARM);
        if (cursor.isNull(isFalseAlarmIndex)) {
            alertLog.setIsFalseAlarm(null);
        } else {
            alertLog.setIsFalseAlarm(cursor.getInt(isFalseAlarmIndex) == 1);
        }
        return alertLog;
    }
}
```

---

### **الجزء الرابع: كلاسات الـ Utilities المساعدة**

#### `util/SharedPreferencesHelper.java`

```java
package com.example.keywordlistenerjava.util;

import android.content.Context;
import android.content.SharedPreferences;

public class SharedPreferencesHelper {
    private static final String PREF_NAME = "SecurityAppPrefs";
    private static final String KEY_LOGGED_IN_USER_ID = "loggedInUserId";
    private static final String KEY_IS_LOGGED_IN = "isLoggedIn";

    private SharedPreferences sharedPreferences;
    private SharedPreferences.Editor editor;

    public SharedPreferencesHelper(Context context) {
        sharedPreferences = context.getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE);
        editor = sharedPreferences.edit();
    }

    public void setLoggedInUser(int userId) {
        editor.putInt(KEY_LOGGED_IN_USER_ID, userId);
        editor.putBoolean(KEY_IS_LOGGED_IN, true);
        editor.apply(); // Apply changes asynchronously
    }

    public int getLoggedInUserId() {
        return sharedPreferences.getInt(KEY_LOGGED_IN_USER_ID, -1); // -1 if no user logged in
    }

    public boolean isLoggedIn() {
        return sharedPreferences.getBoolean(KEY_IS_LOGGED_IN, false);
    }

    public void logoutUser() {
        editor.remove(KEY_LOGGED_IN_USER_ID);
        editor.remove(KEY_IS_LOGGED_IN);
        editor.apply();
    }
}
```

#### `util/PasswordHasher.java` (مهم: هذا مجرد Placeholder)

```java
package com.example.keywordlistenerjava.util;

// !!! هام جداً: هذا الكلاس يستخدم تجزئة بسيطة جداً (MD5) وغير آمنة لإنتاج حقيقي.
// في تطبيق حقيقي، يجب استخدام مكتبات تجزئة قوية ومصممة لتجزئة كلمات المرور
// مثل BCrypt أو PBKDF2 أو Argon2. هذه الخوارزميات تتضمن "salt" وتكرارات لزيادة الأمان.
// هذا الكلاس هو فقط لأغراض العرض التوضيحي للربط مع قاعدة البيانات.

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class PasswordHasher {

    // تجزئة كلمة المرور باستخدام MD5 (غير آمن للتطبيقات الحقيقية)
    public static String hashPassword(String password) {
        try {
            MessageDigest digest = MessageDigest.getInstance("MD5");
            digest.update(password.getBytes());
            byte[] bytes = digest.digest();
            StringBuilder sb = new StringBuilder();
            for (byte b : bytes) {
                sb.append(String.format("%02x", b));
            }
            return sb.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
            return null; // Should not happen with MD5
        }
    }

    // التحقق من كلمة المرور (باستخدام نفس خوارزمية التجزئة غير الآمنة)
    public static boolean verifyPassword(String plainPassword, String hashedPassword) {
        String newHash = hashPassword(plainPassword);
        return newHash != null && newHash.equals(hashedPassword);
    }
}
```

#### `util/LocationHelper.java`

```java
package com.example.keywordlistenerjava.util;

import android.Manifest;
import android.annotation.SuppressLint;
import android.content.Context;
import android.content.pm.PackageManager;
import android.location.Location;
import android.util.Log;

import androidx.core.content.ContextCompat;

import com.google.android.gms.location.FusedLocationProviderClient;
import com.google.android.gms.location.LocationServices;
import com.google.android.gms.location.Priority;
import com.google.android.gms.tasks.CancellationTokenSource;
import com.google.android.gms.tasks.Task;

import java.util.concurrent.ExecutorService;

public class LocationHelper {
    private static final String TAG = "LocationHelper";
    private FusedLocationProviderClient fusedLocationClient;
    private Context context;

    public LocationHelper(Context context) {
        this.context = context;
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(context);
    }

    /**
     * Gets the current location with high accuracy.
     * This method is asynchronous and returns a Task.
     * Make sure you have ACCESS_FINE_LOCATION permission granted.
     *
     * @param executorService An ExecutorService to run success/failure listeners on a background thread.
     * @return A Task that will yield a Location object or an Exception.
     */
    @SuppressLint("MissingPermission") // Suppressed because permission check is done externally
    public Task<Location> getCurrentLocation(ExecutorService executorService) {
        if (ContextCompat.checkSelfPermission(context, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            Log.e(TAG, "Location permission not granted. Cannot get location.");
            // You might want to return a Task that immediately fails
            return null; // Or throw an exception
        }

        Log.d(TAG, "Requesting current location...");
        CancellationTokenSource cancellationTokenSource = new CancellationTokenSource();
        // Request high accuracy location
        return fusedLocationClient.getCurrentLocation(
                Priority.PRIORITY_HIGH_ACCURACY,
                cancellationTokenSource.getToken()
        ).addOnCompleteListener(executorService, task -> {
            if (task.isSuccessful()) {
                Location location = task.getResult();
                if (location != null) {
                    Log.d(TAG, "Location received: " + location.getLatitude() + ", " + location.getLongitude());
                } else {
                    Log.w(TAG, "Current location task successful, but location is null.");
                }
            } else {
                Exception exception = task.getException();
                Log.e(TAG, "Failed to get current location: " + (exception != null ? exception.getMessage() : "Unknown error"), exception);
            }
        });
    }

    /**
     * Generates a Google Maps link for the given latitude and longitude.
     * @param latitude Latitude.
     * @param longitude Longitude.
     * @return A Google Maps URL.
     */
    public static String generateGoogleMapsLink(double latitude, double longitude) {
        return "http://maps.google.com/maps?q=" + latitude + "," + longitude;
    }
}
```

---

### **الجزء الخامس: دمج قاعدة البيانات في الأنشطة والخدمات**

#### `activity/RegisterActivity.java` (مفصل)

```java
package com.example.keywordlistenerjava.activity;

import android.content.Intent;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.example.keywordlistenerjava.R;
import com.example.keywordlistenerjava.db.dao.UserDao;
import com.example.keywordlistenerjava.db.entity.User;
import com.example.keywordlistenerjava.util.PasswordHasher; // للاستخدام التجريبي
import com.example.keywordlistenerjava.util.SharedPreferencesHelper;

import java.util.regex.Pattern; // للتحقق من رقم الهاتف

public class RegisterActivity extends AppCompatActivity {

    private static final String TAG = "RegisterActivity";

    private EditText etFirstName, etLastName, etPhoneNumber, etResidence, etPassword;
    private Button btnRegister;
    private UserDao userDao;
    private SharedPreferencesHelper prefsHelper;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_register);

        userDao = new UserDao(this);
        prefsHelper = new SharedPreferencesHelper(this);

        etFirstName = findViewById(R.id.et_first_name);
        etLastName = findViewById(R.id.et_last_name);
        etPhoneNumber = findViewById(R.id.et_phone_number);
        etResidence = findViewById(R.id.et_residence);
        etPassword = findViewById(R.id.et_password);
        btnRegister = findViewById(R.id.btn_register);

        btnRegister.setOnClickListener(v -> registerUser());
    }

    private void registerUser() {
        String firstName = etFirstName.getText().toString().trim();
        String lastName = etLastName.getText().toString().trim();
        String phoneNumber = etPhoneNumber.getText().toString().trim();
        String residence = etResidence.getText().toString().trim();
        String password = etPassword.getText().toString().trim();

        if (!validateInputs(firstName, lastName, phoneNumber, password)) {
            return;
        }

        User newUser = new User();
        newUser.setFirstName(firstName);
        newUser.setLastName(lastName);
        newUser.setPhoneNumber(phoneNumber);
        newUser.setResidenceArea(residence);
        newUser.setPasswordHash(password); // PasswordHasher will handle hashing inside DAO

        // Perform DB operation on a background thread
        new Thread(() -> {
            userDao.open();
            long userId = userDao.registerUser(newUser); // This handles hashing and default linking
            userDao.close();

            runOnUiThread(() -> { // Update UI on the main thread
                if (userId != -1) {
                    Toast.makeText(this, "تم تسجيل الحساب بنجاح", Toast.LENGTH_SHORT).show();
                    // Log in the newly registered user
                    prefsHelper.setLoggedInUser((int) userId);
                    // Navigate to Main Activity
                    Intent intent = new Intent(RegisterActivity.this, MainActivity.class);
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK); // Clear back stack
                    startActivity(intent);
                    finish();
                } else {
                    Toast.makeText(this, "فشل تسجيل الحساب، قد يكون رقم الهاتف مستخدمًا بالفعل", Toast.LENGTH_LONG).show();
                }
            });
        }).start();
    }

    private boolean validateInputs(String firstName, String lastName, String phoneNumber, String password) {
        if (firstName.isEmpty() || lastName.isEmpty() || phoneNumber.isEmpty() || password.isEmpty()) {
            Toast.makeText(this, "يرجى ملء جميع الحقول المطلوبة", Toast.LENGTH_SHORT).show();
            return false;
        }
        if (password.length() < 6) {
            Toast.makeText(this, "كلمة المرور يجب أن تكون 6 أحرف على الأقل", Toast.LENGTH_SHORT).show();
            return false;
        }
        // Simple phone number validation (can be improved)
        if (!Pattern.matches("^09[0-9]{8}$", phoneNumber)) { // Assuming Syrian mobile numbers start with 09 and are 10 digits
            Toast.makeText(this, "رقم الهاتف غير صحيح (مثال: 09XXXXXXXX)", Toast.LENGTH_SHORT).show();
            return false;
        }
        return true;
    }

    // Optional: If you want to handle lifecycle of DAO.
    @Override
    protected void onDestroy() {
        super.onDestroy();
        // If DAO is used by other threads or in a singleton, you might not close here.
        // For this example, it's fine as a new DAO is created per activity/thread.
        // userDao.close(); // Closed in the thread's finally block or similar.
    }
}
```

#### `activity/LoginActivity.java` (مفصل)

```java
package com.example.keywordlistenerjava.activity;

import android.content.Intent;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.TextView;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

import com.example.keywordlistenerjava.R;
import com.example.keywordlistenerjava.db.dao.UserDao;
import com.example.keywordlistenerjava.db.entity.User;
import com.example.keywordlistenerjava.util.SharedPreferencesHelper;

public class LoginActivity extends AppCompatActivity {

    private static final String TAG = "LoginActivity";

    private EditText etPhoneNumber, etPassword;
    private Button btnLogin;
    private TextView tvRegisterLink;
    private UserDao userDao;
    private SharedPreferencesHelper prefsHelper;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        userDao = new UserDao(this);
        prefsHelper = new SharedPreferencesHelper(this);

        // Check if user is already logged in
        if (prefsHelper.isLoggedIn()) {
            Log.d(TAG, "User already logged in. Navigating to MainActivity.");
            navigateToMainActivity();
            return; // Stop further execution of onCreate
        }

        etPhoneNumber = findViewById(R.id.et_login_phone_number);
        etPassword = findViewById(R.id.et_login_password);
        btnLogin = findViewById(R.id.btn_login);
        tvRegisterLink = findViewById(R.id.tv_register_link);

        btnLogin.setOnClickListener(v -> loginUser());
        tvRegisterLink.setOnClickListener(v -> navigateToRegisterActivity());
    }

    private void loginUser() {
        String phoneNumber = etPhoneNumber.getText().toString().trim();
        String password = etPassword.getText().toString().trim();

        if (phoneNumber.isEmpty() || password.isEmpty()) {
            Toast.makeText(this, "يرجى إدخال رقم الهاتف وكلمة المرور", Toast.LENGTH_SHORT).show();
            return;
        }

        new Thread(() -> {
            userDao.open();
            User user = userDao.authenticateUser(phoneNumber, password);
            userDao.close();

            runOnUiThread(() -> {
                if (user != null) {
                    Toast.makeText(this, "تم تسجيل الدخول بنجاح", Toast.LENGTH_SHORT).show();
                    prefsHelper.setLoggedInUser(user.getUserId());
                    navigateToMainActivity();
                } else {
                    Toast.makeText(this, "فشل تسجيل الدخول: رقم الهاتف أو كلمة المرور غير صحيحة", Toast.LENGTH_LONG).show();
                }
            });
        }).start();
    }

    private void navigateToRegisterActivity() {
        Intent intent = new Intent(LoginActivity.this, RegisterActivity.class);
        startActivity(intent);
    }

    private void navigateToMainActivity() {
        Intent intent = new Intent(LoginActivity.this, MainActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        startActivity(intent);
        finish(); // Finish LoginActivity so user cannot go back to it
    }
}
```

#### `activity/MainActivity.java` (مفصل ومحدث)

```java
package com.example.keywordlistenerjava.activity;

import android.Manifest;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import com.example.keywordlistenerjava.R;
import com.example.keywordlistenerjava.db.dao.AlertLogDao;
import com.example.keywordlistenerjava.db.dao.UserDao;
import com.example.keywordlistenerjava.db.entity.User;
import com.example.keywordlistenerjava.service.PorcupineService;
import com.example.keywordlistenerjava.util.SharedPreferencesHelper;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
    private static final int PERMISSIONS_REQUEST_CODE = 101;

    private TextView tvWelcome, tvTotalAlerts, tvRealAlerts, tvFalseAlerts;
    private Button btnStartPorcupine, btnStopPorcupine, btnViewLog, btnSettings, btnLogout;

    private SharedPreferencesHelper prefsHelper;
    private UserDao userDao;
    private AlertLogDao alertLogDao;
    private ExecutorService dbExecutor; // Executor for database operations

    private int currentUserId;

    // Permissions needed for PorcupineService, Location, and SMS
    private String[] requiredPermissions;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        prefsHelper = new SharedPreferencesHelper(this);
        userDao = new UserDao(this);
        alertLogDao = new AlertLogDao(this);
        dbExecutor = Executors.newSingleThreadExecutor(); // Single thread for DB ops

        // Check if user is logged in, otherwise redirect to Login
        if (!prefsHelper.isLoggedIn()) {
            Log.w(TAG, "No user logged in. Redirecting to LoginActivity.");
            Intent intent = new Intent(MainActivity.this, LoginActivity.class);
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
            startActivity(intent);
            finish();
            return;
        }

        currentUserId = prefsHelper.getLoggedInUserId();
        if (currentUserId == -1) {
            Log.e(TAG, "Logged in user ID is -1. Critical error, logging out.");
            prefsHelper.logoutUser();
            Intent intent = new Intent(MainActivity.this, LoginActivity.class);
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
            startActivity(intent);
            finish();
            return;
        }

        // Initialize UI elements
        tvWelcome = findViewById(R.id.tv_welcome);
        tvTotalAlerts = findViewById(R.id.tv_total_alerts);
        tvRealAlerts = findViewById(R.id.tv_real_alerts);
        tvFalseAlerts = findViewById(R.id.tv_false_alerts);
        btnStartPorcupine = findViewById(R.id.btn_start_porcupine);
        btnStopPorcupine = findViewById(R.id.btn_stop_porcupine);
        btnViewLog = findViewById(R.id.btn_view_log);
        btnSettings = findViewById(R.id.btn_settings);
        btnLogout = findViewById(R.id.btn_logout);

        // Set Listeners
        btnStartPorcupine.setOnClickListener(v -> checkPermissionsAndStartService());
        btnStopPorcupine.setOnClickListener(v -> stopPorcupineService());
        btnViewLog.setOnClickListener(v -> startActivity(new Intent(MainActivity.this, AlertLogActivity.class)));
        btnSettings.setOnClickListener(v -> startActivity(new Intent(MainActivity.this, SettingsActivity.class)));
        btnLogout.setOnClickListener(v -> logoutUser());

        // Build permissions list for current API level
        buildRequiredPermissionsList();

        // Load user info and update stats
        loadUserDataAndStats();
    }

    @Override
    protected void onResume() {
        super.onResume();
        // Refresh stats every time activity is resumed
        loadUserDataAndStats();
    }

    private void loadUserDataAndStats() {
        dbExecutor.execute(() -> {
            userDao.open();
            User currentUser = userDao.getUserById(currentUserId);
            userDao.close();

            alertLogDao.open();
            int total = alertLogDao.getTotalAlertsCount(currentUserId);
            int real = alertLogDao.getRealAlertsCount(currentUserId);
            int aFalse = alertLogDao.getFalseAlertsCount(currentUserId);
            alertLogDao.close();

            runOnUiThread(() -> {
                if (currentUser != null) {
                    tvWelcome.setText("مرحبًا، " + currentUser.getFirstName() + " " + currentUser.getLastName());
                } else {
                    tvWelcome.setText("مرحبًا أيها المستخدم!"); // Fallback
                }
                tvTotalAlerts.setText("عدد البلاغات الإجمالي: " + total);
                tvRealAlerts.setText("عدد البلاغات الحقيقية: " + real);
                tvFalseAlerts.setText("عدد البلاغات الكاذبة: " + aFalse);
            });
        });
    }

    private void buildRequiredPermissionsList() {
        ArrayList<String> permissionsList = new ArrayList<>();
        permissionsList.add(Manifest.permission.RECORD_AUDIO);
        permissionsList.add(Manifest.permission.ACCESS_FINE_LOCATION);
        permissionsList.add(Manifest.permission.ACCESS_COARSE_LOCATION);
        permissionsList.add(Manifest.permission.SEND_SMS);

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) { // Android 9+
            permissionsList.add(Manifest.permission.FOREGROUND_SERVICE);
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) { // Android 13+ (API 33)
            permissionsList.add(Manifest.permission.POST_NOTIFICATIONS);
        }

        requiredPermissions = permissionsList.toArray(new String[0]);
        Log.d(TAG,"Required permissions: " + Arrays.toString(requiredPermissions));
    }

    private void checkPermissionsAndStartService() {
        List<String> permissionsToRequest = new ArrayList<>();
        for (String permission : requiredPermissions) {
            if (ContextCompat.checkSelfPermission(this, permission) != PackageManager.PERMISSION_GRANTED) {
                // Foreground service types (MICROPHONE, LOCATION) are declared in manifest, not requested here
                if (!permission.equals(Manifest.permission.FOREGROUND_SERVICE_MICROPHONE) &&
                    !permission.equals(Manifest.permission.FOREGROUND_SERVICE_LOCATION)) {
                    permissionsToRequest.add(permission);
                }
            }
        }

        if (permissionsToRequest.isEmpty()) {
            startPorcupineService();
        } else {
            ActivityCompat.requestPermissions(this, permissionsToRequest.toArray(new String[0]), PERMISSIONS_REQUEST_CODE);
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == PERMISSIONS_REQUEST_CODE) {
            boolean allGranted = true;
            for (int grantResult : grantResults) {
                if (grantResult != PackageManager.PERMISSION_GRANTED) {
                    allGranted = false;
                    break;
                }
            }

            if (allGranted) {
                startPorcupineService();
            } else {
                Toast.makeText(this, "الأذونات ضرورية لتشغيل خدمة الاستماع.", Toast.LENGTH_LONG).show();
            }
        }
    }

    private void startPorcupineService() {
        Log.d(TAG, "Starting PorcupineService...");
        Intent serviceIntent = new Intent(this, PorcupineService.class);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) { // Android 8.0+
            ContextCompat.startForegroundService(this, serviceIntent);
        } else {
            startService(serviceIntent);
        }
        Toast.makeText(this, "تم بدء خدمة الاستماع (Porcupine)", Toast.LENGTH_SHORT).show();
    }

    private void stopPorcupineService() {
        Log.d(TAG, "Stopping PorcupineService...");
        Intent serviceIntent = new Intent(this, PorcupineService.class);
        stopService(serviceIntent);
        Toast.makeText(this, "تم إيقاف خدمة الاستماع (Porcupine)", Toast.LENGTH_SHORT).show();
    }

    private void logoutUser() {
        stopPorcupineService(); // Stop service before logging out
        prefsHelper.logoutUser();
        Intent intent = new Intent(MainActivity.this, LoginActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        startActivity(intent);
        finish();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        dbExecutor.shutdownNow(); // Shut down the executor when activity is destroyed
    }
}
```

#### `activity/AlertLogActivity.java` (مفصل)

```java
package com.example.keywordlistenerjava.activity;

import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.ProgressBar;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.example.keywordlistenerjava.R;
import com.example.keywordlistenerjava.adapter.AlertLogAdapter;
import com.example.keywordlistenerjava.db.dao.AlertLogDao;
import com.example.keywordlistenerjava.db.entity.AlertLog;
import com.example.keywordlistenerjava.util.SharedPreferencesHelper;

import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class AlertLogActivity extends AppCompatActivity implements AlertLogAdapter.OnItemClickListener {

    private static final String TAG = "AlertLogActivity";

    private RecyclerView recyclerViewAlerts;
    private AlertLogAdapter adapter;
    private ProgressBar progressBar;

    private AlertLogDao alertLogDao;
    private SharedPreferencesHelper prefsHelper;
    private ExecutorService dbExecutor;

    private int currentUserId;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_alert_log);

        // Get logged in user ID
        prefsHelper = new SharedPreferencesHelper(this);
        currentUserId = prefsHelper.getLoggedInUserId();
        if (currentUserId == -1) {
            Toast.makeText(this, "خطأ: المستخدم غير مسجل الدخول.", Toast.LENGTH_SHORT).show();
            finish();
            return;
        }

        alertLogDao = new AlertLogDao(this);
        dbExecutor = Executors.newSingleThreadExecutor();

        recyclerViewAlerts = findViewById(R.id.recycler_view_alerts);
        progressBar = findViewById(R.id.progress_bar_alert_log);

        recyclerViewAlerts.setLayoutManager(new LinearLayoutManager(this));

        loadAlertLogs();
    }

    private void loadAlertLogs() {
        progressBar.setVisibility(View.VISIBLE);
        dbExecutor.execute(() -> {
            alertLogDao.open();
            List<AlertLog> alerts = alertLogDao.getAllAlertsForUser(currentUserId);
            alertLogDao.close();

            runOnUiThread(() -> {
                progressBar.setVisibility(View.GONE);
                if (alerts.isEmpty()) {
                    Toast.makeText(this, "لا توجد بلاغات مسجلة بعد.", Toast.LENGTH_SHORT).show();
                }
                adapter = new AlertLogAdapter(alerts, this); // Pass 'this' for listener
                recyclerViewAlerts.setAdapter(adapter);
            });
        });
    }

    @Override
    public void onMapLinkClick(String mapLink) {
        if (mapLink != null && !mapLink.isEmpty()) {
            try {
                Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(mapLink));
                startActivity(intent);
            } catch (Exception e) {
                Log.e(TAG, "Error opening map link: " + mapLink, e);
                Toast.makeText(this, "لا يمكن فتح رابط الخريطة.", Toast.LENGTH_SHORT).show();
            }
        } else {
            Toast.makeText(this, "لا يوجد رابط خريطة لهذا البلاغ.", Toast.LENGTH_SHORT).show();
        }
    }

    // Optional: Add logic to update false alarm status (e.g., via a long press or context menu)
    // For a real admin panel, this would be on the server side.
    public void updateAlertStatus(int logId, Boolean isFalseAlarm) {
        dbExecutor.execute(() -> {
            alertLogDao.open();
            int rowsAffected = alertLogDao.updateFalseAlarmStatus(logId, isFalseAlarm);
            alertLogDao.close();

            runOnUiThread(() -> {
                if (rowsAffected > 0) {
                    Toast.makeText(this, "تم تحديث حالة البلاغ.", Toast.LENGTH_SHORT).show();
                    loadAlertLogs(); // Refresh the list
                } else {
                    Toast.makeText(this, "فشل تحديث حالة البلاغ.", Toast.LENGTH_SHORT).show();
                }
            });
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        dbExecutor.shutdownNow();
    }
}
```

#### `adapter/AlertLogAdapter.java` (مفصل)

```java
package com.example.keywordlistenerjava.adapter;

import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;

import com.example.keywordlistenerjava.R;
import com.example.keywordlistenerjava.db.entity.AlertLog;

import java.util.List;

public class AlertLogAdapter extends RecyclerView.Adapter<AlertLogAdapter.AlertLogViewHolder> {

    private List<AlertLog> alertLogs;
    private OnItemClickListener listener;

    // Interface for click events (e.g., opening map link)
    public interface OnItemClickListener {
        void onMapLinkClick(String mapLink);
        // void onMarkFalseAlarmClick(int logId); // Add if you implement this
    }

    public AlertLogAdapter(List<AlertLog> alertLogs, OnItemClickListener listener) {
        this.alertLogs = alertLogs;
        this.listener = listener;
    }

    @NonNull
    @Override
    public AlertLogViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item_alert, parent, false);
        return new AlertLogViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull AlertLogViewHolder holder, int position) {
        AlertLog alert = alertLogs.get(position);

        holder.tvLogId.setText("رقم الحالة: " + alert.getLogId());
        holder.tvKeywordUsed.setText("الكلمة المفتاحية: " + alert.getKeywordUsed());
        holder.tvDateTime.setText("التاريخ والوقت: " + alert.getAlertDate() + " " + alert.getAlertTime());
        holder.tvLocation.setText("الموقع: " + String.format("%.6f, %.6f", alert.getLatitude(), alert.getLongitude()));

        // Display status
        String statusText;
        if (alert.getIsFalseAlarm() == null) {
            statusText = "الحالة: قيد التقييم";
        } else if (alert.getIsFalseAlarm()) {
            statusText = "الحالة: بلاغ كاذب";
            holder.tvStatus.setTextColor(holder.itemView.getContext().getResources().getColor(android.R.color.holo_red_dark));
        } else {
            statusText = "الحالة: بلاغ حقيقي";
            holder.tvStatus.setTextColor(holder.itemView.getContext().getResources().getColor(android.R.color.holo_green_dark));
        }
        holder.tvStatus.setText(statusText);

        // Set click listener for map link (or the whole item if that's easier)
        holder.tvLocation.setOnClickListener(v -> {
            if (listener != null) {
                listener.onMapLinkClick(alert.getMapLink());
            }
        });
    }

    @Override
    public int getItemCount() {
        return alertLogs.size();
    }

    static class AlertLogViewHolder extends RecyclerView.ViewHolder {
        TextView tvLogId, tvKeywordUsed, tvDateTime, tvLocation, tvStatus;

        public AlertLogViewHolder(@NonNull View itemView) {
            super(itemView);
            tvLogId = itemView.findViewById(R.id.tv_alert_log_id);
            tvKeywordUsed = itemView.findViewById(R.id.tv_alert_keyword_used);
            tvDateTime = itemView.findViewById(R.id.tv_alert_date_time);
            tvLocation = itemView.findViewById(R.id.tv_alert_location);
            tvStatus = itemView.findViewById(R.id.tv_alert_status);
        }
    }
}
```

#### `res/layout/list_item_alert.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">

        <TextView
            android:id="@+id/tv_alert_log_id"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="رقم الحالة: 123"
            android:textSize="16sp"
            android:textStyle="bold"
            android:textColor="@android:color/black"/>

        <TextView
            android:id="@+id/tv_alert_keyword_used"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="4dp"
            android:text="الكلمة المفتاحية: طوارئ"
            android:textSize="14sp"/>

        <TextView
            android:id="@+id/tv_alert_date_time"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="4dp"
            android:text="التاريخ والوقت: 2025-05-06 10:30:00"
            android:textSize="14sp"/>

        <TextView
            android:id="@+id/tv_alert_location"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="4dp"
            android:text="الموقع: 33.510414, 36.278336 (اضغط لفتح الخريطة)"
            android:textSize="14sp"
            android:textColor="@color/design_default_color_primary"
            android:textStyle="italic"
            android:clickable="true"
            android:focusable="true"/>

        <TextView
            android:id="@+id/tv_alert_status"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="8dp"
            android:text="الحالة: قيد التقييم"
            android:textSize="16sp"
            android:textStyle="bold"/>

    </LinearLayout>

</androidx.cardview.widget.CardView>
```

#### `service/PorcupineService.java` (مقتطفات مهمة بعد التغييرات)

**تأكد من تحديث `fetchLocationAndPerformAction()` و `onKeywordDetected()` لاستخدام الـ DAOs الجديدة.**

```java
// ... (باقي الاستيرادات)
import com.example.keywordlistenerjava.db.dao.AlertLogDao;
import com.example.keywordlistenerjava.db.dao.KeywordNumberLinkDao; // لاسترجاع الأرقام المرتبطة
import com.example.keywordlistenerjava.db.dao.UserDao; // لجلب بيانات المستخدم
import com.example.keywordlistenerjava.db.entity.AlertLog;
import com.example.keywordlistenerjava.db.entity.User;
import com.example.keywordlistenerjava.db.entity.EmergencyNumber; // لجلب أرقام الطوارئ
import com.example.keywordlistenerjava.util.LocationHelper;
import com.example.keywordlistenerjava.util.SharedPreferencesHelper;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors; // للحصول على معرف المستخدم من SharedPreferences

public class PorcupineService extends Service {
    // ... (الثوابت، المتغيرات الأخرى) ...

    private AlertLogDao alertLogDao;
    private KeywordNumberLinkDao keywordNumberLinkDao;
    private UserDao userDao;
    private SharedPreferencesHelper prefsHelper; // لجلب معرف المستخدم الحالي
    private LocationHelper locationHelper; // للمساعدة في الحصول على الموقع

    // ... (onCreate) ...
    @Override
    public void onCreate() {
        super.onCreate();
        // ... (التهيئة الأصلية) ...
        alertLogDao = new AlertLogDao(this);
        keywordNumberLinkDao = new KeywordNumberLinkDao(this);
        userDao = new UserDao(this);
        prefsHelper = new SharedPreferencesHelper(this);
        locationHelper = new LocationHelper(this);
        // ... (باقي التهيئة) ...
    }

    // ... (باقي دوال دورة الحياة، initPorcupine، startPorcupineListening، stopPorcupineListening) ...

    // --- 4. الإجراء عند اكتشاف الكلمة المفتاحية ---
    private void onKeywordDetected() {
        Log.i(TAG, "onKeywordDetected: Porcupine action triggered!");

        // 1. عرض Toast
        Toast.makeText(getApplicationContext(), "Porcupine اكتشف الكلمة!", Toast.LENGTH_SHORT).show();
        updateNotification("Porcupine رصد الكلمة! جارٍ الحصول على الموقع...");

        // 2. الحصول على معرف المستخدم الحالي (من SharedPreferences)
        int currentUserId = prefsHelper.getLoggedInUserId();
        if (currentUserId == -1) {
            Log.e(TAG, "onKeywordDetected: No user logged in, cannot process alert.");
            updateNotification("خطأ: لا يوجد مستخدم مسجل الدخول.");
            stopSelf(); // قد ترغب في إيقاف الخدمة هنا
            return;
        }

        // 3. جلب بيانات المستخدم (للاسم والكنية)
        // 4. جلب الأرقام المرتبطة بالكلمة المفتاحية للمستخدم
        // 5. الحصول على الموقع وإرسال الرسالة
        // يجب أن يتم هذا في thread خلفية
        executorService.execute(() -> {
            User user = null;
            List<EmergencyNumber> recipientNumbers = new ArrayList<>();

            userDao.open();
            user = userDao.getUserById(currentUserId);
            userDao.close();

            if (user == null) {
                Log.e(TAG, "onKeywordDetected: Current user not found in DB! ID: " + currentUserId);
                mainThreadHandler.post(() -> updateNotification("خطأ: بيانات المستخدم غير موجودة."));
                return;
            }

            keywordNumberLinkDao.open();
            // الحصول على الأرقام المرتبطة بالكلمة المفتاحية المكتشفة
            // (سنفترض هنا أن الكلمة المكتشفة هي TARGET_KEYWORD، لكن في تطبيق حقيقي، ستكون لديك الكلمة الفعلية)
            recipientNumbers = keywordNumberLinkDao.getEmergencyNumbersForKeyword(currentUserId, TARGET_KEYWORD);
            keywordNumberLinkDao.close();

            if (recipientNumbers.isEmpty()) {
                Log.w(TAG, "onKeywordDetected: No emergency numbers linked for user " + currentUserId + " with keyword " + TARGET_KEYWORD);
                mainThreadHandler.post(() -> updateNotification("لا توجد أرقام طوارئ مرتبطة بالكلمة المفتاحية."));
                return;
            }
            
            // --- الحصول على الموقع وإرسال الرسالة ---
            locationHelper.getCurrentLocation(executorService) // Run location task on background executor
                .addOnSuccessListener(location -> {
                    if (location != null) {
                        double lat = location.getLatitude();
                        double lon = location.getLongitude();
                        String mapLink = LocationHelper.generateGoogleMapsLink(lat, lon);

                        // بناء الرسالة المطلوبة
                        String messageText = "لقد استعمل " + user.getFirstName() + " " + user.getLastName() +
                                " الكلمة المفتاحية (" + TARGET_KEYWORD + ") في الموقع الجغرافي على الخريطة (" + mapLink + ")";
                        
                        // إرسال الرسالة لجميع المستلمين
                        for (EmergencyNumber number : recipientNumbers) {
                            sendActionMessage(number.getPhoneNumber(), messageText);
                        }

                        // تسجيل البلاغ في قاعدة البيانات
                        AlertLog newAlert = new AlertLog();
                        newAlert.setUserId(currentUserId);
                        newAlert.setKeywordUsed(TARGET_KEYWORD);
                        newAlert.setLatitude(lat);
                        newAlert.setLongitude(lon);
                        newAlert.setMapLink(mapLink);
                        // isFalseAlarm will be null initially

                        alertLogDao.open();
                        long logId = alertLogDao.addAlertLog(newAlert);
                        alertLogDao.close();

                        if (logId != -1) {
                            Log.i(TAG, "Alert successfully logged with ID: " + logId);
                            mainThreadHandler.post(() -> updateNotification("تم إرسال التنبيه وتسجيل البلاغ."));
                        } else {
                            Log.e(TAG, "Failed to log alert to database.");
                            mainThreadHandler.post(() -> updateNotification("تم إرسال التنبيه، لكن فشل تسجيل البلاغ."));
                        }

                    } else {
                        Log.w(TAG, "onKeywordDetected: Location is null.");
                        mainThreadHandler.post(() -> updateNotification("فشل الحصول على الموقع. لم يتم إرسال التنبيه."));
                        sendActionMessageToUser(user.getPhoneNumber(), "فشل الحصول على الموقع. لم يتم إرسال التنبيه بالكامل."); // Consider notifying user
                    }
                })
                .addOnFailureListener(e -> {
                    Log.e(TAG, "onKeywordDetected: Failed to get location: " + e.getMessage(), e);
                    mainThreadHandler.post(() -> updateNotification("خطأ في تحديد الموقع. لم يتم إرسال التنبيه."));
                    sendActionMessageToUser(user.getPhoneNumber(), "خطأ في تحديد الموقع. لم يتم إرسال التنبيه بالكامل."); // Consider notifying user
                });
        });
    }

    // --- 5. إرسال رسالة SMS (محدث) ---
    private void sendActionMessage(String recipientPhoneNumber, String messageContent) {
        Log.d(TAG, "sendActionMessage: Preparing SMS to " + recipientPhoneNumber);
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS) != PackageManager.PERMISSION_GRANTED) {
            Log.e(TAG, "sendActionMessage: SEND_SMS permission missing!");
            mainThreadHandler.post(() -> updateNotification("فشل إرسال SMS (الإذن مفقود)."));
            return;
        }

        try {
            SmsManager smsManager
